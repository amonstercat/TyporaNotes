# 手写RPC框架

## 设计思路

一般情况下，RPC 框架不仅要提供服务发现功能，还要提供负载均衡、容错等功能，这样的 RPC 框架才算真正合格的。简单说一下设计一个最基本的 RPC 框架的思路：

![img](https://github.com/Snailclimb/guide-rpc-framework/raw/master/images/rpc-architure-detail.png)

1. **注册中心** ：注册中心首先是要有的，推荐使用 `Zookeeper`，注册中心负责服务地址的注册与查找，相当于目录服务。服务端启动的时候将服务名称及其对应的地址(ip+port)注册到注册中心，服务消费端根据服务名称找到对应的服务地址，有了服务地址之后，服务消费端就可以通过网络请求服务端了
2. **网络传输** ：既然要调用远程的方法就要发请求，请求中至少**要包含你调用的类名、方法名以及相关参数**吧！推荐基于 NIO 的 `Netty` 框架
3. **序列化** ：既然涉及到网络传输就一定涉及到序列化，！JDK 自带的序列化效率低并且有安全漏洞。 所以还要考虑使用哪种序列化协议，比较常用的有 `hession2`、`kyro`、`protostuff`
4. **动态代理** ：动态代理也是需要的。因为 RPC 的主要目的就是让我们调用远程方法像调用本地方法一样简单，使用动态代理可以屏蔽远程方法调用的细节比如网络传输。也就是说当你调用远程方法的时候，**实际会通过代理对象来传输网络请求**，不然的话，怎么可能直接就调用到远程方法呢？
5. **负载均衡** ：负载均衡也是需要的，举个例子我们的系统中的某个服务的访问量特别大，我们将这个服务部署在了多台服务器上，当客户端发起请求的时候，多台服务器都可以处理这个请求。那么如何正确选择处理该请求的服务器就很关键。负载均衡就是为了避免单个服务器响应同一请求，容易造成服务器宕机、崩溃等问题。





该自定义`RPC`框架主要有以下几部分内容：

## RPC-API

(1) rpc-api文件夹：服务端与客户端的公共调用接口，即测试用的`api`接口与实体

```java
public interface ByeService {

    String bye(String name);

}
public interface HelloService {

    String hello(HelloObject object);

}
//Serializable是一个标记性接口，给JVM看的
public class HelloObject implements Serializable {

    private Integer id;
    private String message;

}
```





## RPC-COMMON

`common`包下主要包含以下内容：

- `entity`:项目中的一些通用的枚举类和工具类:包括了`RpcRequest`和`RpcResponse`，其中**`RpcRequest`包含了RPC请求的 请求号(唯一性ID)、待调用接口名称、待调用方法名称、调用方法的参数、是否为心跳包，是否为重发包；`RpcResponse`则封装了响应对应的请求号、响应状态码、响应数据对象`(T Data)`**
- `enumeration`:定义了一些枚举类，如 `SerializerCode`、`ResponseCode`、`RpcError`错误信息等

```java
public enum SerializerCode {
    KRYO(0),
    JSON(1),
    HESSIAN(2),
    PROTOBUF(3);
    private final int code;
}

public enum PackageType {
    REQUEST_PACK(0),
    RESPONSE_PACK(1);
    private final int code;
}

public enum ResponseCode {
    SUCCESS(200, "调用方法成功"),
    FAIL(500, "调用方法失败"),
    METHOD_NOT_FOUND(500, "未找到指定方法"),
    CLASS_NOT_FOUND(500, "未找到指定类");
    private final int code;
    private final String message;
}
```

- `exception`:用于抛出rpc调用时出现的各种异常
- **`factory`:**主要有两部分，单例工厂以及线程池

```java
//单例工厂
public class SingletonFactory {

    // 用于存储单例对象的映射表：<类，类实例>
    private static Map<Class, Object> objectMap = new HashMap<>();

    private SingletonFactory() {
    // 私有构造函数，禁止其他程序实例化、创建该类对象
    }

    // 获取指定类的单例对象，对外提供该对象的访问方式
    public static <T> T getInstance(Class<T> clazz) {
        Object instance = objectMap.get(clazz);  //从映射表中获取对象实例
        synchronized (clazz) {  //使用类对象作为锁，保证线程安全
            if (instance == null) {  //如果实例为空，表示还未创建对象
                try {
              instance = clazz.newInstance(); //反射——使用无参构造函数创建对象实例
              objectMap.put(clazz, instance); //将对象实例放入映射表中缓存
                } catch (IllegalAccessException | InstantiationException e) {
                    // 发生异常时，抛出运行时异常
                    throw new RuntimeException(e.getMessage(), e);
                }
            }
        }
        return clazz.cast(instance); // 将对象实例转型为指定类型并返回
    }

}



//线程池


//Map存储 <线程名前缀，线程池对象>
private static Map<String, ExecutorService> threadPollsMap = new ConcurrentHashMap<>();


/**
 * 创建默认线程池
 * @param threadNamePrefix 线程名前缀
 * @param daemon 指定是否为守护线程
 * @return ExecutorService对象
 */
public static ExecutorService createDefaultThreadPool(String threadNamePrefix, Boolean daemon) {
    ExecutorService pool = threadPollsMap.computeIfAbsent(threadNamePrefix, k -> createThreadPool(threadNamePrefix, daemon)); // 从映射表中获取线程池，如果不存在则创建新的线程池
    if (pool.isShutdown() || pool.isTerminated()) {
        threadPollsMap.remove(threadNamePrefix); // 如果线程池已关闭或终止，则从映射表中移除
        pool = createThreadPool(threadNamePrefix, daemon); // 创建新的线程池
        threadPollsMap.put(threadNamePrefix, pool); // 将新的线程池放入映射表中
    }
    return pool; // 返回线程池对象
}


//调用ThreadPoolExecutor构造方法来创建线程池
    private static ExecutorService createThreadPool(String threadNamePrefix, Boolean daemon) {
        BlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(BLOCKING_QUEUE_CAPACITY);
        ThreadFactory threadFactory = createThreadFactory(threadNamePrefix, daemon);
        return new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE_SIZE, KEEP_ALIVE_TIME, TimeUnit.MINUTES, workQueue, threadFactory);
    }

//关闭线程池
public static void shutDownAll() {
    //略
}
```



- `util`:包含了`Nacos`工具类(管理`Nacos`连接、注册/注销服务实例)，以及反射的工具类

```java
//NamingService是nacos提供用来实现服务注册、服务订阅、服务发现等功能的api，由NacosNamingService唯一实现，通过这个api就可以跟nacos服务端实现通信
private static final NamingService namingService;


//通过NamingFactory.createNamingService()来创建Nacos中的NamingService
 public static NamingService getNacosNamingService() {
        try {
            return NamingFactory.createNamingService(SERVER_ADDR);
        } catch (NacosException e) {
            logger.error("连接到Nacos时有错误发生: ", e);
            throw new RpcException(RpcError.FAILED_TO_CONNECT_TO_SERVICE_REGISTRY);
        }
    }


//服务注册
public static void registerService(String serviceName, InetSocketAddress address) throws NacosException {
        namingService.registerInstance(serviceName, address.getHostName(), address.getPort());
        NacosUtil.address = address;
        serviceNames.add(serviceName);

}


//获取nacos中的所有实例
public static List<Instance> getAllInstance(String serviceName) throws NacosException {
        return namingService.getAllInstances(serviceName);
    }

//清空nacos中的所有服务
public static void clearRegistry() {
        if(!serviceNames.isEmpty() && address != null) {
            String host = address.getHostName();
            int port = address.getPort();
            Iterator<String> iterator = serviceNames.iterator();
            while(iterator.hasNext()) {
                String serviceName = iterator.next();
                try {
                    namingService.deregisterInstance(serviceName, host, port);
                } catch (NacosException e) {
                    logger.error("注销服务 {} 失败", serviceName, e);
                }
            }
        }
    }
```



## RPC-CORE 核心:computer:



### 自定义注解 Annotation

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Service {
    /*自定义一个@RpcService注解，声明一个name属性，统一修饰提供出去的服务
    在服务提供者启动时，统一将带有@service注解的类通过反射方式创建对象
    以@rpcservice注解提供的name属性当key加入到Map容器中
    * */
    public String name() default "";
}


/**
 * 服务扫描的基包
 * @author ziyang
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ServiceScan {

    public String value() default "";

}

```

上述代码定义了标记该类为`Service`服务类的注解`@Service`以及服务扫描的注解`@ServiceScan`,随后在RPC-Server启动时开启服务扫描，具体如何扫描参考以下代码：

```java
 public SocketServer(String host, int port, Integer serializer) {
        this.host = host;
        this.port = port;
        //创建一个线程池（threadPool），用于处理客户端请求
        threadPool = ThreadPoolFactory.createDefaultThreadPool("socket-rpc-server");
        this.serviceRegistry = new NacosServiceRegistry();//用于服务注册
        this.serviceProvider = new ServiceProviderImpl();//用于服务的提供和管理
        this.serializer = CommonSerializer.getByCode(serializer);
        scanServices();//用于扫描和注册可用的服务
    }


 /**
     * 启动服务后，扫描所有service类，并自动发布到注册中心
     * @throws RpcException
     */
    public void scanServices() {
// 获取调用者 start 服务时所在的主类名, 即 调用者 调用 AbstractRpcServer 的子类 类名
        String mainClassName = ReflectUtil.getStackTrace();
        Class<?> startClass;
        try {
            startClass = Class.forName(mainClassName);
            if(!startClass.isAnnotationPresent(ServiceScan.class)) {
                logger.error("启动类缺少 @ServiceScan 注解");
                throw new RpcException(RpcError.SERVICE_SCAN_PACKAGE_NOT_FOUND);
            }
        } catch (ClassNotFoundException e) {
            logger.error("出现未知错误");
            throw new RpcException(RpcError.UNKNOWN_ERROR);
        }
        String basePackage = startClass.getAnnotation(ServiceScan.class).value();
        if("".equals(basePackage)) {
            basePackage = mainClassName.substring(0, mainClassName.lastIndexOf("."));
        }
        Set<Class<?>> classSet = ReflectUtil.getClasses(basePackage);
        for(Class<?> clazz : classSet) {
            if(clazz.isAnnotationPresent(Service.class)) {
      //在某个类上标注了 @Service 注解，并设置了 name 属性值，
      //可以使用 class.getAnnotation(Service.class).name() 来获取该属性的值
      //该name的值就是Nacos注册中心里的服务名
      String serviceName = clazz.getAnnotation(Service.class).name();
                Object obj;
                try {
                    obj = clazz.newInstance();
                } catch (InstantiationException | IllegalAccessException e) {
                    logger.error("创建 " + clazz + " 时有错误发生");
                    continue;
                }
                if("".equals(serviceName)) {
                    Class<?>[] interfaces = clazz.getInterfaces();
                    for (Class<?> oneInterface: interfaces){
              publishService(obj, oneInterface.getCanonicalName());
                    }
                } else {
             publishService(obj, serviceName); //把扫描到的服务注册到Nacos
                }
            }
        }
    }
```

上述代码主要是一个服务扫描的方法，用于在启动过程中扫描指定包下的类，并将带有@Service注解的类注册为服务。具体步骤如下：

1. 获取当前方法的调用堆栈信息，找到启动类的名称
2. 通过反射加载**启动类**，并检查是否存在`@ServiceScan`注解？如果不存在该注解，将抛出`RpcException`异常
3. 获取`@ServiceScan`注解中指定的基础包名，如果没有指定，则默认为启动类所在包的上级包(**扫描哪个目录下的服务**)
4. 使用反射工具类`ReflectUtil`，根据基础包名获取该包下的所有类的Class对象的集合
5. 遍历类集合，对于每个类，判断是否标注了`@Service`注解？
6. **如果标注了@Service注解，根据注解中定义的`name`属性获取服务名称**，如果没有指定服务名称，则将该类实现的所有接口作为服务名称
7. 创建该类的实例对象
8. 如果服务名称为空(**被扫描的Service服务类上没有指定Name属性值**)，则**遍历该类实现的接口，并将实例对象发布为对应接口的服务**
9. 如果服务名称不为空，则直接将实例对象发布为指定名称(Name值)的服务，例如下面的`HelloService`

```java
@Service(name = "HelloService")
public class HelloServiceImpl implements HelloService {
//具体实现
}
```

10. 最后，将扫描到的服务注册到Nacos或其他服务注册中心





### 编码/解码拦截器 (自定义消息头)

编码/解码的过程对于`Socket`和`Netty`来说是差不多的

| 字段            | 解释                                                         |
| --------------- | ------------------------------------------------------------ |
| Magic Number    | 魔数，表识一个 MRF 协议包，0xCAFEBABE                        |
| Package Type    | 包类型，标明这是一个调用请求还是调用响应                     |
| Serializer Type | 序列化器类型，标明这个包的数据的序列化方式                   |
| Data Length     | 数据字节的长度                                               |
| Data Bytes      | 传输的对象，通常是一个`RpcRequest`或`RpcClient`对象，取决于`Package Type`字段，对象的序列化方式取决于`Serializer Type`字段。 |





`Encode`部分：

将对象编码为字节流，并写入到 `ByteBuf` 中，以便进行网络传输

```java
public class CommonEncoder extends MessageToByteEncoder {

//定义一个常量 MAGIC_NUMBER，用于标识消息的起始。在编码过程中，会将该标识写入到字节流中
// 对象头的魔术: cafe babe 表示 class 类型的文件
    //private static final int MAGIC_NUMBER = 0xCAFEBABE;
    private static final short MAGIC_NUMBER = (short) 0xBABE;

    /**
     * 自定义对象头 协议 16 字节
     * 4 字节 魔数
     * 4 字节 协议包类型
     * 4 字节 序列化类型
     * 4 字节 数据长度
     * @param ctx
     * @param msg
     * @param out
     * @throws Exception
     *
     *  4 bytes代表int类型——因此可传输的有效数据长度最大为2^32bit——>4字节
     *
     *       The transmission protocol is as follows :
     * +---------------+---------------+-----------------+-------------+
     * | Magic Number  | Package Type  | Serializer Type | Data Length |
     * | 4 bytes       | 4 bytes       | 4 bytes         | 4 bytes     |
     * +---------------+---------------+-----------------+-------------+
     * |                           Data Bytes                          |
     * |                       Length: ${Data Length}                  |
     * +---------------+---------------+-----------------+-------------+
     *
     *                            ↓  改良  ↓
     *
     * +---------------+---------------+-----------------+-------------+
     * | Magic Number  | Package Type  | Serializer Type | Data Length |
     * | 2 bytes       | 1 bytes       | 1 bytes         | 4 bytes     |
     * +---------------+---------------+-----------------+-------------+
     * |                           Data Bytes                          |
     * |                       Length: ${Data Length}                  |
     * +---------------+---------------+-----------------+-------------+
     */
    
 @Override
protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
        out.writeShort(MAGIC_NUMBER); // 写进的2个字节
        if (msg instanceof RpcRequest) {
            out.writeByte(PackageType.REQUEST_PACK.getCode()); // 写进的1个字节
        } else {
            out.writeByte(PackageType.RESPONSE_PACK.getCode()); // 写进的1个字节
        }
        out.writeByte(serializer.getCode()); // 写进的1个字节
        byte[] bytes = serializer.serialize(msg);
        int length = bytes.length;
        out.writeInt(bytes.length); // 写进的4个字节
        log.debug("encode object length [{}] bytes", length);
        out.writeBytes(bytes); // 写进的对象内容，二进制形式
    }
}

//然后观察rpc-client如何发送request的

// 将请求对象序列化后发送到服务提供者
ObjectWriter.writeObject(outputStream, rpcRequest, serializer);

// 从服务提供者接收并反序列化响应对象
Object obj = ObjectReader.readObject(inputStream);



public static void writeObject(OutputStream outputStream, Object object, CommonSerializer serializer) throws IOException {

    outputStream.write(intToBytes(MAGIC_NUMBER));
    if (object instanceof RpcRequest) {
        outputStream.write(intToBytes(PackageType.REQUEST_PACK.getCode()));
    } else {
        outputStream.write(intToBytes(PackageType.RESPONSE_PACK.getCode()));
    }
    outputStream.write(intToBytes(serializer.getCode()));
    byte[] bytes = serializer.serialize(object);
    outputStream.write(intToBytes(bytes.length));
    outputStream.write(bytes);
    outputStream.flush();
}
```



`Decode`部分：

```java
public class CommonDecoder extends ReplayingDecoder {   

private static final int MAGIC_NUMBER = 0xCAFEBABE;

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        int magic = in.readInt();//从 ByteBuf 中读取 magic 值，用于验证协议包的标识
        if (magic != MAGIC_NUMBER) {
         //如果 magic 值不等于预设的 MAGIC_NUMBER，表示收到的协议包不被识别，将抛出 RpcException 异常
            logger.error("不识别的协议包: {}", magic);
            throw new RpcException(RpcError.UNKNOWN_PROTOCOL);
        }
        int packageCode = in.readInt(); //读取 packageCode 值，用于确定协议包的类型
        Class<?> packageClass;
        if (packageCode == PackageType.REQUEST_PACK.getCode()) {
            packageClass = RpcRequest.class;
        } else if (packageCode == PackageType.RESPONSE_PACK.getCode()) {
            packageClass = RpcResponse.class;
        } else {
            logger.error("不识别的数据包: {}", packageCode);
            throw new RpcException(RpcError.UNKNOWN_PACKAGE_TYPE);
        }
        int serializerCode = in.readInt();//读取 serializerCode 值，用于确定反序列化器的类型
        CommonSerializer serializer = CommonSerializer.getByCode(serializerCode);
        if (serializer == null) {
            logger.error("不识别的反序列化器: {}", serializerCode);
            throw new RpcException(RpcError.UNKNOWN_SERIALIZER);
        }
        int length = in.readInt();//读取 length 值，用于确定待解码字节数组的长度
        byte[] bytes = new byte[length];//读取 length 长度的字节，将其存储到字节数组 bytes 中 解决TCP粘包
        in.readBytes(bytes);// Netty 中的方法，用于从 ByteBuf 中读取指定长度的字节，并将其存储到字节数组中
        Object obj = serializer.deserialize(bytes, packageClass);//将字节数组 bytes 反序列化为指定的 packageClass 对象
        out.add(obj);//将反序列化后的对象 obj 添加到输出列表 out 中,供后续处理
    }
       
}
```

以上代码的作用是解码收到的字节流，根据协议包的标识、类型、反序列化器等信息，将字节流反序列化为相应的对象，并将其添加到输出列表中供后续处理。这是一个与网络传输相关的解码器，用于在 RPC 框架中处理接收到的网络数据





### 序列化 Serializer

接口`CommonSerializer`定义了一些序列化/反序列化过程中需要实现的公共方法：

```java

public interface CommonSerializer {
    Integer KRYO_SERIALIZER = 0;
    Integer JSON_SERIALIZER = 1;
    Integer HESSIAN_SERIALIZER = 2;
    Integer PROTOBUF_SERIALIZER = 3;
    Integer DEFAULT_SERIALIZER = KRYO_SERIALIZER;// 默认序列化器的编码为Kryo序列化器	
    
    // 根据code获取序列化器实例
    static CommonSerializer getByCode(int code) {
        switch (code) {
            case 0:
                return new KryoSerializer();
            case 1:
                return new JsonSerializer();
            case 2:
                return new HessianSerializer();
            case 3:
                return new ProtobufSerializer();// 返回Protobuf序列化器实例
            default:
                return null;
        }
    }
    byte[] serialize(Object obj);
    Object deserialize(byte[] bytes, Class<?> clazz);
    int getCode();
}
```

下面主要介绍两种序列化机制的实现：



#### `Kryo`

```java
//使用ThreadLocal
private static final ThreadLocal<Kryo> kryoThreadLocal = ThreadLocal.withInitial(() -> {
        Kryo kryo = new Kryo();
        kryo.register(RpcResponse.class);//赋予该 Class 一个从 0 开始的编号
        kryo.register(RpcRequest.class);
        kryo.setReferences(true);//允许引用对象的序列化和反序列化
        kryo.setRegistrationRequired(false);
        return kryo;
    });

```

上述代码创建了一个 `ThreadLocal` 对象 `kryoThreadLocal`，它的泛型类型是 `Kryo`。`ThreadLocal` 提供了线程本地变量的功能，它为每个线程提供了一个独立的变量副本，因此不同的线程访问该变量时不会相互干扰。通过调用 `ThreadLocal` 的 `withInitial()` 方法，传入一个 `Lambda` 表达式，初始化方法会在每个线程第一次访问该变量时执行，并返回初始化的值。

在这段代码中，`Lambda` 表达式中的初始化方法会执行以下操作：

- 创建一个 `Kryo` 对象 `kryo`
- 使用 `kryo.register()` 方法注册 `RpcResponse.class` 和 `RpcRequest.class` 类型，以便在序列化和反序列化时可以正确处理这些类型
- 设置 `kryo` 对象的引用模式为 `true`，允许引用对象的序列化和反序列化
- 设置 `kryo` 对象的注册非必需模式为 `false`，允许序列化和反序列化未注册的类
- 返回初始化后的 `kryo` 对象

因此，`kryoThreadLocal` 是一个 `ThreadLocal` 对象，用于存储每个线程独立的 `Kryo` 对象，保证了线程之间的隔离性和线程安全性

```java

    @Override
    public byte[] serialize(Object obj) {
        try (ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
             Output output = new Output(byteArrayOutputStream)) {
            Kryo kryo = kryoThreadLocal.get();
            kryo.writeObject(output, obj);
            kryoThreadLocal.remove();
            return output.toBytes();
        } catch (Exception e) {
            logger.error("序列化时有错误发生:", e);
            throw new SerializeException("序列化时有错误发生");
        }
    }

/*
 Kryo 的 Input 和 Output 接收一个 InputStream 和 OutputStream，Kryo 通常完成字节数组和对象的转换，所以常用的输入输出流实现为 ByteArrayInputStream/ByteArrayOutputStream
*/
    
    @Override
    public Object deserialize(byte[] bytes, Class<?> clazz) {
        try (ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
             Input input = new Input(byteArrayInputStream)) {
            Kryo kryo = kryoThreadLocal.get();
            Object o = kryo.readObject(input, clazz);
            kryoThreadLocal.remove();
            return o;
        } catch (Exception e) {
            logger.error("反序列化时有错误发生:", e);
            throw new SerializeException("反序列化时有错误发生");
        }
    }
```



参考： [深入理解 RPC 之序列化篇 --Kryo](https://lexburner.github.io/rpc-serialize-1/)



#### `Protostuff`   :grey_exclamation:



> 重点关注Schema的作用：在序列化操作中，库使用 Schema 来确定如何处理每个字段，编码类型和顺序，并在反序列化操作中使用相同的 Schema 进行解析和恢复。

```java
public class ProtobufSerializer implements CommonSerializer {

// 申请一个内存空间用户缓存
private LinkedBuffer buffer = LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE); 
    
// 缓存消息类型和对应的Schema
private Map<Class<?>, Schema<?>> schemaCache = new ConcurrentHashMap<>(); 

    @Override
    @SuppressWarnings("unchecked")
    public byte[] serialize(Object obj) {
        Class clazz = obj.getClass();
        Schema schema = getSchema(clazz); // 获取消息类型对应的Schema
        byte[] data;
        try {
            data = ProtostuffIOUtil.toByteArray(obj, schema, buffer); // 使用Protostuff将对象序列化为字节数组
        } finally {
            buffer.clear(); // 清空缓冲区
        }
        return data; // 返回序列化后的字节数组
    }

    @Override
    @SuppressWarnings("unchecked")
    public Object deserialize(byte[] bytes, Class<?> clazz) {
        Schema schema = getSchema(clazz); // 获取消息类型对应的Schema
        Object obj = schema.newMessage(); // 创建消息类型对应的新对象
        ProtostuffIOUtil.mergeFrom(bytes, obj, schema); // 使用Protostuff将字节数组反序列化为对象
        return obj; // 返回反序列化后的对象
    }

    @Override
    public int getCode() {
        return SerializerCode.valueOf("PROTOBUF").getCode(); // 获取序列化器的编码
    }

    @SuppressWarnings("unchecked")
    private Schema getSchema(Class clazz) {
        Schema schema = schemaCache.get(clazz); // 从缓存中获取消息类型对应的Schema
        if (Objects.isNull(schema)) {
            // 这个schema通过RuntimeSchema进行懒创建并缓存
            // 所以可以一直调用RuntimeSchema.getSchema(),这个方法是线程安全的
            schema = RuntimeSchema.getSchema(clazz); // 使用RuntimeSchema获取消息类型对应的Schema
            if (Objects.nonNull(schema)) {
                schemaCache.put(clazz, schema); // 将Schema放入缓存中
            }
        }
        return schema; // 返回消息类型对应的Schema
    }
}

```

> 解释一下`protostuff`中的`schema`：
>
> 在 `Protostuff` 中，`Schema` 是用于定义和描述序列化和反序列化对象结构的接口。它定义了如何将 Java 对象与字节数组之间进行转换。Schema 接口提供了以下主要功能：
>
> 1. 对象结构定义：Schema 描述了要序列化和反序列化的对象的结构。它定义了对象的字段、类型和顺序。
> 2. 字段序列化：Schema 定义了将对象的字段值序列化为字节数组的方法。它指定了如何将对象的每个字段转换为字节表示形式。
> 3. 字段反序列化：Schema 定义了将字节数组转换为对象的字段值的方法。它指定了如何从字节表示形式中还原对象的每个字段。
>
> 在 Protostuff 中，可以使用默认的 RuntimeSchema 实现来生成对象的 Schema。RuntimeSchema 使用反射来动态生成和缓存对象的 Schema，使得序列化和反序列化过程更高效。
>
> 通过使用 Schema，Protostuff 可以在不依赖于类的特定注解或继承关系的情况下，根据对象的结构进行序列化和反序列化。这使得 Protostuff 能够处理任意 Java 对象，而不仅限于特定的类层次结构或框架
>
> 
>
> 
>
> 
>
> **懒加载（Lazy Loading）**是一种延迟加载的策略，在需要使用某个资源时才进行加载或初始化，而不是在创建对象或启动应用程序时就立即加载。懒加载的目的是优化性能和节省资源，只有在需要时才进行加载，避免了不必要的初始化或加载操作。
>
> 在 Java 中，懒加载常常用于以下场景：
>
> 1. 对象实例化：延迟实例化对象，只有在需要时才进行对象的创建。这在对象的创建和初始化过程较为耗时或占用较多资源时特别有用。常见的实现方式是通过使用双重检查锁定（Double-Checked Locking）或静态内部类的方式来实现懒加载。
> 2. 资源加载：延迟加载资源，只有在需要时才进行资源的加载。例如，在 Web 应用程序中，可以延迟加载配置文件、数据库连接等资源，在首次访问时才进行加载，以避免启动时的额外开销。
> 3. 数据计算：延迟计算和缓存计算结果，只有在需要时才进行计算。这在复杂的计算或耗时的操作中非常有用。例如，使用缓存机制延迟计算和缓存数据库查询结果，以减少对数据库的频繁访问
>
> 

Protostuff 中的 Schema 概念和 Thrift 中的 IDL 之间有一些相似之处，但也有一些区别。它们都用于定义数据结构和序列化协议，但在实现方式和用途上有一些差异。

相似之处：

1. **数据结构定义**：在两者中，都可以使用相应的方式来定义数据结构，包括字段、类型和顺序等。在 Thrift 的 IDL 文件中，你可以定义消息和字段，而在 Protostuff 中，你可以使用 Java 类和注解来定义。
2. **序列化协议**：Thrift 的 IDL 文件和 Protostuff 的 Schema 都描述了数据在序列化和反序列化过程中的布局、字段标识和类型信息，以确保数据可以正确地进行转换和传输。

区别之处：

1. **IDL vs. Java Annotations**：最显著的区别是定义数据结构的方式。Thrift 使用独立的 IDL 文件来定义数据结构和接口，而 Protostuff 使用 Java 类和注解来定义。Thrift 的 IDL 文件需要通过编译器生成代码，而 Protostuff 则不需要额外的代码生成步骤。
2. **跨语言支持**：Thrift 的 IDL 文件可以通过编译器生成多种编程语言的代码，实现跨语言支持。Protostuff 也可以支持多种语言，但更多地关注于 Java 对象的序列化，不像 Thrift 那样专注于跨语言数据交换。
3. **运行时生成 vs. 编译时生成**：`Protostuff` 的 `Schema` **可以在运行时根据 Java 类的反射信息动态生成**，而 Thrift 的代码生成通常是在编译时完成的。这使得 `Protostuff` 在灵活性上可能更强，但可能会略微降低性能。





**重点 :exclamation:** **什么是运行时动态生成？**

对于序列化库如 `Protostuff`，"运行时生成" 是指根据 Java 类的结构和注解信息，动态地生成用于序列化和反序列化的代码，而**不需要事先在编译时生成代码**(`Thrift`的IDL文件是编译时生成的)。这使得库能够根据具体的数据类来进行序列化，而无需手动为每个数据类编写序列化和反序列化代码

具体来说，运行时生成的过程可能包括以下步骤：

1. **分析类信息**：通过 Java 的反射机制，获取要序列化的数据类的字段信息、数据类型、注解等。
2. **生成序列化代码**：根据类的信息，动态地生成用于序列化的代码，包括字段的写入操作、数据编码等。**这些代码可以使用字节码操作库（如 ASM）来创建新的字节码**
3. **生成反序列化代码**：类似地，生成用于反序列化的代码，包括字段的读取操作、数据解码等——创建出新的字节码。
4. **动态加载类**：将生成的序列化和反序列化代码加载到 JVM 中，使其可用于序列化和反序列化操作。



然后我们再来深究一下：

```java
data = ProtostuffIOUtil.toByteArray(obj, schema, buffer);// 使用Protostuff将对象序列化为字节数组
```

我们可以看到`schema.writeTo(output, message)`才是真正的核心，我们继续追进去看看：

```java
public static <T> byte[] toByteArray(T message, Schema<T> schema, LinkedBuffer buffer){
   if (buffer.start != buffer.offset)
     throw new IllegalArgumentException("Buffer previously used and had not been reset.");
   final ProtostuffOutput output = new ProtostuffOutput(buffer);
   try{
     //这才是实现序列化的核心
      schema.writeTo(output, message);
   }
   catch (IOException e){
       throw new RuntimeException("Serializing to a byte array threw an IOException " +
               "(should never happen).", e);
   }
 return output.toByteArray();
}
```



再往下看：

```java
@Override
public void writeTo(Output output, T message) throws IOException{
         CharSequence value = (CharSequence)us.getObject(message, offset);
         if (value != null)
              output.writeString(number, value, false);
}
```

看到了吧其实就是把序列化对象信息保存成`CharSequence`，然后序列化







#### Thrift

典型的序列化和反序列化过程往往需要如下组件：

- `IDL（Interface description language）文件`：参与通讯的各方需要对通讯的内容需要做相关的约定（Specifications）。为了建立一个与语言和平台无关的约定，这个约定需要采用与具体开发语言、平台无关的语言来进行描述。这种语言被称为接口描述语言（IDL），采用IDL撰写的协议约定称之为IDL文件。
- `IDL Compiler`：IDL文件中约定的内容为了在各语言和平台可见，需要有一个编译器，将IDL文件转换成各语言对应的动态库。
- `Stub/Skeleton Lib`：负责序列化和反序列化的工作代码。Stub是一段部署在分布式系统客户端的代码，一方面接收应用层的参数，并对其序列化后通过底层协议栈发送到服务端，另一方面接收服务端序列化后的结果数据，反序列化后交给客户端应用层；Skeleton部署在服务端，其功能与Stub相反，从传输层接收序列化参数，反序列化后交给服务端应用层，并将应用层的执行结果序列化后最终传送给客户端Stub。
- `Client/Server`：指的是应用层程序代码，他们面对的是IDL所生存的特定语言的class或struct。
- `底层协议栈和互联网`：序列化之后的数据通过底层的传输层、网络层、链路层以及物理层协议转换成数字信号在互联网中传递。



> 那么问题来了，什么是`IDL`？
>
> 
>
> 1. **接口定义**：IDL文件用于定义服务接口、方法和数据类型。你可以在IDL文件中定义服务的方法签名、输入参数、输出参数等信息，以及定义数据结构的字段和类型。这样，IDL文件充当了服务接口的合同，定义了客户端和服务器之间的通信约定。
>
>    
>
> 2. **跨语言支持**：Thrift的IDL是一种中立的描述语言，它可以被编译成多种编程语言的代码，从而实现跨语言的支持。使用IDL文件，你可以定义服务接口和数据结构一次，然后通过编译器生成不同语言的代码，使得客户端和服务器可以使用不同的编程语言来实现。
>
>    
>
> 3. **代码生成**：Thrift的IDL文件可以通过Thrift编译器生成各种编程语言的代码，包括客户端和服务器端的代码、数据结构的定义等。这样，你不需要手动实现序列化和反序列化逻辑，Thrift会自动生成相应的代码，大大简化了开发过程。
>
>    
>
> 4. **序列化和反序列化**：Thrift的IDL文件定义了数据结构的布局、字段类型和顺序，这对于序列化和反序列化数据非常重要。IDL文件中的信息可以帮助Thrift在不同语言之间正确地进行数据交换，确保数据的正确性和一致性



`Thrift`是`Facebook`开源提供的一个高性能，轻量级RPC服务框架，其产生正是为了满足当前大数据量、分布式、跨语言、跨平台数据通讯的需求。 但是，Thrift并不仅仅是序列化协议，而是一个RPC框架。**相对于JSON和XML而言，Thrift在空间开销和解析性能上有了比较大的提升**，对于对性能要求比较高的分布式系统，它是一个优秀的RPC解决方案；**但是由于Thrift的序列化被嵌入到Thrift框架里面，Thrift框架本身并没有透出序列化和反序列化接口，这导致其很难和其他传输层协议共同使用（例如HTTP）**

对于需求为高性能，分布式的RPC服务，Thrift是一个优秀的解决方案。它支持众多语言和丰富的数据类型，并对于数据字段的增删具有较强的兼容性。所以非常适用于作为公司内部的面向服务构建（SOA）的标准RPC框架。

不过Thrift的文档相对比较缺乏，目前使用的群众基础相对较少。另外由于其Server是基于自身的Socket服务，所以在跨防火墙访问时，安全是一个顾虑，所以在公司间进行通讯时需要谨慎。 另外**Thrift序列化之后的数据是Binary数组，不具有可读性，调试代码时相对困难**。最后，由于Thrift的序列化和框架紧耦合，无法支持向持久层直接读写数据，所以不适合做数据持久化序列化协议。



假设我们希望将一个用户信息在多个系统里面进行传递；在应用层，如果采用Java语言，所面对的类对象如下所示：

```java
class Address
{
    private String city;
    private String postcode;
    private String street;
}
public class UserInfo
{
    private Integer userid;
    private String name;
    private List<Address> address;
}
```

我们现在来看看Thrift对应的IDL文件：

```idl
struct Address
{ 
    1: required string city;
    2: optional string postcode;
    3: optional string street;
} 
struct UserInfo
{ 
    1: required string userid;
    2: required i32 name;
    3: optional list<Address> address;
}
```



补充一下，如果使用`Protobuf`做序列化，生成的`IDL`文件如下所示：

```idl
message Address
{
  required string city=1;
  optional string postcode=2;
  optional string street=3;
}
message UserInfo
{
	required string userid=1;
	required string name=2;
	repeated Address address=3;
}
```

它的主要问题在于其**所支持的语言相对较少**，另外由于没有绑定的标准底层传输层协议，在公司间进行传输层协议的调试工作相对麻烦。







#### Protubuf

当使用 Protocol Buffers 来序列化一个 `User` 类时，首先需要定义消息的结构。以下是一个示例代码，包括定义 `User` 消息和生成对应的代码：

假设 `User` 类有以下属性：`id`（整数）、`username`（字符串）、`email`（字符串）和 `isVerified`（布尔值）。

1. 首先，创建一个 `.proto` 文件，例如 `user.proto`：

```protobuf
syntax = "proto3";

message User {
  int32 id = 1;
  string username = 2;
  string email = 3;
  bool isVerified = 4;
}
```

然后，使用 Protocol Buffers 编译器将 `.proto` 文件转换为对应的代码：

假设你已经安装了 Protocol Buffers 工具，可以使用以下命令生成 Java 代码：

```shell
protoc --java_out=./user_directory user.proto
```

这将在 `user_directory` 目录下生成与消息结构对应的 Java 类。

在 Java 代码中使用生成的 `User` 类进行序列化和反序列化：

```java
import com.example.user.User; // 导入生成的 User 类
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

public class ProtobufExample {
    public static void main(String[] args) throws IOException {
        // 创建一个 User 对象
        User user = User.newBuilder()
                .setId(1)
                .setUsername("john_doe")
                .setEmail("john@example.com")
                .setIsVerified(true)
                .build();

        // 将对象序列化到文件
        FileOutputStream output = new FileOutputStream("user.ser");
        user.writeTo(output);
        output.close();

        // 从文件中反序列化对象
        FileInputStream input = new FileInputStream("user.ser");
        User deserializedUser = User.parseFrom(input);
        input.close();

        // 输出反序列化后的对象属性
        System.out.println("Deserialized User:");
        System.out.println("ID: " + deserializedUser.getId());
        System.out.println("Username: " + deserializedUser.getUsername());
        System.out.println("Email: " + deserializedUser.getEmail());
        System.out.println("Is Verified: " + deserializedUser.getIsVerified());
    }
}
```









### 服务注册/提供



服务注册：

```java
public interface ServiceRegistry {
//InetSocketAddress表示网络套接字地址,包含了IP地址和端口号，用于标识网络上的一个节点（主机）
void register(String serviceName, InetSocketAddress inetSocketAddress);
}

//Nacos实现服务注册
@Override
public void register(String serviceName, InetSocketAddress inetSocketAddress) {
        try {
            NacosUtil.registerService(serviceName, inetSocketAddress);//核心
        } catch (NacosException e) {
            logger.error("注册服务时有错误发生:", e);
            throw new RpcException(RpcError.REGISTER_SERVICE_FAILED);
        }
    }
```



服务发现：

```java
public interface ServiceDiscovery {
 //根据服务名查找，返回网络套接字地址
InetSocketAddress lookupService(String serviceName);
}

//Nacos下的实现
@Override
public InetSocketAddress lookupService(String serviceName) {
        try {
            List<Instance> instances = NacosUtil.getAllInstance(serviceName);
            if(instances.size() == 0) {
                logger.error("找不到对应的服务: " + serviceName);
                throw new RpcException(RpcError.SERVICE_NOT_FOUND);
            }
            Instance instance = loadBalancer.select(instances);//通过特定负载均衡策略来选择特定服务的一个活动实例
            return new InetSocketAddress(instance.getIp(), instance.getPort());//返回该服务实例的ip+port
        } catch (NacosException e) {
            logger.error("获取服务时有错误发生:", e);
        }
        return null;
    }
```

在Nacos中，`Instance`（实例）是指一个服务提供者（Provider）在特定的网络地址上运行的实例，当一个服务注册到Nacos注册中心时，它会以实例的形式注册。一个服务可以有多个实例，每个实例在Nacos中都会被表示为一个独立的实体。每个实例都会具有以下属性：

1. **网络地址（IP和端口）**：用于标识实例在网络中的位置，其他服务可以通过这个地址访问该实例提供的服务。
2. **健康状态（Health Status）**：表示实例的健康状况，可以是健康、不健康或未知。
3. **元数据（Metadata）**：包含关于实例的其他信息，比如版本号、环境标识、权重等



### 负载均衡 `LoadBalance`

主要实现了随机、轮询、一致性哈希 三种负载均衡策略：

```java
//需实现的接口
public interface LoadBalancer {
    Instance select(List<Instance> instances);
}


//随机获取服务实例

@Override
public Instance select(List<Instance> instances) {
  /*
  从instances列表中随机选择一个实例，并返回该实例。
  通过随机选择实例，可以实现负载均衡的效果，均匀地分配请求到不同的实例上
  */
    return instances.get(new Random().nextInt(instances.size()));
    }


//轮询获取服务实例  没什么好说的，每次获取第index+1个实例
public Instance select(List<Instance> instances) {
     if(index >= instances.size()) {
            index %= instances.size();
      }
      return instances.get(index++);
    }
```





#### 一致性哈希

着重来看一致性哈希：

关于一致性哈希的基础概念就不再赘述，简单介绍下这个负载均衡中的**一致性哈希环**：首先**将服务器（ip+ 端口号）进行哈希**，映射成环上的一个节点，在请求到来时，根据指定的 `hash key` 同样映射到环上，并顺时针选取最近的一个服务器节点进行请求（在本图中，使用的是 `userId` 作为 `hash key`），当环上的服务器较少时，即使哈希算法选择得当，依旧会遇到大量请求落到同一个节点的问题，为避免这样的问题，大多数一致性哈希算法的实现度引入了**虚拟节点**的概念：<img src="https://cdn.jsdelivr.net/gh/amonstercat/blog-images/%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C.png" style="zoom:80%;" />

在上图中，只有两台物理服务器节点：11.1.121.1 和 11.1.121.2，我们通过添加后缀的方式，克隆出了另外三份节点，使得环上的节点分布的均匀。一般来说，物理节点越多，所需的虚拟节点就越少



在RPC（远程过程调用）中进行负载均衡时，一致性哈希策略通常在以下情况下被考虑：

1. **动态节点的加入和离开**：当RPC系统中的节点（服务提供者）数量发生变化时，例如**有新的节点加入或某个节点离开，传统的负载均衡算法（如轮询或随机）可能会导致大量的请求重新分配给不同的节点**，从而影响系统的稳定性和性能。使用一致性哈希策略可以在节点变动时，尽可能保持请求的分配相对稳定，减少不必要的重新分配
2. **缓存和会话保持**：在某些应用场景中，需要保持特定请求或会话的一致性，**即同一个请求或会话始终路由到同一个节点上**。使用一致性哈希策略可以在节点变动时，尽可能保持特定请求或会话的路由一致性，避免因节点变动而导致**缓存失效**或会话中断(**某个查询请求在一个特定实例上缓存过，如果该请求再次传来打到其他节点上那缓存就用不上了**)
3. **数据分片**：当RPC系统中的数据需要进行分片存储时，例如将数据按照某个关键字哈希值的范围划分到不同的节点上，一致性哈希策略可以保证数据的均匀分布，同时在节点变动时尽可能减少数据的迁移量





本项目中关于一致性哈希的实现如下：

**虚拟节点设置的越多，当某个实例挂掉后，环上受到影响的请求的范围就会越小，这里设置为了160**

```java

public class IpHashLoadBalancer implements LoadBalancer{
    @Override
    public Instance select(List<Instance> instances , String address) {
        ConsistentHash ch = new ConsistentHash(instances, 160);//虚拟节点设置为了160
        HashMap<String,Instance> map = ch.map;
        return map.get(ch.getServer(address));
    }
}



//一致性hash 利用treemap实现
class ConsistentHash {
    
    //TreeMap中的key表示服务器的hash值，value表示服务器ip。模拟一个哈希环
    private static TreeMap<Integer,String> Nodes = new TreeMap();
    
    private static int VIRTUAL_NODES = 160;//虚拟节点个数，用户指定，默认160
    private static List<Instance> instances = new ArrayList<>();//真实物理节点集合
    public ConsistentHash(List<Instance> instances, int VIRTUAL_NODES){
        this.instances = instances;
        this.VIRTUAL_NODES = VIRTUAL_NODES;
    }
 
    public static HashMap<String,Instance> map = new HashMap<>();//将服务实例与ip地址一一映射
    //预处理 形成哈希环
    static {
        //程序初始化，将所有的服务器(真实节点与虚拟节点)放入Nodes（底层为红黑树）中
        for (Instance instance : instances) {
            String ip = instance.getIp();
            
            Nodes.put(getHash(ip),ip); //环上放入真实节点
            
            map.put(ip,instance);  // map中存放 真实节点的ip——Instance对应信息
            
            for(int i = 0; i < VIRTUAL_NODES; i++) {
                int hash = getHash(ip+"#"+i);
                Nodes.put(hash,ip); //环TreeMap上放入每个真实节点对应的虚拟节点
            }
        }
    }
    
    
    //得到请求被分配到的实例 —— 对应的Ip地址
    public  String getServer(String clientInfo) {
        int hash = getHash(clientInfo); // 计算客户端请求的hash,不需要放入环上！
        
        //得到大于该Hash值的子红黑树——————因为红黑树满足 AVL&&BST 
        SortedMap<Integer,String> subMap = Nodes.tailMap(hash);
        
        //获取该子树最小元素 ，即顺时针的第一个实例
        Integer nodeIndex = subMap.firstKey();
        
        //没有大于该元素的子树 取整树的第一个元素(想象成一个环，超过尾部则取第一个 key)
        if (nodeIndex == null) {
            nodeIndex = Nodes.firstKey();
        }   
         String virtualNodeIp = Nodes.get(nodeIndex);//通过虚拟节点的哈希值找到对应真实节点的IP
        
        return  map.get(virtualNodeIp); //// 返回真实节点对应的Instance
    }
    
    
    
    //使用FNV1_32_HASH算法计算服务器的Hash值,这里不使用重写hashCode的方法，最终效果没区别
    private static int getHash(String str) {
        final int p = 16777619;
        int hash = (int) 2166136261L;
        for (int i = 0; i < str.length(); i++) {
            hash = (hash^str.charAt(i))*p;
            hash +=hash <<13;
            hash ^=hash >>7;
            hash +=hash <<3;
            hash ^=hash >>17;
            hash +=hash <<5;
            //如果算出来的值为负数 取其绝对值
            if(hash < 0) {
                hash = Math.abs(hash);
            }
        }
        return hash;
    }
}
```



**注意：**

1、这里的`LoadBalancer` 接口的抽象方法内多增加了一个参数`Address`，这是因为我们**之前实现的轮询和随机算法与客户端的地址无关**，而我们**这里是同一个客户端在服务端没有发生变化的前提下都要打到同一台服务器上，这与客户端的地址有关**

2、哈希环采用`TreeMap`红黑树来模拟实现，在`TreeMap`的`api`中有一个`tailMap`() 函数，输入一个`fromKey`，输出的是一个`SortedMap`有序Map，再使用`firstKey`()拿到最近(顺时针第一个)一个大于客户端请求对应哈希值的元素。
3、用虚拟节点来防止某个节点的流量过大而导致的一些问题

4、使用`Hashmap`将服务实例和`ip`地址对应，因为`getService`()得到的是`ip`地址，而我们最后需要返回的是`instance`，可以看到，每个真实节点和其对应的虚拟节点均对应着同一个`instance`

5、实例化一致性哈希环时需要传入实例列表`Instances`和每个实例对应的虚拟节点个数

6、`TreeMap`是基于红黑树实现的，可以保持键的有序性。一致性哈希算法需要将节点和请求映射到哈希环上，并按照顺时针方向进行路由。`TreeMap`的有序性可以方便地获取顺时针方向上的下一个节点，使得请求的路由能够在环上进行快速查找和定位



> 还有一个很重要的问题，**如果客户端请求被映射到某个实例的虚拟节点上，那么如何通过虚拟节点来复原真实节点呢？**

核心代码都在下面了：

```java
public String getServer(String clientInfo) {
    int hash = getHash(clientInfo); // 计算客户端请求的hash，不需要放入环上！
    
    // 得到大于等于该Hash值的子红黑树，即从该虚拟节点开始顺时针方向的部分红黑树
    SortedMap<Integer, String> subMap = Nodes.tailMap(hash);
    
    // 获取该子树的第一个虚拟节点的哈希值
    Integer virtualNodeHash = subMap.firstKey();
    
    // 没有大于等于该哈希值的子树，取整个环的第一个虚拟节点的哈希值（即第一个真实节点）
    if (virtualNodeHash == null) {
        virtualNodeHash = Nodes.firstKey();
    }

    // 根据虚拟节点的哈希值找到对应的真实节点
    String virtualNodeIp = Nodes.get(virtualNodeHash);
    
    // 返回真实节点对应的Instance
    return map.get(virtualNodeIp);
}
```

1. 先通过虚拟节点的Hash值找到对应真实节点的IP值：

   ```java
   //TreeMap中的key表示服务器的hash值，value表示服务器ip。模拟一个哈希环
   private static TreeMap<Integer,String> Nodes = new TreeMap();
   
   String virtualNodeIp = Nodes.get(virtualNodeHash);
   ```

2. 然后再通过存放真实节点Ip和`Instance`实例对应关系的`HashMap`来进行查找

   ```java
   public static HashMap<String,Instance> map = new HashMap<>();//将服务实例与ip地址一一映射
   // 返回真实节点对应的Instance
   return map.get(virtualNodeIp);
   ```

   







### 网络传输 `Transport`

首先来看看RPC-Client和RPC-Server各需要做什么事情：

RPC-Client：

```java
Object sendRequest(RpcRequest rpcRequest);//发送RPC调用请求
```

RPC的客户端通常使用**代理对象**将远程方法调用转发给实际的RPC服务：

它是通过动态代理技术实现的，具体的工作流程如下：

1. **创建代理对象：** 在RPC-Client中，通常会使用Java的动态代理技术创建一个代理对象。**代理对象实现了与RPC服务接口相同的接口，并可以处理对这些接口方法的调用**。
2. **远程方法调用转发：** 当应用程序通过代理对象调用接口方法时，实际上是调用了代理对象的方法。代理对象会将这个方法调用转发给RPC框架，作为一次远程方法调用。
3. **封装RPC请求：** 代理对象会将接口方法调用转换为一个RPC请求对象，并将接口名、方法名、参数等信息封装到请求对象中。
4. **序列化请求对象：** 代理对象会对RPC请求对象进行序列化，将其转换为字节流的形式，以便在网络上传输。
5. **发送RPC请求：** 代理对象会将序列化后的RPC请求通过网络发送给RPC-Server，即远程的RPC服务提供者。
6. **接收RPC响应：** 代理对象会等待并接收RPC-Server返回的响应，该响应包含了远程方法的执行结果或其他信息。
7. **解析RPC响应：** 代理对象会对接收到的RPC响应进行解析，将其反序列化为响应对象。
8. **返回方法结果：** 代理对象将从RPC响应中提取出的方法结果返回给调用方，作为对接口方法调用的结果



RPC-Server:

```java
void start(); //启动服务器(socket启动 netty启动)
<T> void publishService(T service, String serviceName);//发布服务到注册中心
```



#### Sokcet实现

1、客户端

```java
public class SocketClient implements RpcClient {

    //传入自选序列化器及负载均衡机制  
    public SocketClient(Integer serializer, LoadBalancer loadBalancer) {
        this.serviceDiscovery = new NacosServiceDiscovery(loadBalancer);
        this.serializer = CommonSerializer.getByCode(serializer);
    }



    //核心： 发送调用请求 
    @Override
    public Object sendRequest(RpcRequest rpcRequest) {
        if(serializer == null) {
            logger.error("未设置序列化器");
            throw new RpcException(RpcError.SERIALIZER_NOT_FOUND);
        }

        // 通过服务发现查找服务的网络地址,这一步nacos通过特定负载均衡策略来选择该服务的一个活动实例      
        InetSocketAddress inetSocketAddress = serviceDiscovery.lookupService(rpcRequest.getInterfaceName());
        try (Socket socket = new Socket()) {
            socket.connect(inetSocketAddress);// 连接服务提供者
            OutputStream outputStream = socket.getOutputStream();//获取输出流发送数据到对端
            InputStream inputStream = socket.getInputStream();//得到输入流获取对端响应数据
            ObjectWriter.writeObject(outputStream, rpcRequest, serializer);//将请求对象序列化后发送到服务提供者
            Object obj = ObjectReader.readObject(inputStream);//从服务提供者接收并反序列化响应对象
            RpcResponse rpcResponse = (RpcResponse) obj;
            if (rpcResponse == null) {
                logger.error("服务调用失败，service：{}", rpcRequest.getInterfaceName());
                throw new RpcException(RpcError.SERVICE_INVOCATION_FAILURE, " service:" + rpcRequest.getInterfaceName());
            }
            if (rpcResponse.getStatusCode() == null || rpcResponse.getStatusCode() != ResponseCode.SUCCESS.getCode()) {
                logger.error("调用服务失败, service: {}, response:{}", rpcRequest.getInterfaceName(), rpcResponse);
                throw new RpcException(RpcError.SERVICE_INVOCATION_FAILURE, " service:" + rpcRequest.getInterfaceName());
            }
            RpcMessageChecker.check(rpcRequest, rpcResponse);
            return rpcResponse;
        } catch (IOException e) {
            logger.error("调用时有错误发生：", e);
            throw new RpcException("服务调用失败: ", e);
        }
    } 
}
```

> **`socket.getOutputStream()`**：
>
> 这个方法返回一个`OutputStream`对象，用于向Socket的对端发送数据。通过获取输出流，可以将数据写入到Socket的输出流中，然后通过网络发送给对端。
>
> 示例用法：
>
> ```
> OutputStream outputStream = socket.getOutputStream();
> ```
>
> **`socket.getInputStream()`**：
>
> 这个方法返回一个`InputStream`对象，用于从Socket的对端接收数据。通过获取输入流，可以从Socket的输入流中读取数据，这些数据是对端通过网络发送过来的。
>
> 示例用法：
>
> ```
> InputStream inputStream = socket.getInputStream();
> ```



RPC客户端发送请求的主要步骤包括：

1. 检查是否设置了序列化器（serializer），如果没有则抛出异常
2. 通过服务发现（serviceDiscovery）查找指定接口名（rpcRequest.getInterfaceName()）的服务的网络地址（InetSocketAddress），**因为RpcServer在启动时，会将每个方法和当前运行的RpcServer进行绑定**
3. 使用Socket与服务提供者建立连接**(这时只是连接到要调用的方法所属的服务器实例，建立连接后才可以调用服务器端的特定方法)**
4. 获取Socket的输入流和输出流，用于发送请求和接收响应
5. 将请求对象（rpcRequest）使用指定的序列化器（serializer）进行序列化，并通过输出流发送给服务提供者
6. 从输入流中读取响应数据，并反序列化为RpcResponse对象
7. 检查响应对象，如果为空或者状态码不是成功（ResponseCode.SUCCESS.getCode()）则抛出异常
8. 执行RpcMessageChecker的检查，验证请求和响应是否匹配
9. 返回响应结果RpcResponse





2、服务端(稍微复杂些)

```java
public void start() {
    try (ServerSocket serverSocket = new ServerSocket()) {
        serverSocket.bind(new InetSocketAddress(host, port));//创建一个ServerSocket实例，绑定到指定主机（host）和端口（port）
        logger.info("服务器启动……");
        ShutdownHook.getShutdownHook().addClearAllHook();//先清空Nacos中的所有服务
        Socket socket;
        while ((socket = serverSocket.accept()) != null) {
            logger.info("消费者连接: {}:{}", socket.getInetAddress(), socket.getPort());//循环等待客户端连接

            /*
                创建一个新的SocketRequestHandlerThread线程，
                并将连接的Socket、请求处理器（requestHandler）和序列化器（serializer）传递给该线程进行处理
                */
            threadPool.execute(new SocketRequestHandlerThread(socket, requestHandler, serializer));
        }
        threadPool.shutdown();//接受不到新的连接时，关闭线程池
    } catch (IOException e) {
        logger.error("服务器启动时有错误发生:", e);
    }
}
```

> ```java
> executor.execute(task);
> ```
>
> 上述代码中，`executor`是一个`ExecutorService`实例，`task`是一个实现了`Runnable`接口的任务对象。通过调用`execute()`方法，将`task`提交给`executor`进行执行
>
> 需要注意的是，`execute()`方法是一种简单的提交任务的方式，它通常用于不需要获取任务执行结果的场景。如果需要获取任务的执行结果，可以考虑使用`submit()`方法，该方法会返回一个`Future`对象，可以通过该对象获取任务的执行结果或取消任务的执行
>
> 总之，`executor.execute()`方法用于向执行器提交一个任务进行执行，是一种异步执行任务的方式，适用于不需要获取任务执行结果的情况





涉及到的线程`SocketRequestHandlerThread`处理方法为：

```java
public class SocketRequestHandlerThread implements Runnable {
 
/*监听的 socket 和真正用来传送数据的 socket，是「两个」 socket，一个叫作监听 socket，一个叫作已完成连接 socket
成功连接建立之后，双方开始通过 read 和 write 函数来读写数据，就像往一个文件流里面写东西一样   
*/
private Socket socket;  //真正用来传输数据的Socket
public void run() {
        try (InputStream inputStream = socket.getInputStream();
            OutputStream outputStream = socket.getOutputStream()) {
            RpcRequest rpcRequest = (RpcRequest) ObjectReader.readObject(inputStream);
            Object result = requestHandler.handle(rpcRequest);//核心
            RpcResponse<Object> response = RpcResponse.success(result, rpcRequest.getRequestId());
            ObjectWriter.writeObject(outputStream, response, serializer);
        } catch (IOException e) {
            logger.error("调用或发送时有错误发生：", e);
        }
    }
}


//RequestHandler 真正处理RPC请求的方法
public class RequestHandler {
 public Object handle(RpcRequest rpcRequest) {
        Object service = serviceProvider.getServiceProvider(rpcRequest.getInterfaceName());
        return invokeTargetMethod(rpcRequest, service);
    }

    private Object invokeTargetMethod(RpcRequest rpcRequest, Object service) {
        Object result;
        try {
            Method method = service.getClass().getMethod(rpcRequest.getMethodName(), rpcRequest.getParamTypes());
            result = method.invoke(service, rpcRequest.getParameters());
            logger.info("服务:{} 成功调用方法:{}", rpcRequest.getInterfaceName(), rpcRequest.getMethodName());
        } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
            return RpcResponse.fail(ResponseCode.METHOD_NOT_FOUND, rpcRequest.getRequestId());
        }
        return result;
    }

}
```



RPC服务端处理请求的主要步骤包括：

- 创建一个ServerSocket实例，并绑定到指定的主机（host）和端口（port）。
- 打印服务器启动的日志信息。
- 向ShutdownHook中添加清除所有资源的钩子（hook）。
- 进入一个循环，不断接受客户端的连接。当有客户端连接成功时，会打印相应的连接信息。
- 通过线程池（threadPool）创建一个新的SocketRequestHandlerThread线程，并将连接的Socket、请求处理器（requestHandler）和序列化器（serializer）传递给该线程进行处理
- 该线程根据RPC请求中的接口名获取对应的服务提供者对象，然后使用反射调用该服务提供者对象的方法，将RPC请求的参数传递给方法进行处理，并返回执行结果
- 最后，当接受不到新的连接时，关闭线程池





#### Netty实现 :smile_cat:



**服务端**

```java
//NIO方式服务提供侧 
public class NettyServer extends AbstractRpcServer {
private static final EventLoopGroup bossGroup = new NioEventLoopGroup();
private static final EventLoopGroup workerGroup = new NioEventLoopGroup();

public void start() {
ShutdownHook.getShutdownHook().addClearAllHook();
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
try {
//创建ServerBootstrap对象，用于配置和启动服务器
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(bossGroup, workerGroup)
.channel(NioServerSocketChannel.class)//通道类型 NIO
.handler(new LoggingHandler(LogLevel.INFO))
.option(ChannelOption.SO_BACKLOG, 256)
.option(ChannelOption.SO_KEEPALIVE, true)
.childOption(ChannelOption.TCP_NODELAY, true)
.childHandler(new ChannelInitializer<SocketChannel>() {
@Override
protected void initChannel(SocketChannel ch) throws Exception {
ChannelPipeline pipeline = ch.pipeline();
pipeline.addLast(new IdleStateHandler(30, 0, 0, TimeUnit.SECONDS))
.addLast(new CommonEncoder(serializer))
.addLast(new CommonDecoder())
.addLast(new NettyServerHandler());
}
});
//调用ServerBootstrap的bind方法绑定主机（host）和端口（port），
//使当前线程同步等待，直到绑定操作完成或发生异常,
//返回一个 ChannelFuture 对象，表示绑定操作的结果
ChannelFuture future = serverBootstrap.bind(host, port).sync();
future.channel().closeFuture().sync();

} catch (InterruptedException e) {
logger.error("启动服务器时有错误发生: ", e);
} finally {

//优雅关闭：停止接受新的任务并等待已经提交的任务完成，最后释放资源
bossGroup.shutdownGracefully();
workerGroup.shutdownGracefully();
}
}

}
```



仔细来分析一下：

```java
private static final EventLoopGroup bossGroup = new NioEventLoopGroup();
private static final EventLoopGroup workerGroup = new NioEventLoopGroup();
```

以上代码定义了两个静态的`EventLoopGroup`对象：`bossGroup`和`workerGroup`。这些对象是Netty框架中的线程池，用于处理网络事件和IO操作,其中：

- `bossGroup`：用于处理**接受客户端连接的事件**循环组。它**负责监听和接受客户端的连接请求**，**并将连接分配给`workerGroup`中的线程进行处理**。通常，`bossGroup`使用单个线程
- `workerGroup`：用于处理客户端请求的事件循环组。一旦连接被接受，**`workerGroup`中的线程负责处理客户端的请求、读写操作以及其他的网络事件**。可以配置`workerGroup`的线程数，以适应并发处理的需求

(1)同时使用`NioEventLoopGroup`作为`EventLoopGroup`的实现，它基于NIO（非阻塞IO）模型，提供高性能的IO处理能力。通过将`bossGroup`和`workerGroup`分离，可以有效地处理客户端的连接和请求，并且提供了更好的可伸缩性和并发性能

(2)使用`private static final`修饰是因为这些线程池对象应该在整个应用程序的生命周期中共享和重用，静态和不可变的`EventLoopGroup`对象在应用程序中应该仅创建一次，以避免资源浪费和重复创建



```java
//用于初始化新创建的 Channel 对象的处理器链（即 ChannelPipeline）。这个方法是在每个新的 Channel 被创建时调用，用于配置 Channel 的处理逻辑
.childHandler(new ChannelInitializer<SocketChannel>() {
@Override
protected void initChannel(SocketChannel ch) throws Exception {
ChannelPipeline pipeline = ch.pipeline();ChannelPipeline //用来处理网络通信过程中的数据的管道，它将不同的处理器按顺序连接起来，形成一个处理链
pipeline.addLast(new IdleStateHandler(30, 0, 0, TimeUnit.SECONDS))
.addLast(new CommonEncoder(serializer))
.addLast(new CommonDecoder())  //传入协议编码器、解码器
.addLast(new NettyServerHandler()); //Netty处理Rpc-Request请求的Handler
}
});
```

上述代码则用于初始化新创建的 `Channel` 对象的处理器链（即 `ChannelPipeline`）。这个方法是在每个新的 `Channel` 被创建时调用，用于配置 `Channel` 的处理逻辑

> `ChannelFuture` 具有以下用途：
>
> 1. 异步操作的状态和结果：`ChannelFuture` 可以用于获取异步操作的状态和结果。通过调用 `ChannelFuture` 提供的方法，可以判断操作是否完成、操作是否成功、获取操作返回的结果等。这使得在进行异步操作时能够及时获取和处理操作的结果。
> 2. 异步操作的完成事件：`ChannelFuture` 可以用于注册一个监听器，以便在异步操作完成时触发相应的事件处理逻辑。通过注册监听器，可以在操作完成时执行特定的回调方法，从而实现异步操作的结果处理或执行后续操作。
> 3. 链式操作的协调：`ChannelFuture` 支持方法链的方式，使得可以将多个异步操作串联起来，并在前一个操作完成后触发下一个操作。通过连续调用 `ChannelFuture` 提供的方法，可以实现异步操作的顺序执行和依赖关系的协调。



对应的`NettyServerHandler`为

1、处理从网络通道读取到的 `RpcRequest` 消息:

```java

protected void channelRead0(ChannelHandlerContext ctx, RpcRequest msg) throws Exception {

try {
if(msg.getHeartBeat()) {
//如果是心跳包，则仅记录日志并返回，不进行进一步处理
logger.info("接收到客户端心跳包...");
return;
}
logger.info("服务器接收到请求: {}", msg);
/*
调用 requestHandler.handle(msg) 处理该请求消息，并获取处理结果 result。
检查当前通道是否是活跃的并且可写的。
如果是，则通过 ctx.writeAndFlush() 将 RpcResponse.success(result, msg.getRequestId()) 发送回客户端
*/
Object result = requestHandler.handle(msg);
if (ctx.channel().isActive() && ctx.channel().isWritable()) {
ctx.writeAndFlush(RpcResponse.success(result, msg.getRequestId()));
} else {
logger.error("通道不可写");
}
} finally {
ReferenceCountUtil.release(msg);//释放对请求消息 msg 的引用，以确保内存的正确释放
}
}
```



2、监听所有客户端发送的心跳包：

```java
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleState state = ((IdleStateEvent) evt).state();
            if (state == IdleState.READER_IDLE) {
                logger.info("长时间未收到心跳包，断开连接...");
                ctx.close();
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
```

当接收到一个事件时，首先判断该事件是否为`IdleStateEvent`类型，即判断是否是空闲状态事件。如果是空闲状态事件，代码会进一步判断空闲状态的类型

如果空闲状态是`READER_IDLE`，也就是在一段时间内未接收到数据包，表示长时间未收到心跳包，那么会记录一条信息日志，提示长时间未收到心跳包，并关闭与客户端的连接（通过`ctx.close()`方法）

如果接收到的事件不是空闲状态事件，那么会调用父类的`userEventTriggered`方法来处理其他类型的事件



对应的客户端发送心跳包的逻辑如下：

```java

public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {

    if (evt instanceof IdleStateEvent) {//先判断是否为空闲状态事件？
        IdleState state = ((IdleStateEvent) evt).state();
        if (state == IdleState.WRITER_IDLE) {
            // 如果空闲状态是WRITER_IDLE，即一段时间内没有向远程服务器写入数据，则需要发送心跳包来保持连接
            logger.info("发送心跳包 [{}]", ctx.channel().remoteAddress());
            //获取与远程服务器的通信通道（channel）
            Channel channel = ChannelProvider.get((InetSocketAddress) ctx.channel().remoteAddress(), CommonSerializer.getByCode(CommonSerializer.DEFAULT_SERIALIZER));
            RpcRequest rpcRequest = new RpcRequest();
            rpcRequest.setHeartBeat(true);//标识这是一个心跳包请求
            channel.writeAndFlush(rpcRequest).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);//将心跳包请求(rpcRequest)发送出去
        }
    } else {
        super.userEventTriggered(ctx, evt);//如果接收到的事件不是空闲状态事件，那么会调用父类的userEventTriggered方法来处理其他类型的事件
    }
}
```















**客户端**

在自定义 RPC 框架中使用 Netty，`channelRead0` 方法通常起到处理接收到的网络数据，即 RPC 请求和响应的作用。RPC 框架的客户端和服务器端都会使用 Netty 来传输数据，而 `channelRead0` 方法就是用来处理这些传输的数据。

在 RPC 框架中，客户端会将 RPC 请求序列化后发送到服务器端，服务器端接收到请求后会反序列化，并根据请求内容执行相应的服务。然后服务器会将执行结果序列化后发送给客户端。`channelRead0` 方法在服务器端的 `ChannelHandler` 中用于处理接收到的 RPC 请求数据，执行相应的服务逻辑，并发送响应数据。

对于 RPC 客户端，异步执行的结果通常是通过使用 `CompletableFuture` 来实现的。当客户端发送 RPC 请求后，可以在 `channelRead0` 方法中处理服务器端返回的响应数据，并将响应结果设置到相应的 `CompletableFuture` 中。这样，在客户端代码中，你可以通过监听这个 `CompletableFuture` 来异步等待并获取服务器端的执行结果。



来看看服务端的`channelRead0()`方法：

```java

@Override
protected void channelRead0(ChannelHandlerContext ctx, RpcRequest msg) throws Exception {
    //用于处理从网络通道读取到的 RpcRequest 消息
    try {
        if(msg.getHeartBeat()) {
            //如果是心跳包，则仅记录日志并返回，不进行进一步处理
            logger.info("接收到客户端心跳包...");
            return;
        }
        logger.info("服务器接收到请求: {}", msg);
        /*
          调用 requestHandler.handle(msg) 处理该请求消息，并获取处理结果 result。
       检查当前通道是否是活跃的并且可写的。
      如果是，则通过 ctx.writeAndFlush() 将 RpcResponse.success(result, msg.getRequestId()) 发送回客户端
             */
        Object result = requestHandler.handle(msg);
        if (ctx.channel().isActive() && ctx.channel().isWritable()) {
            ctx.writeAndFlush(RpcResponse.success(result, msg.getRequestId()));
        } else {
            logger.error("通道不可写");
        }
    } finally {
        ReferenceCountUtil.release(msg);//释放对请求消息 msg 的引用，以确保内存的正确释放
    }
}

```



再来看看客户端的`channelRead0()`方法：

```java
@Override
protected void channelRead0(ChannelHandlerContext ctx, RpcResponse msg) throws Exception {
        try {
            logger.info(String.format("客户端接收到消息: %s", msg));
            unprocessedRequests.complete(msg);
        } finally {
            ReferenceCountUtil.release(msg);
        }
    }


//用于将某个 RPC 响应（RpcResponse）设置到相应的 CompletableFuture 对象中，从而完成异步的响应。
    public void complete(RpcResponse rpcResponse) {
        CompletableFuture<RpcResponse> future = unprocessedResponseFutures.remove(rpcResponse.getRequestId());
        if (null != future) {
            /*
            如果找到了对应的 CompletableFuture，则调用其 complete 方法，
            将 RPC 响应对象作为参数传递进去，表示异步操作已经完成，并设置结果为该响应。
             */
            future.complete(rpcResponse);
        } else {
            throw new IllegalStateException();
        }
    }
```

注意上述代码中的`complete()`方法：用于将某个 RPC 响应（`RpcResponse`）设置到相应的 `CompletableFuture` 对象中，从而完成异步的响应。具体来说它做了以下几件事情：

1. 从 `unprocessedResponseFutures` 中根据 RPC 响应的 `requestId` 获取对应的 `CompletableFuture` 对象。
2. 如果找到了对应的 `CompletableFuture`，则调用其 `complete` 方法，将 RPC 响应对象作为参数传递进去，表示异步操作已经完成，并设置结果为该响应。
3. 如果在 `unprocessedResponseFutures` 中找不到对应的 `CompletableFuture`，则抛出 `IllegalStateException` 异常，表示出现了不正常的状态。

这段代码的目的是为了在客户端的异步发送 RPC 请求后，能够在服务器端返回响应时，将响应结果设置到对应的 `CompletableFuture` 中，从而通知客户端的异步操作已经完成，可以获取到相应的结果。通常，这种方式被用于实现一个基于 `CompletableFuture` 的异步 RPC 调用





test-client文件夹：测试用的客户端项目

test-server文件夹：测试用的服务端项目





## 常见问题



### **多种序列化框架对比(包含`Thrift`)**

`Hessian`、`Kryo` 和 `Protostuff` 都是常见的序列化框架，它们在序列化的实现和特性上存在一些区别

1. 序列化效率：Kryo 和 Protostuff 通常被认为是比 Hessian 更高效的序列化框架。Kryo 是一个高性能的序列化库，它使用基于字节码生成的序列化代码，可以快速地序列化和反序列化对象。Protostuff 利用 RuntimeSchema 在运行时动态生成和缓存对象的 Schema，也具有较高的序列化和反序列化性能。相比之下，Hessian 使用基于反射的方式，性能相对较低。
2. 序列化体积：Kryo 和 Protostuff 通常能够产生较小的序列化体积，因为它们在序列化时能够更紧凑地表示数据。这主要是因为 Kryo 和 Protostuff 都采用了二进制格式，以及使用了一些优化技术如字段压缩、引用共享等。Hessian 使用的是基于文本的序列化格式，因此生成的序列化数据体积相对较大。
3. 类型支持：Hessian 是一种基于类似于 Java 的类型系统的序列化格式，它可以直接序列化和反序列化 Java 对象和基本类型。Kryo 和 Protostuff 则需要预先注册或配置需要序列化的类或对象。Kryo 和 Protostuff 支持更灵活的对象结构，可以序列化没有默认构造函数、匿名类、内部类等



在 Java 中使用 Thrift 进行序列化，需要进行以下步骤：

1. 定义数据结构：使用 Thrift 的 IDL（Interface Definition Language）语言编写数据结构的定义文件（.thrift），描述要序列化的数据结构和接口定义
2. 生成代码：使用 Thrift 编译器（thrift）根据定义文件生成对应的 Java 代码。生成的代码包括数据结构的类定义、序列化和反序列化方法等
3. 序列化和反序列化：使用生成的 Thrift 类，通过调用序列化方法（如 `TBase.write()`）将对象序列化为字节数组，或调用反序列化方法（如 `TBase.read()`）将字节数组反序列化为对象

而`Thrift` 的序列化和其他序列化技术（如 Protobuf、JSON、Kryo 等）的区别主要体现在以下几个方面：

1. 序列化格式：Thrift 使用自己的二进制序列化格式，它将数据结构和字段信息编码为紧凑的字节流。其他序列化技术使用不同的序列化格式，如 Protobuf 使用二进制格式，JSON 使用文本格式，Kryo 使用基于字节码的格式等。
2. 数据结构定义：Thrift 使用 IDL 文件来定义数据结构和接口，通过编译器生成对应的类和方法。其他序列化技术通常使用类注解、配置或代码方式来描述数据结构和字段。
3. 跨语言支持：Thrift 是一个跨语言的序列化技术，可以在不同的编程语言之间进行数据交换。Thrift 提供了多种语言的支持，如 Java、C++、Python 等。其他序列化技术的跨语言支持程度不同，有些只适用于特定语言。
4. 功能扩展：Thrift 提供了丰富的特性，如动态架构演化、服务治理、数据压缩等。它可以根据定义文件进行动态的架构演化，支持版本兼容性和数据迁移。其他序列化技术的功能可能更加简化，仅专注于序列化和反序列化。



假设我们希望将一个用户信息在多个系统里面进行传递；在应用层，如果采用Java语言，所面对的类对象如下所示：

```java
class Address
{
    private String city;
    private String postcode;
    private String street;
}
public class UserInfo
{
    private Integer userid;
    private String name;
    private List<Address> address;
}
```

我们现在来看看`Thrift`对应的IDL文件：

```idl
struct Address
{ 
    1: required string city;
    2: optional string postcode;
    3: optional string street;
} 
struct UserInfo
{ 
    1: required string userid;
    2: required i32 name;
    3: optional list<Address> address;
}
```

补充一下，如果使用`Protobuf`做序列化，生成的`IDL`文件如下所示：

```idl
message Address
{
  required string city=1;
  optional string postcode=2;
  optional string street=3;
}
message UserInfo
{
	required string userid=1;
	required string name=2;
	repeated Address address=3;
}
```

它的主要问题在于其**所支持的语言相对较少**，另外由于没有绑定的标准底层传输层协议，在公司间进行传输层协议的调试工作相对麻烦





最后来看一下`ProtoStuff`：





















### 全局唯一性ID问题 :crying_cat_face:

传统的单体架构的时候，基本是单库然后业务单表的结构。每个业务表的ID一般我们都是从1增，通过`AUTO_INCREMENT=1`设置自增起始值，但是在分布式服务架构模式下分库分表的设计，使得多个库或多个表存储相同的业务数据。这种情况根据数据库的自增ID就会产生相同ID的情况，不能保证主键的唯一性

![](https://cdn.jsdelivr.net/gh/amonstercat/blog-images/arch-z-id-1.png)



如上图，如果第一个订单存储在 DB1 上则订单 ID 为1，当一个新订单又入库了存储在 DB2 上订单 ID 也为1。我们系统的架构虽然是分布式的，但是在用户层应是无感知的，重复的订单主键显而易见是不被允许的



为什么要考虑分布式ID的唯一性：

1. **避免冲突：** 如果多个节点生成了相同的ID，那么在数据存储、任务分配或消息传递时可能会发生冲突。唯一的ID可以确保每个请求或任务都有一个独特的标识，避免了这类冲突问题
2. **数据一致性：** 在分布式系统中，数据的一致性非常重要。唯一的ID可以用作数据记录的主键，确保在不同节点上操作相同数据时不会出现数据混乱或错误
3. **请求跟踪：** 通过在请求中使用唯一ID，系统可以追踪请求的处理过程，从而更容易进行故障排除和性能分析
4. **幂等性：** 在分布式系统中，网络通信可能会出现重试或失败。唯一的ID可以确保即使请求在网络中发生故障并重试，系统也可以保持幂等性，即对于相同的请求只产生一次结果
5. **防止重复处理：** 如果不使用唯一ID，当网络通信异常时，可能会导致请求被重复处理，造成不必要的副作用





再来看看雪花算法：

`SnowFlake`算法生成id的结果是一个64bit大小的整数，它的结构如下图：

![snowflake](https://cdn.jsdelivr.net/gh/amonstercat/blog-images/snowflake.webp)

- `1位`，不用。二进制中最高位为1的都是负数，但是我们生成的id一般都使用整数，所以这个最高位固定是0
- `41位`，用来记录时间戳（毫秒）。
  - 41位可以表示$2^{41}-1$个数字，
  - 如果只用来表示正整数（计算机中正数包含0），可以表示的数值范围是：0 至 $2^{41}-1$，减1是因为可表示的数值范围是从0开始算的，而不是1。
  - 也就是说41位可以表示$2^{41}-1$个毫秒的值，转化成单位年则是$(2^{41}-1) / (1000 * 60 * 60 * 24 * 365) = 69$年
- `10位`，用来记录工作机器id。
  - 可以部署在$2^{10} = 1024$个节点，**包括`5位datacenterId`和`5位workerId`**
  - `5位（bit）`可以表示的最大正整数是$2^{5}-1 = 31$，即可以用0、1、2、3、....31这32个数字，来表示不同的datecenterId或workerId
- `12位`，序列号，用来记录同毫秒内产生的不同id。
  - `12位（bit）`可以表示的最大正整数是$2^{12}-1 = 4095$，即可以用0、1、2、3、....4094这4095个数字，来表示同一机器同一时间截（毫秒)内产生的4095个ID序号

由于在Java中64bit的整数是long类型，所以在Java中`SnowFlake`算法生成的id就是long来存储的



然后我们来看一下本项目中是怎么做的



(1) 10位`WorkerID`  即 0-1023



`WorkIdServer.java`用于生成分布式系统中的机器ID(`workerId`):

```java
public class WorkerIdServer {
    private static long workerId = 0;

    static {
        if (workerId == 0) {
            //初始化为1
            workerId = 1;
            //得到服务器机器名称
            String hostName = IpUtils.getPubIpAddr();

            if (redisClientConfig.existsWorkerId(hostName) ) {
                // 如果redis中存在该服务器名称，则直接取得workerId
                workerId = Long.parseLong(redisClientConfig.getWorkerIdForHostName(hostName));
            } else {
                /**
                 *  采用HashMap哈希命中算法来生成一个workerId
                 * 对 为负数 的hash 值取正
                 */
                int h = hostName.hashCode() & 0x7fffffff; // = 0b0111 1111 1111 1111 1111 1111 1111 1111 = Integer.MAX_VALUE
                h = h ^ h >>> 16;
                int id = h % 1024;  //生成的机器ID(workerId)范围是0到1023（包含0和1023）

                workerId = id;
                if (!redisClientConfig.existsWorkerId(hostName)) {
                    long oldWorkerId = workerId;
                    while (redisClientConfig.existsWorkerIdSet(workerId)) {
                        workerId = (workerId + 1L) % 1024;
                        if (workerId == oldWorkerId) {
                            log.error("machine code node cannot be applied, nodes number has reached its maximum value");
                            throw new WorkerIdCantApplyException(String
                                                                 .format("Machine code node cannot be applied, Nodes number has reached its maximum value"));
                        }
                    }
                    redisClientConfig.setWorkerId(hostName, workerId);
                    redisClientConfig.setWorkerIdSet(workerId);
                }
            }

        }
    }
}
```

使用`Redis`来保存每个服务器的`workerId`，并确保每个服务器的`workerId`都是唯一的。**当服务器启动时，会自动根据哈希算法生成一个机器ID(范围为 0-1023)，并保存到`Redis`中，以便后续使用**



(2) `Sid.java`:

该类的目的是为了生成唯一的21位数字字符串ID，其中前6位是表示年月日的字符串，后15位是基于`Snowflake`算法生成的唯一ID(在下面的`IdWorker.java`中讲解如何生成):

```java
  // 配置ID生成器
    public static synchronized void configure() {
        long workerId = WorkerIdServer.getWorkerId(ServerSelector.REDIS_SERVER.getCode()); // 获取工作进程ID
        idWorker = new IdWorker(workerId) {
            // 创建 IdWorker 实例，并重写其中的 getEpoch 方法，用于获取当前时间的毫秒数
            @Override
            public long getEpoch() {
                return TimeUtils.midnightMillis();
            }
        };
    }

 /**
     * 一天最大毫秒86400000，最大占用27比特
     * 27+10+11=48位 最大值281474976710655(15字)，YK0XXHZ827(10字)
     * 6位(YYMMDD)+15位，共21位
     *
     * @return 固定21位数字字符串
     */
    //生成固定21位数字字符串的唯一ID
    public static String next() {
        long id = idWorker.nextId();
        String yyMMdd = new SimpleDateFormat("yyMMdd").format(new Date());
        return yyMMdd + String.format("%015d", id);
    }
```



(3) `IdWorker.java`

自定义的分布式唯一ID生成器（`IdWorker`），通过组合时间戳、工作机器ID和并发序列号来生成唯一的ID。这个自定义ID生成器的特点在于对系统时间回拨的处理,代码中的主要字段和方法解释如下：

1. `epoch`：表示服务器运行时的起始时间戳，默认为 `1288834974657L`(该时间戳 `epoch` 是一个特定的时间点，通常表示系统或服务器开始运行的时间。在这个时间点之前，生成的ID中的时间戳部分将小于 `epoch`，而在这个时间点之后，生成的ID中的时间戳部分将大于或等于 `epoch`)————————**占41bit**
2. `workerIdBits` 和 `sequenceBits`：分别表示工作机器ID和并发序列号所占的位数，默认分别为10位和12位。
3. `maxWorkerId` 和 `sequenceMask`：表示工作机器ID和并发序列号的最大取值，计算方式为将对应的位数全部设置为1
4. `nextId()`：生成下一个唯一ID的方法，使用 `synchronized` 关键字保证线程安全。

- 首先获取当前时间戳 `timestamp`。
- 如果检测到时间回拨（`timestamp < lastMillis`），则需要处理回拨情况。
- 在时间回拨情况下，如果已经恢复到上一次生成ID的时间戳（`timestamp > lastMillis`），则将`isLockBack`标记为 false，表示回拨已经恢复正常。
- 在发生时间回拨时，会将时间戳调整为上一次生成ID的时间戳，并将`isLockBack`标记为 true，表示仍然处于回拨状态。如果是首次发生时间回拨，会记录回拨发生的时间戳到`lockBackTimestamp`。
- 如果当前时间戳与上一次时间戳相同，说明在同一毫秒内，需要更新并发序列号。如果序列号已经达到最大值，则阻塞到下一毫秒再生成ID。
- 如果当前时间戳大于上一次时间戳，表示已经进入下一毫秒，需要将并发序列号重置为0。
- 最后，根据工作机器ID、时间戳和并发序列号的组合，生成最终的唯一ID。

该自定义ID生成器实现了对系统时间回拨的处理，通过标记和调整时间戳，确保生成的唯一ID在发生时间回拨时依然保持唯一性。这样的机制在分布式系统中可以更好地应对时间回拨导致的ID生成问题

```java
public class IdWorker {
    /**
     * 自定义 分布式唯一号 id
     *  1 位 符号位
     * 41 位 时间戳
     * 10 位 工作机器 id
     * 12 位 并发序列号
     *
     *       The distribute unique id is as follows :
     * +---------------++---------------+---------------+-----------------+
     * |     Sign      |     epoch     |    workerId   |     sequence     |
     * |    1 bits     |    41 bits    |    10 bits   |      12 bits      |
     * +---------------++---------------+---------------+-----------------+
     */

    /**
     * 服务器运行时 开始时间戳
     */
    protected long epoch = 1288834974657L;
//    protected long epoch = 1387886498127L; // 2013-12-24 20:01:38.127
    /**
     * 机器 id 所占位数
     */
    protected long workerIdBits = 10L;
    /**
     * 最大机器 id
     * 结果为 1023
     */
    protected long maxWorkerId = -1L ^ (-1L << workerIdBits);
    /**
     * 并发序列在 id 中占的位数
     */
    protected long sequenceBits = 12L;

    /**
     * 机器 id 掩码
     */
    protected long workerIdShift = sequenceBits;
    
    
    
    
    //生成下一个ID
    public synchronized long nextId() {
        long timestamp = millisGen();
        // 上次判定 仍处于 回拨时间内 并且 当前时间系统时间 已经 恢复到 上一次 生成 id 时间戳
        //logger.info("timestamp " + timestamp + " lastMillis " + lastMillis);
        if (isLockBack && timestamp > lastMillis) {
            logger.info(">> Clock dial back to normal <<");
            isFirstLockBack = true;
            isLockBack = false;
        }

        // 发生 时间回拨
        if (timestamp < lastMillis) {
            /**
             * 逻辑 恢复 上一个生成 id 时间戳，当成 时钟回拨 没发生
             * 1. 回拨前 时间戳 大于 上一个 系统时间 生成 id 时间戳
             * 2. 回拨前 时间戳 等于 上一个 系统时间 生成 id 时间戳
              */
            timestamp = lastMillis;
            // 判定 当前仍在 回拨时间内
            isLockBack = true;
            // 首次发生 回拨
            if (isFirstLockBack) {
                logger.warn(">> Clock callback occurs when the ID is generated <<");
                // 记录 当前回拨的时间戳（只会在首次记录）
                lockBackTimestamp = lastMillis;
                // 已经发生回拨了
                isFirstLockBack = false;
            }
        }
        // 当前时间戳 与 上一个 时间戳 在同一毫秒 或 发生时间回拨 逻辑恢复
        if (timestamp == lastMillis) {
            sequence = (sequence + 1) & sequenceMask;
            // 序列号已经最大了，需要阻塞新的时间戳
            // 表示这一毫秒并发量已达上限，新的请求会阻塞到新的时间戳中去
            if (sequence == 0)
                // 发生时间回拨 不能去 阻塞， 因为使用到了当前时间
                if (isLockBack) {
                    // 直接让 上一毫秒 + 1， 产生新的 序列号
                    timestamp = ++lastMillis;
                } else {
                    timestamp = tilNextMillis(lastMillis);
                }
        // 当前毫秒 大于 上一个毫秒，更新 序列号
        } else {
            sequence = 0; // 竞争不激烈时，id 都是偶数
            // 竞争不激烈时 每毫米 刚开始序列号 id 分布均匀
            //sequence = timestamp & 1; // 0 或者 1
        }

        // 前面如果 发生时间回拨 会恢复（逻辑上，系统时间还是没有恢复）发生回拨时的 时间戳

        //正常状态 保存 上一次时间戳
        if (!isLockBack) lastMillis = timestamp;

        long diff = timestamp - getEpoch();

        return (diff << timestampLeftShift) |
                (workerId << workerIdShift) |
                sequence;
    }
    
```





(4)最后来看看`Rpc-Client`的代理是如何使用雪花算法的：

```java
public class RpcClientProxy {

   public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      RpcRequest rpcRequest = new RpcRequest.Builder()
              /**
               * 使用雪花算法 解决分布式 RPC 各节点生成请求号 id 一致性问题
               */
              .requestId(Sid.next())  //生成唯一性ID
              /**
               * 没有处理时间戳一致问题，可通过 synchronized 锁阻塞来获取
               */
              //.requestId(UUID.randomUUID().toString())
              .interfaceName(method.getDeclaringClass().getName())
              .methodName(method.getName())
              .parameters(args)
              .paramTypes(method.getParameterTypes())
              .returnType(method.getReturnType())
              /**这里心跳指定为false，一般由另外其他专门的心跳 handler 来发送
               * 如果发送 并且 hearBeat 为 true，说明触发发送心跳包
               */
              .heartBeat(Boolean.FALSE)
              .reSend(Boolean.FALSE)  //默认设置超时请求位为false
              .build();
      RpcResponse rpcResponse = null;

      if (rpcClient instanceof SocketClient) {
         rpcResponse = revokeSocketClient(rpcRequest, method);
      } else if (rpcClient instanceof NettyClient){
         rpcResponse = revokeNettyClient(rpcRequest, method);
      }

      return rpcResponse == null ? null : rpcResponse.getData();
   }
}

```





### 如何解决请求幂等性问题？



1.首先每个`RpcRequest`必须有一个全局唯一性ID，且第一次发送该`Request`时设置`ReSend`(重发标志位)为`False`

2.这样`RpcServer`在收到该请求并解析包内容后，先去`Redis`中查找有无对应的`Key`为`RequestId`的键值对，如果没有的话——也证明了不是重发包，则调用`RequestHandler`(反射)来执行被调用方法，并将调用结果写入缓存

3.如果由于网络拥塞原因，`RpcClient`没有收到第一次发送`Request`对应的`Response`，在超时(timeout=1000)后

```java
@Reference(retries = 2, timeout = 1000, asyncTime = 3000, giveTime = 1)
private static HelloWorldService service = rpcClientProxy.getProxy(HelloWorldService.class, Client.class);
```

就会发送重发包，`RpcServer`在接收到重发包后Server端直接判断`ReSend=True`,则直接去缓存中查询结果并返回给`RpcClient`

> 有一个问题：重发的时候如何使用上一次生成的唯一性requestId呢?

当需要在重发时使用之前生成的唯一性 `requestId` 时，你可以在 `RpcRequest` 对象中保留这个 ID，以便后续的重发操作使用。在 `RpcRequest` 类中添加一个属性来存储原始的 `requestId`，然后在重发时使用这个属性即可：

```java
class RpcRequest {
    private long originalRequestId;  //重发时才会使用
    private long requestId;
    private String payload;
    private RpcResponse response;
}


class RpcClient {
    private SnowflakeGenerator idGenerator;
    private Map<Long, RpcRequest> requests;
    private ScheduledExecutorService scheduler;
    private long timeout;

    public RpcClient() {
        idGenerator = new SnowflakeGenerator(); // Initialize your Snowflake generator
        requests = new HashMap<>();
        scheduler = Executors.newScheduledThreadPool(1);
        timeout = 10000; // Timeout in milliseconds (10 seconds)
    }

    public void sendRequest(String payload) {
        long requestId = idGenerator.generateId(); //雪花算法生成唯一性ID
        RpcRequest request = new RpcRequest(requestId, payload);
        requests.put(requestId, request);

        // Send the request using your communication mechanism (e.g., network socket)

        // Schedule a task to handle timeouts
        scheduler.schedule(() -> handleTimeout(request), timeout, TimeUnit.MILLISECONDS);
    }

    //超时处理
    private void handleTimeout(RpcRequest request) {
        long requestId = request.getRequestId();

        if (requests.containsKey(requestId)) {
            // The request hasn't received a response within the timeout
            // Handle timeout logic here (e.g., re-send the request)
            resendRequest(request);
        }
    }
    
    //重发处理
    private void resendRequest(RpcRequest request) {
        long originalRequestId = request.getOriginalRequestId();
        // Resend the request using the originalRequestId
        // This could involve re-sending the payload, updating the timestamp, etc.
    }

    public void handleResponse(RpcResponse response) {
        long requestId = response.getRequestId();
        RpcRequest request = requests.get(requestId);

        if (request != null) {
            request.setResponse(response);
            requests.remove(requestId);
        }
    }
}

```











### 缓存服务怎么做的？  :astonished:

`NettyServer`在启动时会自动加载一些预配置：

```java
public class NettyServer extends AbstractRpcServer {

static {
        AbstractRedisConfiguration.getServerConfig();
        /**
         * 其他 预加载选项
         * NettyServer 类中的静态块在类加载时会被执行。
         * 同时，NettyChannelDispatcher 类中的静态方法也会在类加载时被调用
         */
        NettyChannelDispatcher.init();
    }
}
```

可以看出来加载`NettyServer`类的同时也会加载`NettyChannelDispatcher` 类中的静态方法，而`Redis`缓存的设置就在这里，`NettyChannelDispatcher` 类是用于处理客户端发送的 RPC 请求的核心类，我们来着重看一下该类中的一些核心方法：

1、 `static{}`静态代码块：

下面代码主要功能是从配置文件中读取 `Redis` 相关的配置信息，并根据读取的配置设置相关的变量值，同时设置`requestHandler`(`JDKrequestHandler`)来进行后续请求处理：

```java
static {
        // 使用InPutStream流读取properties文件
        String currentWorkPath = System.getProperty("user.dir");
        PropertyResourceBundle configResource = null;
        try (BufferedReader bufferedReader = new BufferedReader(new FileReader(currentWorkPath + "/config/resource.properties"));) {

            configResource = new PropertyResourceBundle(bufferedReader);
            redisServerWay = configResource.getString(PropertiesConstants.REDIS_SERVER_WAY);

            if ("jedis".equals(redisServerWay) || "default".equals(redisServerWay) || StringUtils.isBlank(redisServerWay)) {
                log.info("find redis client way attribute is jedis");
            } else if ("lettuce".equals(redisServerWay)) {
                log.info("find redis client way attribute is lettuce");
                try {
                    redisServerAsync = configResource.getString(PropertiesConstants.REDIS_SERVER_ASYNC);

                    if ("false".equals(redisServerAsync) || "default".equals(redisServerAsync) || StringUtils.isBlank(redisServerAsync)) {
                        log.info("find redis server async attribute is false");
                    } else if ("true".equals(redisServerAsync)) {
                        log.info("find redis server async attribute is lettuce");
                    } else {
                        throw new RuntimeException("redis server async attribute is illegal!");
                    }

                } catch (MissingResourceException redisServerAsyncException) {
                    log.warn("redis server async attribute is missing");
                    log.info("use default redis server default async: false");
                    redisServerAsync = "false";
                }
            } else {
                throw new RuntimeException("redis server async attribute is illegal!");
            }

        } catch (MissingResourceException redisServerWayException) {
            log.warn("redis client way attribute is missing");
            log.info("use default redis client default way: jedis");
            redisServerWay = "jedis";
        } catch (IOException ioException) {
            log.info("not found resource from resource path: {}", currentWorkPath + "/config/resource.properties");
            try {
                ResourceBundle resource = ResourceBundle.getBundle("resource");
                redisServerWay = resource.getString(PropertiesConstants.REDIS_SERVER_WAY);
                if ("jedis".equals(redisServerWay) || "default".equals(redisServerWay) || StringUtils.isBlank(redisServerWay)) {
                    log.info("find redis server way attribute is jedis");
                } else if ("lettuce".equals(redisServerWay)) {
                    log.info("find redis server way attribute is lettuce");
                    try {
                        redisServerAsync = resource.getString(PropertiesConstants.REDIS_SERVER_ASYNC);

                        if ("false".equals(redisServerAsync) || "default".equals(redisServerAsync) || StringUtils.isBlank(redisServerAsync)) {
                            log.info("find redis server async attribute is false");
                        } else if ("true".equals(redisServerAsync)) {
                            log.info("find redis server async attribute is lettuce");
                        } else {
                            throw new RuntimeException("redis server async attribute is illegal!");
                        }

                    } catch (MissingResourceException redisServerAsyncException) {
                        log.warn("redis server async attribute is missing");
                        log.info("use default redis server default async: false");
                        redisServerAsync = "false";
                    }
                } else {
                    throw new RuntimeException("redis client way attribute is illegal!");
                }

            } catch (MissingResourceException resourceException) {
                log.info("not found resource from resource path: {}", "resource.properties");
                log.info("use default redis server way: jedis");
                redisServerWay = "jedis";
            }
            log.info("read resource from resource path: {}", "resource.properties");

        }
        requestHandler = new JdkRequestHandler();
    }
```



2、`disptach()`  

> 
>
> 一般使用`Redis`来减少`RpcServer`处理的思路如下：
>
> 1. 客户端发送请求：当客户端发送一个需要保证幂等性的请求时，先生成一个唯一的请求ID，并将请求ID和请求参数一起发送到服务器端。
> 2. 服务器端处理请求：服务器端在接收到请求后，首先根据请求ID去查询Redis缓存，检查是否存在该请求ID的处理结果。
> 3. Redis缓存处理结果：如果Redis缓存中存在该请求ID的处理结果，说明该请求之前已经被处理过，并且结果已经被缓存了。服务器可以直接从缓存中取得结果并返回给客户端，避免重复处理
>
> 
>
> 但是以上思路存在一个问题，就是服务端在接受到`Request`时首先要查询缓存，我们可以抛弃`Redis`检验重发包的思路，转为在`Request`上加一个标志位`ReSend`(默认为`false`)，每次发送请求后则设置该标志位为`true`,这样服务器校验该标志位如果为`true`后，就直接查询Redis获取Result结果并返回就可以了

```java

    public static void dispatch(ChannelHandlerContext ctx, RpcRequest msg) {
        operationExecutorService.submit(() -> {
            log.info("server has received request package: {}", msg);
            // 到了这一步，如果请求包在上一次已经被 服务器成功执行，接下来要做幂等性处理，也就是客户端设置超时重试处理
            /**
             * 改良 2023.1.9
             * 使用 Redis 实现分布式缓存
             * 改良 2023.3.19
             * 抛弃 Redis 检验 重发包
             * 采用 客户端  请求包 标志位，减少一次 判断 Redis IO 操作
             */
            Object result = null;
            AbstractRedisConfiguration redisServerConfig = AbstractRedisConfiguration.getServerConfig();
            //if (!redisServerConfig.existsRetryResult(msg.getRequestId())) {
            if (!msg.getReSend()) {
                log.info("[requestId: {}, reSend: {}] does not exist, store the result in the distributed cache", msg.getRequestId(), msg.getReSend());
                result = requestHandler.handler(msg);
                if (result != null) {
                    writeResultToChannel(ctx, msg, result);

                    String redisServerWay = AbstractRedisConfiguration.getRedisServerWay();
                    if ("jedis".equals(redisServerWay))
                        redisServerConfig.setRetryRequestResultByString(msg.getRequestId(), JsonUtils.objectToJson(result));
                    else {
                        String redisServerAsync = AbstractRedisConfiguration.getRedisServerAsync();
                        if ("true".equals(redisServerAsync)) {
                            redisServerConfig.asyncSetRetryRequestResult(msg.getRequestId(), serializer.serialize(result));
                        } else {
                            redisServerConfig.setRetryRequestResultByBytes(msg.getRequestId(), serializer.serialize(result));
                        }
                    }
                } else {
                    writeResultToChannel(ctx, msg, null);
                    String redisServerAsync = AbstractRedisConfiguration.getRedisServerAsync();
                    if ("true".equals(redisServerAsync)) {
                        redisServerConfig.asyncSetRetryRequestResult(msg.getRequestId(), null);
                    } else {
                        redisServerConfig.setRetryRequestResultByBytes(msg.getRequestId(), null);
                    }
                }
            } else {
                String redisServerWay = AbstractRedisConfiguration.getRedisServerWay();
                if ("jedis".equals(redisServerWay)) {
                    result = redisServerConfig.getResultForRetryRequestId2String(msg.getRequestId());
                    if (result != null) {
                        result = JsonUtils.jsonToPojo((String) result,  msg.getReturnType());
                    }
                } else {
                    result = redisServerConfig.getResultForRetryRequestId2Bytes(msg.getRequestId());
                    if (result != null) {
                        result = serializer.deserialize((byte[]) result, msg.getReturnType());
                    }
                }
                log.debug("Previous results:{} ", result);
                log.info(" >>> Capture reSend package [requestId: {} [method: {}, returnType: {}] <<< ", msg.getRequestId(), msg.getMethodName(), msg.getReturnType());
                writeResultToChannel(ctx, msg, result);
            }

        });
    }

```



1. 接收到 RPC 请求：`dispatch` 方法被调用时，会传入 `ChannelHandlerContext` 和 `RpcRequest` 对象 `msg`，其中 `ChannelHandlerContext` 表示当前通道的上下文，`RpcRequest` 包含了客户端发送的 `RPC` 请求信息
2. 提交任务到线程池：使用 `operationExecutorService` 线程池，将请求处理任务提交，以便在后台线程中执行请求处理逻辑，避免阻塞当前线程
3. 判断请求是否需要重发：通过检查 `msg` 对象的 `getReSend()` 方法来判断请求是否需要重发，这是一个客户端设置的标志位。如果不需要重发，则表示是首次请求，并且没有在分布式缓存中找到缓存结果
4. 处理首次请求：
   - 调用 `requestHandler.handler(msg)` 方法处理请求，得到处理结果 `result`
   - 如果结果 `result` 不为空，则通过 `writeResultToChannel` 方法将结果写入通道 `ctx` 并返回给客户端，**同时根据配置将结果保存到分布式缓存中**
5. 处理重发请求：
   - 如果请求需要重发，则根据 `msg.getRequestId()` 从分布式缓存中获取之前的结果，并将结果 `result` 进行反序列化，并使用 `writeResultToChannel` 方法将结果写入通道 `ctx` 并返回给客户端







### Netty 异步非阻塞是怎么做的？















### 客户端心跳机制如何实现？

心跳机制在 `RPC` 上应用的很广泛，本项目对心跳机制的实现很简单，应对措施是服务端强制断开连接，当然有些 `RPC` 框架实现了服务端去主动尝试重连。

本项目中心跳机制的核心是通过客户端监听`IdleStateEvent`中的写事件，重写`userEventTriggered()`

方法，在该方法中捕获 `IdleState.WRITER_IDLE`事件（未在指定时间内向服务器发送数据），然后向 `Server`端发送一个心跳包

- 原理

对于心跳机制的应用，其实是使用了 `Netty` 框架中的一个 `handler` 处理器，通过该处理器，去定时发送心跳包，让服务端知道该客户端保持活性状态。

- 实现

利用了 `Netty` 框架中的 `IdleStateEvent` 事件监听器，重写`userEventTriggered()` 方法，在服务端监听读操作，读取客户端的写操作，在客户端监听写操作，监听本身是否还在活动，即有没有向服务端发送请求。

如果客户端没有主动断开与服务端的连接，而继续保持连接着，那么**客户端的写操作超时后，也就是客户端的监听器监听到客户端没有的规定时间内做出写操作事件，那么这时客户端该处理器主动发送心跳包给服务端，保证客户端让服务端确保自己保持着活性。**





在 `Netty` 中，`IdleStateEvent` 是一个特殊的事件，用于表示连接的空闲状态。当一个连接在一段时间内没有进行读操作、写操作或读写操作时，就会触发 `IdleStateEvent` 事件。**为了在连接空闲时执行特定的逻辑，你可以重写 `userEventTriggered()` 方法**，它是 `ChannelInboundHandler` 接口中的一个方法



客户端监听写空闲事件(`WRITER_IDLE`)：

```java
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {  //如果发生了
            空闲事件
            IdleState state = ((IdleStateEvent) evt).state();
            // 客户端没有发送数据了(发生了写空闲)  ——————从而 发送心跳包 给 服务端
            if (state == IdleState.WRITER_IDLE) {
                log.debug("Send heartbeat packets to server[{}]", ctx.channel().remoteAddress());
                NettyChannelProvider.get((InetSocketAddress) ctx.channel().remoteAddress(), CommonSerializer.getByCode(CommonSerializer.HESSIAN_SERIALIZER));
                RpcRequest rpcRequest = new RpcRequest();
                rpcRequest.setHeartBeat(Boolean.TRUE);
                ctx.writeAndFlush(rpcRequest).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
```



服务端监听读空闲事件(`READER_IDLE`):

```java
 public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleState state = ((IdleStateEvent) evt).state();
            if (state == IdleState.READER_IDLE) {
                log.info("Heartbeat packets have not been received for a long time");
                ctx.channel().close();
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
```



同时心跳机制还保证了TCP的长连接：

Netty 中提供了 `IdleStateHandler` 类专门用于处理心跳，所以是长连接，如果没有设置这个，默认一般是短连接







### 动态代理 代理的是什么？

动态代理允许你创建一个代理对象，该代理对象可以在调用方法时，将方法调用转发到远程的服务端，然后将结果返回给调用方。以下是关于动态代理在自定义RPC框架中的相关概念：

1. **代理对象**：

**代理对象是在客户端创建的一个对象，它具有与实际服务对象相同的接口**。当客户端调用代理对象的方法时，**代理对象会将方法调用转发给远程的服务端，然后接收并返回执行结果**。代理对象隐藏了底层的网络通信细节，**使得客户端可以像调用本地方法一样调用远程的服务。**



2. **动态代理**：

动态代理是在运行时创建代理对象的技术。Java 提供了 `java.lang.reflect` 包来支持动态代理。通过使用动态代理，你可以在运行时创建一个实现了指定接口的代理类，该代理类的方法调用会被转发到指定的调用处理器（InvocationHandler）



3. **代理的什么**：

在自定义RPC框架中，代理的是远程的服务接口。当你在客户端代码中定义一个服务接口，并通过动态代理创建了代理对象时，这个代理对象实际上代理了远程的服务接口。通过代理对象，你可以在客户端调用远程服务的方法，而这些方法调用会被代理对象转发到服务端执行。



例如`Rpc-Client`想要远程调用`HelloService`方法时：

```java
SocketClient client = new SocketClient(CommonSerializer.KRYO_SERIALIZER);
RpcClientProxy proxy = new RpcClientProxy(client);
HelloService helloService = proxy.getProxy(HelloService.class);
HelloObject object = new HelloObject(12, "This is a message"); 

//生成的代理对象helloService在调用hello()时，实际上会调用invocationHandler中的invoke()
String res = helloService.hello(object);
System.out.println(res);
```

- 创建一个SocketClient对象，使用CommonSerializer.KRYO_SERIALIZER作为序列化器参数。
- 创建一个RpcClientProxy对象，将上一步创建的SocketClient作为参数传入。RpcClientProxy用于创建远程服务接口的代理对象。
- **通过RpcClientProxy的getProxy()方法获取HelloService的代理对象。这个代理对象实现了HelloService接口，并能够将接口方法调用转发给RPC服务端**
- 创建一个HelloObject对象，传入参数12和"This is a message"。
- 调用HelloService的hello()方法，传入上一步创建的HelloObject对象，并将返回结果赋给res变量。
- 输出res变量，即远程方法调用的结果



然后我们再来加深一下对JDK动态代理的理解:

先来看看JDK动态代理中生成代理对象的方法：

```java
 public <T> T getProxy(Class<T> clazz) { // 生成代理对象
        return (T) Proxy.newProxyInstance(clazz.getClassLoader(), new Class<?>[]{clazz}, this);
    }
```

上例中我们想要远程调用`HelloService`接口实现类的方法，所以这里的`clazz`指的就是`HelloService`接口，`Proxy.newProxyInstance()`所自动生成的代理对象实现了指定接口的方法，并且方法的调用会被转发到实现代理逻辑的 `InvocationHandler` 上



但是有一点要注意：

在使用动态代理创建代理对象时，`clazz` 参数通常应该是服务接口本身，而不是它的实现类。

动态代理的目的是为了在客户端调用时，将方法调用转发到远程服务端。在这种情况下，客户端不需要关心具体的实现类，只需要知道服务接口的方法签名即可。代理对象的作用是隐藏底层的远程通信细节，让客户端可以通过接口方法来访问远程服务。

因此，当调用 `getProxy(Class<T> clazz)` 方法时，`clazz` 参数应该是服务接口的 `Class` 对象，而不是它的实现类。这样可以确保代理对象实现了服务接口的方法，并将方法调用转发到远程服务端。



然后我们再来看一下生成的代理对象实际调用的`invoke()`方法，生成的代理对象替`RpcClient`来发送远程调用请求：

```java

    public Object invoke(Object proxy, Method method, Object[] args) {
        logger.info("调用方法: {}#{}", method.getDeclaringClass().getName(), method.getName());
        RpcRequest rpcRequest = new RpcRequest(UUID.randomUUID().toString(), method.getDeclaringClass().getName(),
                method.getName(), args, method.getParameterTypes(), false);
        RpcResponse rpcResponse = null;
        if (client instanceof NettyClient) {
            try {
                //异步
                CompletableFuture<RpcResponse> completableFuture = (CompletableFuture<RpcResponse>) client.sendRequest(rpcRequest);
                rpcResponse = completableFuture.get();
            } catch (Exception e) {
                logger.error("方法调用请求发送失败", e);
                return null;
            }
        }
        if (client instanceof SocketClient) {
            //同步
            rpcResponse = (RpcResponse) client.sendRequest(rpcRequest);
        }
        RpcMessageChecker.check(rpcRequest, rpcResponse);
        return rpcResponse.getData();
```







### 超时重传机制是怎么实现的？

```java

   private RpcResponse revokeNettyClient(RpcRequest rpcRequest, Method method)  throws Throwable {
      RpcResponse rpcResponse = null;
      if (pareClazz == null) {
         CompletableFuture<RpcResponse> completableFuture = (CompletableFuture<RpcResponse>) rpcClient.sendRequest(rpcRequest);
         rpcResponse = completableFuture.get();

         RpcMessageChecker.checkAndThrow(rpcRequest, rpcResponse);
         return rpcResponse;
      }

      /**
       * 服务组名、重试机制实现
       */
      long timeout = 0L;
      long asyncTime = 0L;
      int retries = 0;
      int giveTime = 0;
      boolean useRetry = false;

      Field[] fields = pareClazz.getDeclaredFields();
      for (Field field : fields) {
         if (field.isAnnotationPresent(Reference.class) &&
                 method.getDeclaringClass().getName().equals(field.getType().getName())) {
            retries = field.getAnnotation(Reference.class).retries();
            giveTime =field.getAnnotation(Reference.class).giveTime();
            timeout = field.getAnnotation(Reference.class).timeout();
            asyncTime =field.getAnnotation(Reference.class).asyncTime();
            useRetry = true;
            String name = field.getAnnotation(Reference.class).name();
            String group = field.getAnnotation(Reference.class).group();
            if (!"".equals(name)) {
               rpcRequest.setInterfaceName(name);
            }
            if (!"".equals(group)) {
               rpcRequest.setGroup(group);
            }
            break;
         }
      }


      /**
       * 1、识别不到 @Reference 注解执行
       * 2、识别到 @Reference 且 asyncTime 缺省 或 asyncTime <= 0
       */
      if (!useRetry || asyncTime <= 0 || giveTime <= 0) {
         log.debug("discover @Reference or asyncTime <= 0, will use blocking mode");
         long startTime = System.currentTimeMillis();
         log.info("start calling remote service [requestId: {}, serviceMethod: {}]", rpcRequest.getRequestId(), rpcRequest.getMethodName());
         CompletableFuture<RpcResponse> completableFuture = (CompletableFuture<RpcResponse>) rpcClient.sendRequest(rpcRequest);
         rpcResponse = completableFuture.get();
         long endTime = System.currentTimeMillis();

         log.info("handling the task takes time {} ms", endTime - startTime);

         RpcMessageChecker.checkAndThrow(rpcRequest, rpcResponse);

      } else {
         /**
          * 识别到 @Reference 注解 且 asyncTime > 0 执行
          */
         log.debug("discover @Reference and asyncTime > 0, will use blocking mode");
         if (timeout >= asyncTime) {
            log.error("asyncTime [ {} ] should be greater than timeout [ {} ]", asyncTime, timeout);
            throw new AsyncTimeUnreasonableException("Asynchronous time is unreasonable, it should greater than timeout");
         }
         long handleTime = 0;
         boolean checkPass = false;
         for (int i = 0; i < retries; i++) {
            if (handleTime >= timeout) {
               // 超时重试
               TimeUnit.SECONDS.sleep(giveTime);
               rpcRequest.setReSend(Boolean.TRUE);
               log.warn("call service timeout and retry to call [ rms: {}, tms: {} ]", handleTime, timeout);
            }
            long startTime = System.currentTimeMillis();
            log.info("start calling remote service [requestId: {}, serviceMethod: {}]", rpcRequest.getRequestId(), rpcRequest.getMethodName());
            CompletableFuture<RpcResponse> completableFuture = (CompletableFuture<RpcResponse>) rpcClient.sendRequest(rpcRequest);
            try {
               rpcResponse = completableFuture.get(asyncTime, TimeUnit.MILLISECONDS);
            } catch (TimeoutException e) {
               // 忽视 超时引发的异常，自行处理，防止程序中断
               log.warn("recommend that asyncTime [ {} ] should be greater than current task runeTime [ {} ]", asyncTime, System.currentTimeMillis() - startTime);
               continue;
            }

            long endTime = System.currentTimeMillis();
            handleTime = endTime - startTime;

            if (handleTime < timeout) {
               // 没有超时不用再重试
               // 进一步校验包
               checkPass = RpcMessageChecker.check(rpcRequest, rpcResponse);
               if (checkPass) {
                  log.info("client call success [ rms: {}, tms: {} ]", handleTime, timeout);
                  return rpcResponse;
               }
               // 包被 劫持触发 超时重发机制 保护重发
            }
         }
         log.info("client call failed  [ rms: {}, tms: {} ]", handleTime, timeout);
         // 客户端在这里无法探知是否成功收到服务器响应，只能确定该请求包 客户端已经抛弃了
         unprocessedRequests.remove(rpcRequest.getRequestId());
         throw new RetryTimeoutException("The retry call timeout exceeds the threshold, the channel is closed, the thread is interrupted, and an exception is forced to be thrown!");
      }

      return rpcResponse;
   }
```



1. 首先，代码检查了是否识别到了 `@Reference` 注解，并获取了相关的配置参数，比如 `retries`（重试次数）、`timeout`（**总超时时间**）、`asyncTime`（异步超时时间）、`giveTime`（重试间隔时间）等等。
2. 如果没有识别到 `@Reference` 注解，或者 `asyncTime` 小于等于0，或者 `giveTime` 小于等于0，就说明不需要使用超时重传机制，那么就会执行阻塞模式的调用（blocking mode）。此时会记录调用的开始时间，发送请求，等待响应返回，然后检查响应是否合法。
3. 如果识别到了 `@Reference` 注解且 `asyncTime` 大于0，那么代码就进入超时重传的逻辑。
   - 首先，检查 `timeout` 是否大于等于 `asyncTime`，如果不满足，抛出异常。
   - 进入重试循环，最多进行 `retries` 次重试。每次循环都会先检查 `handleTime` 是否已经超过了 `timeout`，如果是，则认为调用超时，等待一段时间 `giveTime` 后继续重试，同时标记 `rpcRequest` 为需要重新发送。
   - 在每次循环内，记录调用开始时间，发送请求，使用 `CompletableFuture.get()` 方法获取响应，但此时设置了超时时间为 `asyncTime` 毫秒。如果在 `asyncTime` 时间内没有获取到响应，会捕获 `TimeoutException`，表示调用尚未完成，但不会中断线程，继续进行下一次循环。
   - 在获取到响应后，计算调用时间 `handleTime`，如果 `handleTime` 小于 `timeout`，则进行进一步的校验。如果校验通过，返回响应；否则，进入下一次循环。
4. 如果重试循环结束后，仍然没有成功获取合法响应，说明调用失败。此时会从 `unprocessedRequests` 中移除当前请求，并抛出 `RetryTimeoutException` 异常，表示超时重传次数已经超过阈值。





在测试`Rpc-Client`发起调用时的代码如下：

```java
@Reference(name = "helloService", group = "1.0.0",timeout = 1000, asyncTime = 3000)
private static HelloWorldService service = rpcClientProxy.getProxy(HelloWorldService.class, Client.class);
```

















### Nacos相关问题



**1、为什么选用Nacos作为RPC服务的注册中心？**(从`Nacos`的优点答)

(1) Nacos 集群状态下**同时支持CP+AP**：节点之间会互相同步服务实例，用来保证服务信息的一致性

CP：（leader选举-raft算法）

AP：（distro协议，类Gossip协议）





(2) **服务健康检查**：Nacos Server会开启一个定时任务用来检查注册服务实例的健康情况，对于超过15s没有收到客户端心跳的实例会将它的healthy属性置为false(客户端服务发现时不会发现)，如果某个实例超过30秒没有收到心跳，直接剔除该实例(被剔除的实例如果恢复发送心跳则会重新注册)，即底层原理是通过`ScheduledExecutorService`定时向server发送数据包，然后启动一个线程检测服务端的返回，如果在指定时间没有返回则认为服务端出了问题，服务端也会根据发来的心跳包不断更新服务的状态。



`ScheduledExecutorService` 是 Java 中用于创建和管理按计划执行任务的接口。它允许你在特定的时间间隔或者延迟之后执行任务。这对于需要定期执行一些操作或者延迟执行某些操作非常有用。

可以创建一个 `ScheduledExecutorService` 实例：

```java
ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
```

接下来，你可以使用 `schedule` 方法来安排一个任务在特定的延迟之后执行：

```java
Runnable task = () -> {
    // 这里放置你要执行的任务逻辑
    System.out.println("Task executed at: " + System.currentTimeMillis());
};

long delay = 5; // 延迟 5 秒后执行任务
executorService.schedule(task, delay, TimeUnit.SECONDS);
```



或者也可以使用 `scheduleAtFixedRate` 方法来按固定的时间间隔重复执行任务(`Nacos`使用的策略)：

```java
long initialDelay = 2; // 初始延迟 2 秒
long period = 10; // 每隔 10 秒重复执行任务
executorService.scheduleAtFixedRate(task, initialDelay, period, TimeUnit.SECONDS);
```

当你不再需要这个 `ScheduledExecutorService` 时，记得要关闭它以释放资源：

```java
executorService.shutdown();
```





(3) **服务发现：**服务消费者（Nacos Client）在调用服务提供者的服务时，会发送一个REST请求给Nacos Server，获取上面注册的服务清单，并且**缓存**在Nacos Client本地，同时会在Nacos Client本地开启一个定时任务定时拉取服务端最新的注册表信息更新到本地缓存









**2、`Nacos`如何实现服务健康检测的？**







3、`Nacos`如何实现**服务注册**的？



https://developer.aliyun.com/article/1058265





### 序列化/反序列化协议对比



***JSON\***

- JSON 进行序列化的额外空间开销比较大，对于大数据量服务这意味着需要巨大的内存和磁盘开销； 
- JSON 没有类型，但像 Java 这种强类型语言，需要通过反射统一解决，所以性能不会太好（比如反序列化时先反序列化为String类，要自己通过反射还原）





***Kryo\***：

- 使用变长的int和long保证这种基本数据类型序列化后尽量小 
- 需要传入完整类名或者利用 register() 提前将类注册到Kryo上，**其类与一个int型的ID相关联，序列中只存放这个ID，因此序列体积就更小** 
- **不是线程安全的**，要通过`ThreadLocal`或者创建Kryo线程池来保证线程安全 
- 不需要实现Serializable接口 
- 字段增、减，序列化和反序列化时无法兼容 
- 必须拥有无参构造函数 



***Hessian\***：

- **使用固定长度存储int和long** 
- 将所有类字段信息都放入序列化字节数组中，直接利用字节数组进行反序列化，不需要其他参与，因为存的东西多处理速度就会慢点。 
- 把复杂对象的所有属性存储在一个Map中进行序列化。所以在父类、子类存在同名成员变量的情况下，Hessian序列化时，先序列化子类，然后序列化父类，因此反序列化结果会导致子类同名成员变量被父类的值覆盖 
- 需要实现Serializable接口 
- 兼容字段增、减，序列化和反序列化 
- 必须拥有无参构造函数 
- Java 里面一些常见对象的类型不支持，比如：
  - Linked 系列，LinkedHashMap、LinkedHashSet 等； 
  - Locale 类，可以通过扩展 ContextSerializerFactory 类修复； 
  - Byte/Short 反序列化的时候变成 Integer。



***Protobuf:\***

- 序列化后体积相比 JSON、Hessian 小很多
- IDL 能清晰地描述语义，所以足以帮助并保证应用程序之间的类型不会丢失，无需类似XML 解析器；
- 序列化反序列化速度很快，不需要通过反射获取类型；
- 打包生成二进制流
- 预编译过程不是必须的















# 微服务项目



# 



## Nacos 注册/配置中心的使用



1、作为服务的注册中心

启动四个微服务后，在浏览器输入 http://192.168.101.65:8848/nacos/，即可看到当前正在活动的服务实例：

![](https://cdn.jsdelivr.net/gh/amonstercat/blog-images/nacos1.png)

注意:

- namespace:命名空间表示不同的开发环境下(例如 开发环境、测试环境、生产环境)
- group:组则表示不同的项目
- 默认在本机只为每个服务启动了一个实例，其实可以在单机上模拟多实例启动(让多实例运行在不同端口)，然后通过负载均衡将请求分配至不同实例





2、作为服务的配置中心

然后我们可以通过`Nacos Config`将一些公用配置、私有配置放在Nacos的配置中心里，实现配置的集中管理以及热更新部署： 例如下面是将content-service 生产环境下的yaml放到配置中心内：

![](https://cdn.jsdelivr.net/gh/amonstercat/blog-images/nacos-config.png)



需要注意的是，和服务注册相关的配置不能由nacos-config集中管理，因为假如要管content-api微服务的相关配置项，首先要先将 `cotent-api` 服务注册到nacos里呀，因此下列**和服务注册的相关配置还是要在项目里的`application.yaml`里进行配置**：

```yaml
#微服务配置
spring:
  application:
    name: content-api
  cloud:
    nacos:
      server-addr: 192.168.101.65:8848
      discovery:
        namespace: dev148
        group: xuecheng-plus-project
      config:              #config文件存放的目录
        namespace: dev148
        group: xuecheng-plus-project
        file-extension: yaml
        refresh-enabled: true
```





同时Nacos的配置中心支持热更新，比如我想修改某个微服务content-api的数据源，直接在`Nacos`的`DashBoard`里修改`content-service`对应的yaml文件内容即可，无需重启微服务，`content-service-dev.yaml`在config中的配置内容如下：

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://192.168.101.65:3306/xc148_content?serverTimezone=UTC&userUnicode=true&useSSL=false&
    username: root
    password: mysql
    
xxl:
  job:
    admin: 
      addresses: http://192.168.101.65:8088/xxl-job-admin
    executor:
      appname: coursepublish-job
      address: 
      ip: 
      port: 8999
      logpath: /data/applogs/xxl-job/jobhandler
      logretentiondays: 30
    accessToken: default_token
test_config:
 a: 2a
 b: 2b
 c: 2c    
```





## GateWay 网关的使用



项目采用`Spring Cloud Gateway`作为网关，网关在请求路由时需要知道每个微服务实例的地址，项目使用`Nacos`作用服务发现中心和配置中心，整体的架构图如下：

![](https://cdn.jsdelivr.net/gh/amonstercat/blog-images/clip_image001.png)

流程如下：

1、微服务启动，将各自服务的所有实例注册到 Nacos ， Nacos记录了各微服务实例的地址。

2、网关从Nacos读取服务列表，包括服务名称、服务地址等。

3、后续请求会先到达网关，网关将请求路由到具体的微服务。

要使用网关首先搭建Nacos，Nacos有两个作用：

1、服务发现中心

微服务将自身注册至Nacos，网关从Nacos获取微服务列表。

2、配置中心

微服务众多，它们的配置信息也非常复杂，为了提供系统的可维护性，微服务的配置信息统一在Nacos配置。

在搭建Nacos服务发现中心之前需要搞清楚两个概念：`namespace`和`group`

`namespace`：用于区分环境、比如：开发环境、测试环境、生产环境。

`group`：用于区分项目，比如：xuecheng-plus项目、xuecheng2.0项目



### 路由选择

网关作为项目请求的入口，最主要的作用就是实现路由，观察其配置文件：

```yaml
server:
  port: 63010 # 网关端口
spring:
  cloud:
    gateway:
#      filter:
#        strip-prefix:
#          enabled: true
      routes: # 网关路由配置
        - id: content-api #为这个路由规则自定义一个唯一的标识符id
          # uri: http://127.0.0.1:8081 # 路由的目标地址 http就是固定地址
          uri: lb://content-api # 路由的目标地址 lb就是负载均衡，后面跟服务名称
          predicates: # 路由断言，也就是判断请求是否符合路由规则的条件
            - Path=/content/** #断言要求请求路径必须以 /content/ 开头才会匹配路由
#          filters:
#            - StripPrefix=1
        - id: system-api
          # uri: http://127.0.0.1:8081
          uri: lb://system-api
          predicates:
            - Path=/system/**
#          filters:
#            - StripPrefix=1
        - id: media-api
          # uri: http://127.0.0.1:8081
          uri: lb://media-api
          predicates:
            - Path=/media/**
#          filters:
#            - StripPrefix=1
```

我们来看看网关对`content-api`服务的路由处理：

1. `uri: lb://content-api`：指定该路由的目标地址。`lb://` 前缀表示负载均衡，**后面跟着服务名称（在注册中心中注册的服务名）**。在这个配置中，网关将传入的请求转发到名为 `content-api` 的服务上，通过负载均衡进行分发。
2. `predicates` 部分：定义路由的断言条件，即判断请求是否满足路由规则的条件;`                  - Path=/content/**`：这个断言要求请求的路径必须以 `/content/` 开头才会匹配这个路由
3. 这里的 lb://***  ： 这里的负载均衡实现使用了 Spring Cloud Gateway 集成的负载均衡功能，并且通常是通过服务注册中心 Nacos 本地集成来实现的



相应地，再来看一下`content-api-dev.yaml`中的内容：

```yaml
server:
  servlet:
    context-path: /content  # 应用的根路径，意味着所有的请求都需要以 /content 开头
  port: 63040
```

例如，如果你向该应用发送一个请求，如 `http://localhost:63040/content/some/path`，由于上下文路径被设置为 `/content`，该请求会被映射到应用的根路径 `/content`，然后由应用处理 `/some/path` 部分的逻辑。这种方式可以帮助将不同的应用隔离在不同的上下文路径下，以防止路径冲突





最后我们来总结一下网关如何控制访问content-api微服务的？

1、 启动 `content-api` 服务：

 通过配置 `content-api.yaml` 中的端口 `63040` 和上下文路径 `/content` ，该服务将会在 http://localhost:63040/content 上监听请求

2、启动 Spring Cloud Gateway 服务： 

通过配置 `gateway.yaml` 中的端口 `63010` 和路由规则，Spring Cloud Gateway 将会在 `http://localhost:63010` 上监听请求，并将满足条件的请求转发给 `content-api` 服务。

3、发送请求到 Gateway： 

要访问 `content-api` 服务，需要向 Spring Cloud Gateway 发送请求，然后 Gateway 会将请求转发给 `content-api` 服务。假设希望访问 `content-api` 中的一个接口，如 `/some-endpoint`，需要发送请求到 Spring Cloud Gateway 的地址，路径以 `/content/some-endpoint` 开头。例如：`http://localhost:63010/content/some-endpoint`

这样，Spring Cloud Gateway 会将请求转发给 `content-api` 服务，`content-api` 服务会处理 `/some-endpoint` 部分的逻辑。 





当然也可以绕过网关直接通过63040端口访问content-api的微服务，只是这样网关就起不到路由过滤的作用了



我们来看一下经过网关测试接口和不经过网关直接访问的两种方式：

直接访问 content-api:

```http
GET  localhost:63040/content/course-category/tree-nodes
```



经网关路由访问：

```http
GET  localhost:63040/content/course-category/tree-nodes
```





最终都可以访问到结果：

![](https://cdn.jsdelivr.net/gh/amonstercat/blog-images/gateway2.png)









### 白名单放行















## MinIO 分布式文件存储

在大数据领域，通常的设计理念都是无中心和分布式。Minio分布式模式可以帮助你搭建一个高可用的对象存储服务，你可以使用这些存储设备，而不用考虑其真实物理位置。

它将分布在不同服务器上的多块硬盘组成一个对象存储服务。由于硬盘分布在不同的节点上，分布式Minio避免了单点故障

Minio使用**纠删码**技术来保护数据，它是一种恢复丢失和损坏数据的数学算法，它将数据分块冗余的分散存储在各各节点的磁盘上，所有的可用磁盘组成一个集合，上图由8块硬盘组成一个集合，**当上传一个文件时会通过纠删码算法计算对文件进行分块存储，除了将文件本身分成4个数据块，还会生成4个校验块，数据块和校验块会分散的存储在这8块硬盘上**

使用纠删码的好处是即便丢失一半数量（N/2）的硬盘，仍然可以恢复数据。 比如上边集合中有4个以内的硬盘损害仍可保证数据恢复，不影响上传和下载，如果多于一半的硬盘坏了则无法恢复



### 上传图片

课程图片是宣传课程非常重要的信息，在新增课程界面上传课程图片，也可以修改课程图片

上传课程图片总体流程如下：

1、前端进入上传图片界面

2、上传图片，请求媒资管理服务。

3、媒资管理服务将图片文件存储在MinIO。

4、媒资管理记录文件信息到数据库。

5、前端请求内容管理服务保存课程信息，在内容管理数据库保存图片地址。









## Sentinel实现服务限流与降级









## SQL查询速度慢 如何优化？





## 如何保证Mysql读写分离?



> 1、主从复制原理是什么？

<img src="https://cdn.jsdelivr.net/gh/amonstercat/blog-images/mysql%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6.png" style="zoom:67%;" />

(1) **半同步复制**： 要求主库完成BinLog日志写入后立即将数据同步到从库，解决从库数据丢失问题

(2) **并行复制**：  要求从库开启多线程并行执行收取到的日志，保证用户能够尽快读取到请求，解决延时问题



> 2、什么是**主从同步延时**？

<img src="https://cdn.jsdelivr.net/gh/amonstercat/blog-images/%E4%B8%BB%E4%BB%8E%E5%BB%B6%E6%97%B6.png" style="zoom:67%;" />



**项目场景：用户购买了一节课后，支付完成立即跳转回主页，此时主页应该显示刚刚下单成功的课程信息，但是由于读请求被`send`到从库(或者`redis`)，这样就导致用户无法立即看到改变**













> 3、那么如何**解决读写分离后，先写后查可能带来的数据不一致**问题？



(1)方案一  当前线程每次插入完毕后让其休眠，保证从库完成日志重放：

<img src="https://cdn.jsdelivr.net/gh/amonstercat/blog-images/%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E5%BB%B6%E8%BF%9F%E4%B8%80%E8%87%B4%E6%80%A7%201.png" style="zoom: 50%;" />



(2)方案二  利用框架特性解决：

<img src="https://cdn.jsdelivr.net/gh/amonstercat/blog-images/%E5%BB%B6%E8%BF%9F%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%882.png" style="zoom: 50%;" />







后面贴代码















## 如何实现数据库缓存一致性



> 用户A**更新数据时，只更新数据库不更新缓存，而是删除缓存中的数据。然后用户B读取数据时，发现缓存中没了数据之后，再从数据库中读取数据，更新到缓存中**。这个策略是有名字的：**`Cache Aside`** ————旁路缓存策略
>
> <img src="https://cdn.jsdelivr.net/gh/amonstercat/blog-images/%E6%97%81%E8%B7%AF%E7%BC%93%E5%AD%98.webp" style="zoom: 67%;" />
>
> 
>
> **写策略的步骤：**
>
> - 更新数据库中的数据；
> - 删除缓存中的数据
>
> 
>
> **读策略的步骤：**
>
> - 如果读取的数据命中了缓存，则直接返回数据(这种情况**只可能是连续发生两次读请求**)；
> - **如果读取的数据没有命中缓存，则从数据库中读取数据，然后将数据写入到缓存，并且返回给用户**
>
> 
>
> 而针对写策略，到底该选择哪种顺序呢？
>
> - 先删除缓存，再更新数据库；
> - 先更新数据库，再删除缓存；
>
> 下面会结合实际业务场景来说明一下

先来说明一下业务场景：在线课堂中有一个课程信息页面，学生用户和教师用户可以浏览课程详情。

假设用户A(教师)、B(学生)进入课程详情页：

A负责修改某一门课程的详情页(写请求)，B负责读课程详情(读请求)，这种情况下如果A先删除了`redis`缓存，再执行更新操作到数据库，而B线程在中间负责读，会发生什么？

1. 假设教师A先删除了`Redis`中的数据，然后执行更新操作，但是`update`操作通常比较费时；
2. 在教师A执行更新操作期间，B线程想要查询课程详情，先查`Redis`发现缓存中没有课程数据，故再去查数据库，查到了旧的数据`old-comment`并把`old-comment`再次写入缓存中；
3. 在B执行完查询操作后A才完成了`update`操作，此时A再去修改数据库，数据库中存放的就是`new-comment`，这就导致了数据库与缓存的不一致性！(因为正常情况下A执行完`update`操作后 ，后续的读请求会先查询缓存，发现缓存中没有后再去查数据库，然后把查到的数据写入至缓存，这种情况是不会有问题的！)



解决方案就是延迟双删策略：(以评论举例)

1. 用户A想要添加评论，但在添加之前，先删除Redis缓存中的评论数据，然后执行数据库的更新操作
2. 在用户A执行数据库更新操作期间，设置一个短暂的延迟时间（例如几十毫秒或一百毫秒——>延迟时间的设置是为了保证在更新操作期间，用户B有机会先读取旧的评论数据并写回到Redis缓存中，以避免数据不一致性的问题。但是如果设置的太短了，用户B就会读取到用户A完成`update`操作后数据库内的新值，并再次把新值写入缓存，这时候延迟双删策略就没有必要了，）
3. 如果用户B同时想要读取评论，在这短暂的延迟时间内，用户B会发现Redis缓存中没有评论数据，于是从数据库中读取旧的评论数据（如果有的话），并将这些旧的评论数据再次写入Redis缓存
4. 当延迟时间结束后，用户A完成数据库的更新操作，将最新的评论数据写入数据库
5. 在数据库更新后的短暂时间内，再次删除Redis缓存中的评论数据，以保持数据库和缓存的一致性



该方法的弊端如下：

(1) 首先要了解数据库的性能，包括更新操作的平均耗时。延迟时间应该略大于数据库更新操作的平均耗时，以确保大部分情况下数据库更新已经完成。

(2) 考虑业务逻辑和数据处理时间。如果更新评论的业务逻辑较复杂，可能需要更长的延迟时间

(3) 延迟时间不宜设置得过长，以避免影响用户体验。用户在添加评论后，可能期望立即看到自己的评论。因此，延迟时间应尽量控制在用户可以接受的范围内。



代码实现如下：

1、`Service`层：

```java
// CommentService.java
@Service
public class CommentService {
    @Autowired
    private CommentRepository commentRepository;
    
    @Autowired
    private RedisTemplate<String, Comment> redisTemplate;

    @Transactional // 事务管理
    //写策略 采用延迟双删策略
    public void addCommentWithDelay(Comment comment) {
        // Step 1: Delete Redis cache before updating the database
        String cacheKey = "comment:" + comment.getId();
        redisTemplate.delete(cacheKey);

        
        /* Step 2:
        为读取操作（用户 B）提供时间，以从 Redis 检索旧数据并写回
        */
        try {
            //这里的延迟时间一般设置为几百毫秒
            Thread.sleep(100); // Add a short delay (e.g., 100 milliseconds) 
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // Step 3: Update the database with the new comment 完成mysql的更新操作
        commentRepository.save(comment);

        // Step 4: After database update, delete the Redis cache again for consistency
        //再次删除缓存
        redisTemplate.delete(cacheKey);
    }

    public Comment getComment(Long commentId) {
        String cacheKey = "comment:" + commentId;
        Comment cachedComment = redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedComment == null) {
            // 读策略：如果redis里有则直接读，没有则去读数据库读完再去写入redis
            cachedComment = commentRepository.findById(commentId).orElse(null);
            if (cachedComment != null) {
                redisTemplate.opsForValue().set(cacheKey, cachedComment);
            }
        }

        return cachedComment;
    }
}

```



2、`Controller`层：

```java
@RestController
public class CommentController {
    @Autowired
    private CommentService commentService;

    @PostMapping("/comments")
    public ResponseEntity<String> addComment(@RequestBody Comment comment) {
        commentService.addCommentWithDelay(comment);
        return ResponseEntity.ok("Comment added successfully.");
    }

    @GetMapping("/comments/{commentId}")
    public ResponseEntity<Comment> getComment(@PathVariable Long commentId) {
        Comment comment = commentService.getComment(commentId);
        if (comment != null) {
            return ResponseEntity.ok(comment);
        } else {
            return ResponseEntity.notFound().build();
        }
    }
}
```



> 问题1：**「如果用了mysql的读写分离架构怎么办？」** :heavy_exclamation_mark: :heavy_exclamation_mark:
>
> 在这种情况下，造成数据不一致的原因如下，还是两个请求，一个请求A进行更新操作，另一个请求B进行查询操作。
>
> （1）请求A进行写操作，删除缓存
>
> （2）请求A将数据写入数据库了，
>
> （3）请求B查询缓存发现，缓存没有值
>
> （4）请求B去从库查询，**这时，还没有完成主从同步，因此查询到的是旧值**
>
> （5）请求B将旧值写入缓存
>
> （6）数据库完成主从同步，从库变为新值，导致数据库与缓存不一致性
>
> 
>
> 解决方案依旧采取双删延时策略。只是，**睡眠时间修改为在主从同步的延时时间基础上**，加几百ms。
>
> 
>
> 问题2：**「采用这种同步淘汰策略，吞吐量降低怎么办？」**
>
> > 那就将第二次删除作为异步的。自己起一个线程，异步删除。这样写的请求就不用沉睡一段时间后(因为整个写请求包括了最后一次删除缓存的时间)再返回了。这么做就可以加大吞吐量



我们最后再来看一下**异步删除缓存**是如何做到的(这可能也是该项目中的最优解)？

1. 使用`Rabbitmq`实现异步删除：

```java
@Service
public class CommentService {
    @Autowired
    private CommentRepository commentRepository;

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Transactional
    public void addCommentWithAsyncCacheEviction(Comment comment) {
        // 步骤1：在更新数据库之前，删除 Redis 缓存
        String cacheKey = "comment:" + comment.getId();
        redisTemplate.delete(cacheKey);

        // 步骤2：更新数据库中的评论信息
        commentRepository.save(comment);

        // 步骤3：向消息队列发送消息异步执行缓存删除任务 (这里也可以使用Futrue类)
        rabbitTemplate.convertAndSend("comment-cache-eviction-queue", cacheKey);
        //  非阻塞的，会立即返回，不会等待消费者的处理结果。
    }
}

@Component
public class CommentCacheEvictionConsumer {

    @Autowired
    private RedisTemplate<String, Comment> redisTemplate;
    
    // 消息的处理是在与应用程序主线程分离的新线程中完成的，而不会阻塞主线程的执行
    @RabbitListener(queues = "comment-cache-eviction-queue")  //监听哪个队列
    public void handleCacheEviction(String cacheKey) {
        // 异步删除 Redis 缓存
        redisTemplate.delete(cacheKey);
    }
}
```







2. 使用JUC中自带的`Future`类实现：

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

@Service
public class CommentService {
    @Autowired
    private CommentRepository commentRepository;

    @Autowired
    private RedisTemplate<String, Comment> redisTemplate;

    private ExecutorService executorService = Executors.newFixedThreadPool(5); // 创建线程池，大小为5

    public void addCommentWithAsyncCacheEviction(Comment comment) {
        
        //步骤1：第一次删除缓存 
        
        // 步骤2：更新数据库中的评论信息
        commentRepository.save(comment);
        
        // 步骤3：第二次删除 Redis 缓存（异步操作）
        String cacheKey = "comment:" + comment.getId();
        executorService.execute(() -> {
            redisTemplate.delete(cacheKey);
        });

        
    }
}
```

需要注意的是，`execute` 方法是一个无返回值的异步操作，如果希望异步任务有返回值，可以使用 `submit` 方法，并配合 `Future` 或 `CompletableFuture` 获取异步任务的结果。

例如，使用 `CompletableFuture` 后修改一小部分代码即可：

```java
public CompletableFuture<Void> addCommentWithAsyncCacheEviction(Comment comment) {
    
    // 第一次删除缓存

    // 步骤1：更新数据库中的评论信息
    commentRepository.save(comment);

    // 步骤2：删除 Redis 缓存（异步操作）
    String cacheKey = "comment:" + comment.getId();
    CompletableFuture<Void> cacheEvictionFuture = CompletableFuture.runAsync(() -> {
        redisTemplate.delete(cacheKey);
    }, executorService);

    return cacheEvictionFuture; // 返回删除缓存的 CompletableFuture
}
```





还有一个场景： 每个课程有一个课程详情，假如教师管理员要修改某一门课程的描述(`CourseInfo`)，此时有学生要读该课程，这也涉及到了数据库与缓存一致性的问题







## 如何使用线程池技术异步执行/提高响应速度？

> 先来看看业务场景：网络在线课堂涉及到的业务中，有许多适合使用异步执行的场景。异步执行可以提高用户体验，增加系统的并发能力，以及优化后台任务的处理速度。以下是一些适合使用异步执行的在线课堂业务场景：
>
> 1. **视频转码和处理**：当老师上传视频课程时，后台可以异步进行视频转码和处理，将视频格式转换为适合在线播放的格式，以及提取视频封面图等。
> 2. **异步通知和消息推送**：课程开放、新课程发布、作业截止等事件可以通过异步方式发送通知和消息推送给学生或老师。
> 3. **文件上传和处理**：学生提交作业或其他资料时，后台可以异步处理文件上传和存储，确保上传的文件可以快速保存。
> 4. **课程评价和统计**：课程评价和统计可能涉及到较复杂的数据处理，可以使用异步方式进行数据的计算和更新。
> 5. **生成课程报表**：生成课程相关的报表和统计信息可能耗时较长，可以通过异步执行来避免影响其他业务的响应速度。



**场景1：当有新课程发布时，通过异步方式发送通知和消息推送给学生**



后台执行了以下任务：

1. **异步通知和消息推送给学生**：当有新课程发布事件发生时，后台会通过异步方式进行通知和消息推送给相关的学生。这些通知可以是课程开放通知、新课程发布通知、作业截止通知等。
2. **其他处理逻辑**：后台可能还会执行一些其他处理逻辑，例如更新数据库中的课程信息、生成课程报表、发送邮件或短信通知等。

前台（或客户端）执行的任务是触发新课程发布事件，即调用`/publish`接口来发布新课程。这个过程是用户在前台（或客户端）进行的交互操作，通过调用接口告知后台有新课程发布



总体流程如下：

1. 用户（前台）调用`/publish`接口发布新课程，该接口由`CourseController`中的`publishNewCourse`方法处理。
2. `publishNewCourse`方法会发布`NewCourseEvent`事件，该事件包含了新课程的相关信息。
3. `Spring`框架会将`NewCourseEvent`事件传递给`CourseNotificationService`中的`handleNewCourseEvent`方法进行处理。因为`handleNewCourseEvent`方法使用了`@Async`注解，所以该方法会被异步执行。
4. `handleNewCourseEvent`方法中的异步任务会在后台线程中执行，执行内容包括异步通知和消息推送给学生，以及其他处理逻辑(真正的`Insert`操作)。

**前台（或客户端）执行的任务是发布新课程事件，告知后台有新课程发布。后台执行的任务是处理该事件，包括异步通知和消息推送给学生**，以及其他相关处理逻辑。通过异步执行，可以实现异步处理这些任务，提高接口的响应速度，并且将一些耗时的操作放到后台执行，避免阻塞主线程



通常情况下，发布新课程涉及的`insert`操作逻辑应该是在后台异步做的。在典型的设计中，前台（或客户端）只负责接收用户的输入和触发事件，而后台负责处理具体的业务逻辑，包括数据库操作等。

具体流程如下：  **观察者模式:exclamation:** :exclamation: :exclamation: 

1. 前台（或客户端）接收用户输入的新课程信息，例如课程名称、课程描述、教师信息等。
2. 前台通过调用`/publish`接口来触发新课程发布事件。该接口由`CourseController`中的`publishNewCourse`方法处理。
3. `publishNewCourse`方法会发布`NewCourseEvent`事件，该事件包含了新课程的相关信息。
4. `Spring`框架会将`NewCourseEvent`事件传递给`CourseNotificationService`中的`handleNewCourseEvent`方法进行处理。因为`handleNewCourseEvent`方法使用了`@Async`注解，并且通过自定义线程池异步执行，所以该方法会在后台线程中执行。
5. 在`handleNewCourseEvent`方法的异步执行中，包含了具体的业务逻辑，**例如将新课程的信息插入数据库（`insert`操作）。这些数据库操作是在后台异步线程(线程池)中执行的。**

通过异步执行数据库操作，可以避免数据库操作对主线程的阻塞，提高了接口的响应速度。同时，将数据库操作放在后台执行，也可以更好地管理数据库连接和资源，以及提高系统的并发性能。

总结：发布新课程涉及的`insert`操作逻辑应该是在后台异步做的，具体是在`CourseNotificationService`中的`handleNewCourseEvent`方法的异步执行中处理数据库操作。前台（或客户端）只负责触发事件，不需要关心具体的数据库操作和后台异步处理的细节。



然后我们来看看代码如何实现？

1、 首先自定义一个线程池`Bean`:

```java
@Configuration
@EnableAsync
/*
@EnableAsync 是 Spring 框架中的一个注解，它用于启用异步方法的支持。
当在 Spring 应用中使用了 @EnableAsync 注解，Spring 就会创建一个异步任务执行器，
以便在后台执行带有 @Async 注解的方法
*/
public class ThreadConfig {
    
@Bean("asyncExecutor")
public Executor asyncServiceExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();// 设置核心线程数
    executor.setCorePoolSize(5);
// 设置最大线程数
    executor.setMaxPoolSize(20);
    //配置队列大小
    executor.setQueueCapacity(Integer.MAX_VALUE);// 设置线程活跃时间(秒)
    executor.setKeepAliveSeconds(60);
// 设置默认线程名称
    executor.setThreadNamePrefix("码神之路博客项目");// 等待所有任务结束后再关闭线程池
    executor.setWaitForTasksToCompleteOnShutdown(true);//执行初始化
    executor.initialize();
    return executor;
}
}
```

在上述配置中，我们通过`ThreadPoolTaskExecutor`定义了一个名为`asyncExecutor`的线程池，设置了核心线程数为5，最大线程数为20，任务队列容量为100



2、接下来，创建一个事件监听器，用于处理新课程发布事件，并在监听方法上使用`@Async`注解标记为异步方法：

```java
@Service
public class CourseNotificationService {

    @Autowired
    private final Executor asyncExecutor;
                                          //这里的参数要对应创建的线程池
    public CourseNotificationService(@Qualifier("asyncExecutor") Executor asyncExecutor) {
        this.asyncExecutor = asyncExecutor;
    }

    @EventListener
    @Async                            //该监听器监听NewCourseEvent类型的事件
    public void handleNewCourseEvent(NewCourseEvent event) {
        
        
        
        // 通过自定义线程池异步执行
        asyncExecutor.execute(() -> {
            // 获取新课程信息
            String courseId = event.getCourseId();
            // 其他处理逻辑，例如发送通知和消息推送给学生
            System.out.println("异步方式发送通知和消息推送给学生，新课程ID：" + courseId);
        });
    }
}

```

在`CourseNotificationService`中，我们将异步任务的执行交给了我们自定义的线程池`asyncExecutor`。这样，当有新课程发布事件发生时，异步任务将在我们的线程池中执行



> - `@Async`注解默认会使用Spring的异步任务执行器来管理线程池和任务调度。你可以通过在配置类中进行适当的设置来自定义异步任务执行器
>
>   
>
> - 确保`@EnableAsync`注解在一个配置类中，并且这个配置类被Spring扫描到
>
>   
>
> - 异步方法必须是`public`修饰的，因为Spring通过代理机制来实现异步调用
>
> 





3、最后，当有新课程发布时，通过发布`NewCourseEvent`事件，即可触发异步通知：

```java
@RestController
public class CourseController {

    private final ApplicationEventPublisher eventPublisher;//事件发布器

    public CourseController(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    @PostMapping("/publish")
    public String publishNewCourse(@RequestBody NewCourseEvent newCourseEvent) {
        // 新课程发布逻辑...
        // 发布新课程事件
        eventPublisher.publishEvent(newCourseEvent);
        return "New course published.";
    }
}
```



通过在`CourseNotificationService`中使用`@Async`注解和自定义线程池，我们实现了对业务的异步处理，而对`Controller`层的代码没有任何影响。当有新课程发布时，`Controller`层调用发布新课程事件的方法，将事件发布给后台的`Service`层。`Service`层中的`handleNewCourseEvent`方法被异步执行，使用了自定义的线程池









## 消息队列的应用(包含了视频上传部分:exclamation: )



> 在解决数据库缓存一致性问题时使用到了`rabbitmq`来异步执行缓存删除工作，这里就不赘述了



场景1:基于`MinIO`实现视频上传功能时,使用消息队列来后台执行视频处理逻辑



(1)首先我们来看一下`Controller`层的实现：

当管理员通过`POST`请求访问 "`/uploadVideo`" 接口，并提供视频文件的路径，视频上传的请求会被接收并传递给 `VideoUploader` 的 `uploadVideo` 方法。`VideoUploader` 会将视频上传到`MinIO`，并将视频处理任务发送到`RabbitMQ`的消息队列中。然后，视频处理消费者 `VideoProcessingConsumer` 会监听该消息队列，收到消息后调用 `VideoProcessor` 的 `processVideo` 方法来处理视频

```java
@RestController
public class VideoUploadController {

    @Autowired
    private VideoUploader videoUploader;
    @PostMapping("/uploadVideo")
    public String uploadVideo(@RequestParam("videoPath") String videoPath) {
        try {
            // 管理员上传视频
            videoUploader.uploadVideo(videoPath);
            return "Video upload request received. Video will be processed asynchronously.";
        } catch (Exception e) {
            return "Failed to upload video: " + e.getMessage();
        }
    }
}
```



(2) 再看看`VideoUploader`部分做了哪些事情：

```java

@Service
public class VideoUploader {

    @Autowired
    private MinioClient minioClient;

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void uploadVideo(String videoPath) {
        try {
            // 上传视频到MinIO
            String bucketName = "videos"; // 假设存储桶名称为 "videos"
            String objectName = "video_" + System.currentTimeMillis() + ".mp4"; // 生成唯一的对象名称
            minioClient.putObject(PutObjectArgs.builder()
                                  .bucket(bucketName)
                                  .object(objectName)
                                  .filename(videoPath)
                                  .build());

            // 将视频处理任务发送到RabbitMQ消息队列
            String queueName = "video-processing-queue";
            rabbitTemplate.convertAndSend(queueName, objectName);
        } catch (Exception e) {
            // 处理异常
        }
    }
}
```



(3)  `VideoProcessingConsumer`视频处理消费者部分： 消息处理方

```java
@Component
public class VideoProcessingConsumer {

    @Autowired
    private VideoProcessor videoProcessor;

    @RabbitListener(queues = "video-processing-queue")
    public void handleVideoProcessing(String objectName) {
        try {
            
            // 执行视频处理逻辑
            
            // 从MinIO下载视频
            String bucketName = "videos"; // 存储桶名称
            minioClient.getObject(GetObjectArgs.builder()
                                  .bucket(bucketName)
                                  .object(objectName)
                                  .build());

            // TODO: 进行视频处理逻辑，例如视频转码、压缩、存储等操作
            
            videoProcessor.processVideo(objectName);
        } catch (Exception e) {
            // 处理异常
        }
    }
}
```











场景2: 实现支付结果通知

<img src="https://cdn.jsdelivr.net/gh/amonstercat/blog-images/mq.png" style="zoom:67%;" />

1. `PayNotifyConfig.java`

```java
@Configuration
public class PayNotifyConfig {

  //交换机
  public static final String PAYNOTIFY_EXCHANGE_FANOUT = "paynotify_exchange_fanout";

  //支付结果处理反馈队列
  public static final String PAYNOTIFY_REPLY_QUEUE = "paynotify_reply_queue";

  //支付结果通知消息类型
  public static final String MESSAGE_TYPE = "payresult_notify";

  //声明交换机 注入bean
  @Bean(PAYNOTIFY_EXCHANGE_FANOUT)
  public FanoutExchange paynotify_exchange_fanout(){
     // 三个参数：交换机名称、是否持久化、当没有queue与其绑定时是否自动删除
     return new FanoutExchange(PAYNOTIFY_EXCHANGE_FANOUT, true, false);
  }

```





2. `PayNotifyTask.java`

```java
@Component
public class PayNotifyTask extends MessageProcessAbstract {

    @Autowired
    RabbitTemplate rabbitTemplate;

    @Autowired
    MqMessageService mqMessageService;

    //任务调度入口
    @XxlJob("NotifyPayResultJobHandler")
    public void notifyPayResultJobHandler() throws Exception {
        // 分片参数
        int shardIndex = XxlJobHelper.getShardIndex();
        int shardTotal = XxlJobHelper.getShardTotal();
        log.debug("shardIndex="+shardIndex+",shardTotal="+shardTotal);
        //只查询支付通知的消息
        process(shardIndex,shardTotal, PayNotifyConfig.MESSAGE_TYPE,100,60);
    }


    //执行任务的具体方法
    @Override
    public boolean execute(MqMessage mqMessage) {
        log.debug("向消息队列发送支付结果通知消息:{}",mqMessage);
        //发布消息
        send(mqMessage);  //使用交换机广播模式发送消息

        return false;
    }

    //监听支付结果通过回复队列
    //接收回复
    @RabbitListener(queues = PayNotifyConfig.PAYNOTIFY_REPLY_QUEUE)   //指定监听哪一个队列
    public void receive(String message) {
        log.debug("收到支付结果通知回复:{}",message);
        MqMessage mqMessage = JSON.parseObject(message, MqMessage.class);
        //完成消息发送，最终该消息删除了
        mqMessageService.completed(mqMessage.getId());
    }


    /**
     * @description 发送支付结果通知
     * @param message  消息内容
     * @return void
     * @author Mr.M
     * @date 2022/9/20 9:43
     */
    private void send(MqMessage message){
            //要发送的消息
        String jsonStringMsg = JSON.toJSONString(message);

        //开始发送消息,使用fanout交换机，通过广播模式发送消息
        rabbitTemplate.convertAndSend(PayNotifyConfig.PAYNOTIFY_EXCHANGE_FANOUT,"",jsonStringMsg);

        log.debug("向消息队列发送支付结果通知消息完成:{}",message);

    }
}

```





场景3：在学生选课时，将选课请求异步处理，而不是直接阻塞用户线程等待选课结果返回

假设学生选课系统中有一个服务叫做"`CourseService`"，它负责处理学生选课请求。当学生提交选课请求时，`CourseService`可以将选课任务发送到`RabbitMQ`消息队列，然后**立即返回给用户一个响应，告知选课请求已接收并正在处理**。

另外，有一个名为"`CourseProcessor`"的异步服务，它从`RabbitMQ`消息队列中接收选课任务，并进行选课处理。一旦选课处理完成，`CourseProcessor`可以将选课结果发送到另一个消息队列，比如"`SelectionResultQueue`"

然后，可以有另一个服务，比如"NotificationService"，它从"SelectionResultQueue"中接收选课结果，并通知相关的学生选课结果，比如发送邮件或短信通知选课成功或失败。

这样的异步处理流程可以有效地将选课请求和选课结果的处理解耦，并且可以在系统负载较高时提高吞吐量，因为选课请求不会直接阻塞在处理中。

简要流程如下：
1. 学生提交选课请求到`CourseService`
2. `CourseService`将选课任务发送到Ra·bbitMQ消息队列
3. `CourseService`立即返回响应给学生，告知选课请求已接收
4. `CourseProcessor`从`RabbitMQ`消息队列中接收选课任务，进行选课处理
5. `CourseProcessor`将选课结果发送到"`SelectionResultQueue`"
6. **`NotificationService`从"`SelectionResultQueue`"接收选课结果，并通知学生选课结果**

使用`RabbitMQ`实现异步处理可以有效地提高学生选课系统的性能和可伸缩性，确保系统在高负载情况下也能保持高效运行



















## 分布式锁在项目中的具体应用？



微服务项目中，一个微服务通常会有多个实例在运行(用户请求到来时涉及到路由负载均衡)，打到一个特定实例上的多个线程请求，针对这一个实例(单个`jvm`进程)使用同步锁`synchronized`是可以实现这些线程的同步操作的，但是如果想要在多个实例(多个`jvm`进程)上实现所有线程请求的同步，这就涉及到了分布式锁，本项目中使用`Redisson`来实现分布式锁

先来看看业务场景：

选课冲突处理：当多个学生同时选择某门课程时，可能发生选课冲突。为了避免冲突，可以使用分布式锁来确保同时只有一个学生能够成功选取该课程



然后我们再来讲一下优化的过程

(1) 使用`setnx`命令，设置过期时间但是过期时间不好确定(取决于业务)，问题有两点：

- 设置的过期时间较短：让key到期后自动释放，可能当前线程操作未完成锁就被释放；

- 手动删除：每次当前线程执行完后在finally中手动释放锁(`del key`),但是由于锁已经过期，锁当前被别人占用，此时再删除就删掉的是其他线程持有的锁了

  

(2) 依旧使用`setnx`命令，不设置过期时间，由当前线程执行完操作后判断锁是否还是之前所设置的，是的话再手动删除

这种方式也存在问题，因为**判断锁与释放锁的代码不是一个原子性操作，多线程场景下CPU是基于时间片轮转调度的，假如当前线程判断现在的锁是它所设置的``<K,V>=<"lock",01>`，但是马上让出了时间片给线程2，且此时锁失效，线程2判断到锁失效故执行`setnx`命令设置为(`<K,V>=<"lock",02>`)，然后再次让出时间片给线程1，此时线程1执行删除锁命令就把线程2的锁给删掉了**！！

```java
finally{
    if(redis.call("get","lock")=="01")
    {
        释放锁： redis.call("del","lock");
    }
}
```

(3) 解决以上原子性问题的办法是基于lua脚本，lua脚本在执行上述操作时可以保证原子性，但是也存在问题，**因为如果当前实例突然崩溃，`finally`中的代码没有执行呢**?由于没有设置过期时间，这个锁就永远不会被释放掉！！

这里给出基于`setnx`的分布式锁实现：

```java
//通过setnx实现分布式锁  2.0
public CoursePublish getCoursePublishCache2(Long courseId){

    //先从缓存中查询
    String jsonString = (String) redisTemplate.opsForValue().get("course:" + courseId);
    if(StringUtils.isNotEmpty(jsonString)){
        //System.out.println("========从缓存中查询===========");
        //将json转成对象返回
        CoursePublish coursePublish = JSON.parseObject(jsonString, CoursePublish.class);
        return coursePublish;
    }else{
        //使用setnx向redis设置一个key，谁设置成功谁拿到了分布式锁
        Boolean lock001 = redisTemplate.opsForValue().setIfAbsent("lock001", "001");
        if(lock001){
            //获取锁这个人去执行这里边的代码
        }
        synchronized (this){
            //再次从缓存中查询一下
            jsonString = (String) redisTemplate.opsForValue().get("course:" + courseId);
            if(StringUtils.isNotEmpty(jsonString)){
                //将json转成对象返回
                CoursePublish coursePublish = JSON.parseObject(jsonString, CoursePublish.class);
                return coursePublish;
            }
            System.out.println("从数据库查询...");
            //如果缓存中没有，要从数据库查询
            CoursePublish coursePublish = coursePublishMapper.selectById(courseId);
            //将从数据库查询到的数据存入缓存
            redisTemplate.opsForValue().set("course:" + courseId,JSON.toJSONString(coursePublish),300, TimeUnit.SECONDS);

            return coursePublish ;
        }
    }
}
```



(4) 基于`redisson`实现，`redisson`分布式锁优势如下：

- 将多个`redis`操作lua脚本作为整体提交，保证性能的同时保证整体原子性

- 看门狗自动延续锁生命周期，防止未处理完锁过期问题，但是同时造成了阻塞，甚至锁死

- 实现了自旋锁 ：发现锁被占用后`get ttl`进行`while true`对应的时间

- 实现了重入锁 ：发现锁后再看一下`clientid`是不是自己，如果是+1

```java
//解决缓存击穿，使用Redission实现分布式锁 3.0
public CoursePublish getCoursePublishCache(Long courseId){
    //先从缓存中查询
    String jsonString = (String) redisTemplate.opsForValue().get("course:" + courseId);
    if(StringUtils.isNotEmpty(jsonString)){
        System.out.println("========从缓存中查询===========");
        //将json转成对象返回
        CoursePublish coursePublish = JSON.parseObject(jsonString, CoursePublish.class);
        return coursePublish;
    }else{
        //使用setnx向redis设置一个key，谁设置成功谁拿到了锁
        // Boolean lock001 = redisTemplate.opsForValue().setIfAbsent("lock001", "001",300,TimeUnit.SECONDS);
        //使用redisson获取锁
        /*
        *当使用Redisson的getLock方法获取锁时，如果对应的key在Redis中不存在，Redisson          会自动在Redis中设置一个键值对（key-value）
        */
        RLock lock = redissonClient.getLock("coursequery:" + courseId);  //为每个course都设置锁
        //获取分布式锁
        lock.lock();
        //拿到锁的才会执行try finally代码块
        try{
            //   Thread.sleep(35000);
            //再次从缓存中查询一下
            jsonString = (String) redisTemplate.opsForValue().get("course:" + courseId);
            if(StringUtils.isNotEmpty(jsonString)){
                //将json转成对象返回
                CoursePublish coursePublish = JSON.parseObject(jsonString, CoursePublish.class);
                return coursePublish;
            }
            System.out.println("从数据库查询...");
            //如果缓存中没有，要从数据库查询
            CoursePublish coursePublish = coursePublishMapper.selectById(courseId);
            //将从数据库查询到的数据存入缓存 ,缓存中设置Key为 "course:id"
            redisTemplate.opsForValue().set("course:" + courseId,JSON.toJSONString(coursePublish),300, TimeUnit.SECONDS);
            return coursePublish ;
        }finally {
            //释放锁 不管过不过期 业务完成后还是必须手动释放
            lock.unlock();  // redisson解锁方法内部封装了lua脚本保证原子性
        }
    }
}
```

核心就是为每个被抢的课程在`redis`中`set`一个`courseid`，所有选课用户同时`get`该`courseid`对应的锁，抢到的才能执行后续逻辑





再考虑一个问题，虽然`Redisson`解决了锁过期时间无法确定的问题(通过后台`WatchDog`线程实现锁自动续期)，但是假如当前线程出现异常并没有执行`finally`代码块中的`unlock()`方法，这和之前`setnx`命令实现分布式锁出现的问题一样。我们来看看`redisson`源码是如何保证锁一定会被释放的：

```java
// 锁释放，调用unlockAsync(Thread.currentThread().getId()) 方法来异步执行锁的释放操作
public void unlock() {
    try {
        get(unlockAsync(Thread.currentThread().getId()));
    } catch (RedisException e) {
        if (e.getCause() instanceof IllegalMonitorStateException) {
            throw (IllegalMonitorStateException) e.getCause();
        } else {
            throw e;
        }
    }
}

// 进入 unlockAsync(Thread.currentThread().getId()) 方法 入参是当前线程的id
public RFuture<Void> unlockAsync(long threadId) {
    RPromise<Void> result = new RedissonPromise<Void>();
    //执行lua脚本 删除key
    RFuture<Boolean> future = unlockInnerAsync(threadId);

    future.onComplete((opStatus, e) -> {
        // 无论执行lua脚本是否成功 执行cancelExpirationRenewal(threadId) 方法来删除EXPIRATION_RENEWAL_MAP中的缓存
        cancelExpirationRenewal(threadId); //停止锁的续期机制

        if (e != null) {
            result.tryFailure(e);
            return;
        }

        if (opStatus == null) {
            IllegalMonitorStateException cause = new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                    + id + " thread-id: " + threadId);
            result.tryFailure(cause);
            return;
        }

        result.trySuccess(null);
    });

    return result;
}


// 此方法会停止 watch dog 机制
/*
从 EXPIRATION_RENEWAL_MAP 中查找与当前锁关联的续期任务，
然后根据 threadId 来删除对应的线程标识，并检查是否还有其他线程在持有锁。
如果没有其他线程持有锁，就会取消续期任务并将其从 EXPIRATION_RENEWAL_MAP 中移除。
*/

void cancelExpirationRenewal(Long threadId) {
    ExpirationEntry task = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (task == null) {
        return;
    }
    
    if (threadId != null) {
        task.removeThreadId(threadId);
    }

    if (threadId == null || task.hasNoThreads()) {
        Timeout timeout = task.getTimeout();
        if (timeout != null) {
            timeout.cancel();
        }
        EXPIRATION_RENEWAL_MAP.remove(getEntryName());
    }
}
```



释放锁的操作中 有一步操作是从 EXPIRATION_RENEWAL_MAP 中获取 `ExpirationEntry` 对象，然后将其remove，结合watch dog中的续期前的判断：

```java
EXPIRATION_RENEWAL_MAP.get(getEntryName());
if (ent == null) {
    return;
}
```

可以得出结论：如果释放锁操作本身异常了，`watch dog` 还会不停的续期吗？不会，因为无论释放锁操作是否成功，`EXPIRATION_RENEWAL_MAP`中的目标 `ExpirationEntry` 对象已经被移除了(因为线程异常退出了，在保存当前活跃线程的Map里也找不到了)，`watch dog`通过判断后就不会继续给锁续期了



当然`Redisson`实现分布式锁也有问题，如果`Redis`配置了集群的情况下(集群中节点宕机导致数据转移) ，后面就是`RedLock`的事了





## 项目中遇到了哪些问题 怎样解决？



1、`ThreadLocal`实现线程隔离：

保证多个用户（例如 A、B、C）可以同时预约同一个特定时间段（例如 8点到9点）的课程。使用 `ThreadLocal` 可以确保每个线程（即每个用户）的预约请求都是独立的，不会相互干扰

首先，我们创建一个 `ReservationInfo` 实体类来保存预约信息：

```java
public class ReservationInfo {
private Long userId;
private Long courseId;
// 其他预约信息，如预约时间等

// 省略构造函数、getter和setter方法
}
```



2、然后在`ReservationService` 中使用 `ThreadLocal` 来存储当前线程的预约信息：

```java
public class ReservationService {
    private static ThreadLocal<ReservationInfo> threadLocal = new ThreadLocal<>();
    private Map<Long, List<Course>> userCoursesMap = new HashMap<>();

    public void reserveCourse(Long userId, Long courseId) {
        // 从数据库或其他地方获取课程信息
        Course course = getCourseById(courseId);

        if (course != null) {
            // 创建预约信息
            ReservationInfo reservationInfo = new ReservationInfo();
            reservationInfo.setUserId(userId);
            reservationInfo.setCourseId(courseId);
            // 其他预约信息的设置，如预约时间等

            // 将预约信息存入 ThreadLocal
            threadLocal.set(reservationInfo);

            // 处理预约逻辑
            if (checkIfCanReserve(course)) {
                // 可以预约，将预约的课程添加到用户的预约列表中
                List<Course> userCourses = userCoursesMap.computeIfAbsent(userId, k -> new ArrayList<>());
                userCourses.add(course);
                System.out.println("用户 " + userId + " 成功预约课程 " + course.getCourseName());
            } else {
                System.out.println("用户 " + userId + " 不能预约课程 " + course.getCourseName());
            }

            // 预约完成后，从 ThreadLocal 中清除预约信息，确保线程池中的线程不会复用 ThreadLocal 中的数据
            threadLocal.remove();
        } else {
            System.out.println("课程 " + courseId + " 不存在");
        }
    }

    private boolean checkIfCanReserve(Course course) {

        // 假设这里需要根据课程的容量和已预约人数来判断是否满额
        // 假设 Course 类有相应的容量和已预约人数属性，这里只是一个示例
        int capacity = course.getCapacity();
        int reservedCount = course.getReservedCount();
        return reservedCount < capacity;
    }


    return true;
}

private Course getCourseById(Long courseId) {
    // 这里假设根据课程ID从数据库或其他地方获取课程信息
    // 这里只是一个示例，实际实现需要根据具体的数据访问方式来实现
    // 在示例中，我们直接返回一个虚拟的课程对象
    return new Course(courseId, "课程A");
}
}

```

在上述示例中，当用户发起预约请求时，`reserveCourse` 方法首先将预约信息存入 `ThreadLocal` 中，然后进行预约处理。在预约处理完成后，需要调用 `threadLocal.remove()` 方法将 `ThreadLocal` 中的预约信息清除，以确保线程池中的线程不会复用 `ThreadLocal` 中的数据

这样，当**多个用户同时发起预约请求时，每个线程都有自己独立的预约信息存储在 `ThreadLocal` 中，从而避免了用户预约冲突的问题**







2、如何排查项目中出现的`OOM`？





# 实习项目问题梳理



## 1、同步第三方渠道打卡数据时接口限流



### 单机限流：Guava-RateLimiter

原先使用的是基于Guava的限流器,以获取飞书打卡数据接口为例：

```java
public class AccessLimitService {
    
      private final ConcurrentHashMap<String, RateLimiter> larkRateLimiterMap = new ConcurrentHashMap<>();
    
       // 存储以entId为键的限流器对象的Map，按照企业ID对获取飞书打卡数据来进行限流
      private final ConcurrentHashMap<Long, RateLimiter> absLarkRateLimiterConcurrentHashMap = new ConcurrentHashMap<>();
      private final int RATE_LIMIT_COUNT = 50;
      private static final int DING_RATE_LIMIT_COUNT = 19;

      public void larkAcquireByCorpId(String corpId) {
        RateLimiter rateLimiter = larkRateLimiterMap.get(corpId);
        if (Objects.isNull(rateLimiter)) {
            rateLimiter = RateLimiter.create(RATE_LIMIT_COUNT);
            larkRateLimiterMap.put(corpId, rateLimiter);
        }
        rateLimiter.acquire();
    }

        /**
     * 根据entId对打卡接口的访问进行限流。
     * 打卡接口飞书单独定义接口权限每S 40个 两台机子 每个19 防止跑满
     * @param entId 企业ID
     */
    public void larkAbsenceAcquireByEnt(Long entId) {
        // 获取对应entId的限流器对象
        RateLimiter rateLimiter = absLarkRateLimiterConcurrentHashMap.get(entId);
        if (Objects.isNull(rateLimiter)) {
            // 如果限流器对象为空，则创建新的限流器对象，设置限流速率，并放入absLarkRateLimiterConcurrentHashMap中
            rateLimiter = RateLimiter.create(19);  
            absLarkRateLimiterConcurrentHashMap.put(entId, rateLimiter);
        }
        // 从限流器中获取许可，若没有可用的许可，则阻塞直到获取到许可
        rateLimiter.acquire();
    }
}
```

注意这行代码： `rateLimiter = RateLimiter.create(19);`  通过RateLimiter初始化一个限流器（这里默认创建的是平滑突发限流器），Guava是Google开源的一个工具包，其中的RateLimiter是实现了令牌桶算法的一个限流工具类。Guava的 `RateLimiter`提供了令牌桶算法实现：平滑突发限流(SmoothBursty)和平滑预热限流(SmoothWarmingUp)实现。

![image-20240328175341394](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403281753540.png)

`SmoothBursty` 和 `SmoothWarmingUp` 是两种不同的限流策略，它们在实现上有一些区别：

1. **SmoothBursty**：
   - `SmoothBursty` 实现了一种“突发式”限流策略，其中存储的许可数被转换为零限制时间。这意味着当有存储的许可时，不会施加任何限制，请求可以立即被授予。
   - 在 `SmoothBursty` 中，存储的许可数被设置为最大突发秒数乘以每秒许可数。这意味着在未使用时，存储的许可数将等于最大突发秒数乘以每秒许可数。
   - 在 `SmoothBursty` 中，`storedPermitsToWaitTime` 方法总是返回零，因为没有等待时间，请求可以立即得到服务。
2. **SmoothWarmingUp**：
   - `SmoothWarmingUp` 实现了一种“平滑预热”限流策略，其中存储的许可数被转换为等待时间。这意味着在请求到达时，可能会有一段时间的等待，直到存储的许可数足够。
   - 在 `SmoothWarmingUp` 中，存储的许可数根据预热期间的长度以及稳定间隔和冷却间隔的比率进行计算。预热期间的长度越长，存储的许可数可以累积得越多。
   - 在 `SmoothWarmingUp` 中，`storedPermitsToWaitTime` 方法会计算存储的许可数对应的等待时间，以及新请求需要的稳定间隔时间，然后返回二者之和作为总的等待时间。

总的来说，`SmoothBursty` 策略不会对请求施加任何等待时间，而 `SmoothWarmingUp` 策略会根据存储的许可数和预热期间的长度来决定请求的等待时间，以实现一种平滑的限流效果。

再来看个例子加深一下理解，针对平滑突发限流有：

使用 `RateLimiter`的静态方法创建一个限流器，设置每秒放置的令牌数为5个。返回的RateLimiter对象可以保证1秒内不会给超过5个令牌，并且以固定速率进行放置，达到平滑输出的效果。

```java
public void testSmoothBursty() {
 RateLimiter r = RateLimiter.create(5);
 while (true) {
 System.out.println("get 1 tokens: " + r.acquire() + "s");
 }
 /**
     * output: 基本上都是0.2s执行一次，符合一秒发放5个令牌的设定。
     * get 1 tokens: 0.0s 
     * get 1 tokens: 0.182014s
     * get 1 tokens: 0.188464s
     * get 1 tokens: 0.198072s
     * get 1 tokens: 0.196048s
     * get 1 tokens: 0.197538s
     * get 1 tokens: 0.196049s
     */
}
```

`RateLimiter`使用令牌桶算法，会进行令牌的累积，如果获取令牌的频率比较低，则不会导致等待，直接获取令牌。

```java
public void testSmoothBursty2() {
 RateLimiter r = RateLimiter.create(2);
 while (true)
 {
    System.out.println("get 1 tokens: " + r.acquire(1) + "s");
 try {
    Thread.sleep(2000);
 } catch (Exception e) {}
 System.out.println("get 1 tokens: " + r.acquire(1) + "s");
 System.out.println("get 1 tokens: " + r.acquire(1) + "s");
 System.out.println("get 1 tokens: " + r.acquire(1) + "s");
 System.out.println("end");
 /**
       * output:
       * get 1 tokens: 0.0s
       * get 1 tokens: 0.0s
       * get 1 tokens: 0.0s
       * get 1 tokens: 0.0s
       * end
       * get 1 tokens: 0.499796s
       * get 1 tokens: 0.0s
       * get 1 tokens: 0.0s
       * get 1 tokens: 0.0s
       */
 }
}
```

`RateLimiter`由于会累积令牌，所以可以应对突发流量。在下面代码中，有一个请求会直接请求5个令牌，但是由于此时令牌桶中有累积的令牌，足以快速响应。 `RateLimiter`在没有足够令牌发放时，采用滞后处理的方式，也就是前一个请求获取令牌所需等待的时间由下一次请求来承受，也就是代替前一个请求进行等待。

```java
public void testSmoothBursty3() {
 RateLimiter r = RateLimiter.create(5);
 while (true)
 {
    System.out.println("get 5 tokens: " + r.acquire(5) + "s");
    System.out.println("get 1 tokens: " + r.acquire(1) + "s");
    System.out.println("get 1 tokens: " + r.acquire(1) + "s");
    System.out.println("get 1 tokens: " + r.acquire(1) + "s");
    System.out.println("end");
 /**
       * output:
       * get 5 tokens: 0.0s
       * get 1 tokens: 0.996766s 滞后效应，需要替前一个请求进行等待
       * get 1 tokens: 0.194007s
       * get 1 tokens: 0.196267s
       * end
       * get 5 tokens: 0.195756s
       * get 1 tokens: 0.995625s 滞后效应，需要替前一个请求进行等待
       * get 1 tokens: 0.194603s
       * get 1 tokens: 0.196866s
       */
 }
}
```

针对平滑预热限流有：

`RateLimiter`的 `SmoothWarmingUp`是带有预热期的平滑限流，它启动后会有一段预热期，逐步将分发频率提升到配置的速率。 比如下面代码中的例子，创建一个平均分发令牌速率为2，预热期为3分钟。由于设置了预热时间是3秒，令牌桶一开始并不会0.5秒发一个令牌，而是形成一个平滑线性下降的坡度，频率越来越高，在3秒钟之内达到原本设置的频率，以后就以固定的频率输出。这种功能适合系统刚启动需要一点时间来“热身”的场景。

```java
public void testSmoothwarmingUp() {
 RateLimiter r = RateLimiter.create(2, 3, TimeUnit.SECONDS);
 while (true)
 {
   System.out.println("get 1 tokens: " + r.acquire(1) + "s");
   System.out.println("get 1 tokens: " + r.acquire(1) + "s");
   System.out.println("get 1 tokens: " + r.acquire(1) + "s");
   System.out.println("get 1 tokens: " + r.acquire(1) + "s");
   System.out.println("end");
 /**
       * output:
       * get 1 tokens: 0.0s
       * get 1 tokens: 1.329289s
       * get 1 tokens: 0.994375s
       * get 1 tokens: 0.662888s  上边三次获取的时间相加正好为3秒
       * end
       * get 1 tokens: 0.49764s  正常速率0.5秒一个令牌
       * get 1 tokens: 0.497828s
       * get 1 tokens: 0.49449s
       * get 1 tokens: 0.497522s
       */
 }
}
```



其常用方法如下：

| 修饰符和类型       | 方法和描述                                                   |
| ------------------ | ------------------------------------------------------------ |
| double             | **acquire()** 从RateLimiter获取一个许可，该方法会被阻塞直到获取到请求 |
| double             | **acquire(int permits)**从RateLimiter获取指定许可数，该方法会被阻塞直到获取到请求 |
| static RateLimiter | create(double permitsPerSecond)根据指定的稳定吞吐率创建RateLimiter，这里的吞吐率是指每秒多少许可数（通常是指QPS，每秒多少查询） |
| static RateLimiter | **create(double permitsPerSecond, long warmupPeriod, TimeUnit unit)**根据指定的稳定吞吐率和预热期来创建RateLimiter，这里的吞吐率是指每秒多少许可数（通常是指QPS，每秒多少个请求量），在这段预热时间内，RateLimiter每秒分配的许可数会平稳地增长直到预热期结束时达到其最大速率。（只要存在足够请求数来使其饱和） |
| double             | **getRate()**返回RateLimiter 配置中的稳定速率，该速率单位是每秒多少许可数 |
| void               | **setRate(double permitsPerSecond)**更新RateLimite的稳定速率，参数permitsPerSecond 由构造RateLimiter的工厂方法提供。 |
| String             | toString()返回对象的字符表现形式                             |
| boolean            | **tryAcquire()**从RateLimiter 获取许可，如果该许可可以在无延迟下的情况下立即获取得到的话 |
| boolean            | **tryAcquire(int permits)**从RateLimiter 获取许可数，如果该许可数可以在无延迟下的情况下立即获取得到的话 |
| boolean            | **tryAcquire(int permits, long timeout, TimeUnit unit)**从RateLimiter 获取指定许可数如果该许可数可以在不超过timeout的时间内获取得到的话，或者如果无法在timeout 过期之前获取得到许可数的话，那么立即返回false （无需等待） |
| boolean            | **tryAcquire(long timeout, TimeUnit unit)**从RateLimiter 获取许可如果该许可可以在不超过timeout的时间内获取得到的话，或者如果无法在timeout 过期之前获取得到许可的话，那么立即返回false（无需等待） |



最后再来看看 SmoothBursty平滑突发限流的实现原理，在解析SmoothBursty原理前，先了解一下几个重要成员变量的含义：

```java
//当前存储令牌数
double storedPermits;
//最大存储令牌数
double maxPermits;
//添加令牌时间间隔
double stableIntervalMicros;
/**
 * 下一次请求可以获取令牌的起始时间
 * 由于RateLimiter允许预消费，上次请求预消费令牌后
 * 下次请求需要等待相应的时间到nextFreeTicketMicros时刻才可以获取令牌
 */
private long nextFreeTicketMicros = 0L;
```

https://zhuanlan.zhihu.com/p/60979444

特别注意RateLimiter是单机的，也就是说它无法跨JVM使用，设置的1000QPS，那也在单机中保证平均1000QPS的流量。假设集群中部署了10台服务器，想要保证集群1000QPS的接口调用量，那么RateLimiter就不适用了，集群流控最常见的方法是使用强大的Redis来实现。



### 分布式限流：Redisson-RRateLimiter

单节点模式下，使用RateLimiter进行限流一点问题都没有，但是线上是分布式系统，布署了多个节点，而且多个节点最终调用的是同一个短信服务商接口。虽然我们对单个节点能做到将 QPS 限制在400/s，但是多节点条件下，如果每个节点均是400/s，那么到服务商那边的总请求就是`节点数量×400/s`，于是限流效果失效。使用该方案对单节点的阈值控制是难以适应分布式环境的。

对打卡数据获取接口进行分布式限流的代码如下所示：

```java
public class RateLimiterUtils {
    private static ConcurrentHashMap<String, RRateLimiter> limiterMap = new ConcurrentHashMap<>();

    public static RRateLimiter getLimiter(String key) {
        RRateLimiter rateLimiter = limiterMap.get(key);
        if (Objects.isNull(limiterMap.get(key))) {
            synchronized (RateLimiterUtils.class) {
                if (Objects.isNull(limiterMap.get(key))) {
                    RedissonClient redissonClient = SpringContextUtil.getBean(RedissonClient.class);
                    rateLimiter = redissonClient.getRateLimiter(key);
                    limiterMap.put(key, rateLimiter);
                }
            }
        }
        return rateLimiter;
    }
}
public class AccessLimitService {

    @Resource
    private RedissonClient redissonClient;    
    
    public void larkAcquireByCorpId(String corpId) {

        RRateLimiter rRateLimiter = RateLimiterUtils.getLimiter("mobile:limiter:lark:" + corpId);
        rRateLimiter.trySetRate(RateType.OVERALL,100,1, RateIntervalUnit.SECONDS);
        rRateLimiter.acquire();
    }
    
    public void larkAbsenceAcquireByEnt(Long entId) {
        //打卡接口飞书单独定义接口权限每S 40个 两台机子 每个19 防止跑满
        RRateLimiter rRateLimiter = redissonClient.getRateLimiter("mobile:limiter:lark:absence" + entId);
        rRateLimiter.trySetRate(RateType.OVERALL,40,1, RateIntervalUnit.SECONDS);
        rRateLimiter.acquire();
    }
}








--------------------------------------------------   
 优化过后的 ，其实差别不大
--------------------------------------------------   

public class AccessLimitService {

    /**
     * 限流的BUFFER
     */
    private static final long LIMIT_BUFFER = 1;
    
    private static final long LIMITER_WAIT_TIME = 15;
    
    @Resource
    private RedisLimitTemplate redisLimitTemplate;
    
    public void larkAcquireByCorpId(String corpId) {
        String key = "mobile:limiter:lark:" + corpId;
        log.info("larkAcquireByCorpId:{} ", key);
        redisLimitTemplate.initRateLimiter(key, 100 - LIMIT_BUFFER, 1, RateIntervalUnit.SECONDS);
        boolean acquire = redisLimitTemplate.tryAcquire(key, 3, TimeUnit.SECONDS);
        if (!acquire) {
            throw new CustomException("请求被限流控制，请稍后重试");
        }
    }
    
        /**
     * 打卡接口飞书单独定义接口权限每S 40次
     */
    public void larkAbsenceAcquireByEnt(Long entId) {
        String key = "mobile:limiter:lark:absence" + entId;
        log.info("larkAbsenceAcquireByEnt:{} ", key);
        redisLimitTemplate.initRateLimiter(key, 40 - LIMIT_BUFFER, 1, RateIntervalUnit.SECONDS);
        boolean acquire = redisLimitTemplate.tryAcquire(key, 3, TimeUnit.SECONDS);
        if (!acquire) {
            throw new CustomException("请求被限流控制，请稍后重试");
        }
    }
```

RRateLimiter 接口的实现类几乎都在 RedissonRateLimiter 上，我们看看前面调用 RRateLimiter 方法时，这些方法的对应源码实现。接下来下面就着重讲讲这里限流算法中，一共用到的 3个 redis key。

> **1. key 1：Hash 结构**

就是前面 `setRate` 设置的 hash key。按照之前限流器命名“LIMITER_NAME”，这个 redis key 的名字就是 `LIMITER_NAME`。一共有3个值：

1. `rate`：代表速率
2. `interval`：代表多少时间内产生的令牌
3. `type`：代表单机还是集群

> **2. key 2：ZSET 结构**

ZSET 记录获取令牌的时间戳，用于时间对比，redis key 的名字是 `{LIMITER_NAME}:permits`。下面讲讲 ZSET 中每个元素的 member 和 score：

- `member`： 包含两个内容：(1)一段8位随机字符串，为了唯一标志性当次获取令牌；（2）数字，即当次获取令牌的数量。不过这些是压缩后存储在 redis 中的，在工具上看时会发现乱码。
- `score`：记录获取令牌的时间戳，如：1667025166312（对应 2022-10-29 14:32:46）

> **3. key 3： String 结构**

记录的是当前令牌桶中剩余的令牌数。redis key 的名字是 `{LIMITER_NAME}:value`。



RRateLimiter在初始化限流器时，会调用`trySetRate()`方法，直接来看它的源码：

![image-20240328214143475](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403282141589.png)

点进`trySetRateAsync()`方法：

![image-20240328214226420](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403282142555.png)

redis命令都是通过`CommandExecutor`来发送到redis服务执行的，可以看到它同时继承了 **同步和异步(sync/async)** 两种调用方式。

```java
public interface CommandExecutor extends CommandSyncExecutor, CommandAsyncExecutor {
}
```

在分布式锁的实现中是用了同步的 `CommandExecutor`，是因为锁的获取和释放是有强一致性要求的，需要实时知道结果方可进行下一步操作。这里就先不讲这些，后面讲分布式锁的时候再深入。。。。

`trySetRateAsync()`方法中核心是其中的 lua 脚本，摘出来看看：

```java
redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);
redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);
return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);
```

发现基于一个 hash 类型的redis key 设置了3个值，这里的命令是 `hsetnx`，该命令用于为哈希表中不存在的的字段赋值：

- 如果哈希表不存在，一个新的哈希表被创建并进行 `hset` 操作。
- 如果字段已经存在于哈希表中，操作无效。
- 如果 key 不存在，一个新哈希表被创建并执行 `hsetnx` 命令。

这意味着，这个方法只能做配置的初始化，如果后期想要修改配置参数，该方法并不会生效。我们来看看另外一个方法。

重新设置是，不管该key之前有没有用，一切都清空回到初始化，重新设置。对应实现类中的源码是`setRate()`方法：

![image-20240328220514612](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403282205731.png)

核心是其中的 lua 脚本，摘出来看看：

```lua
redis.call('hset', KEYS[1], 'rate', ARGV[1]);
redis.call('hset', KEYS[1], 'interval', ARGV[2]);
redis.call('hset', KEYS[1], 'type', ARGV[3]);
redis.call('del', KEYS[2], KEYS[3]);
```

上述的参数如下：

- `KEYS[1]`：hash key name
- `KEYS[2]`：string(value) key name
- `KEYS[3]`：zset(permits) key name
- `ARGV[1]`：rate
- `ARGV[2]`：interval
- `ARGV[3]`：type

通过这个 lua 的逻辑，就能看出直接用的是 `hset`，会直接重置配置参数，**并且同时会将已产生数据的string(value)、zset(permits) 两个key 删掉。是一个彻底的重置方法。**

这里回顾一下 `trySetRate` 和 `setRate`，在限流器不变的场景下，我们可以多次调用 `trySetRate`，但是不能调用 `setRate`。因为每调用一次，`redis.call('del', KEYS[2], KEYS[3])` 就会将限流器中数据清空，也就达不到限流功能。

----------



在初始化限流器完成后，开始获取令牌，前面是铺垫，下面就着重讲讲获取令牌的源码吧。

`acquire` 和 `tryAcquire` 均可用于获取指定数量的令牌，不过 `acquire` 会阻塞等待，而 `tryAcquire` 会等待 `timeout` 时间，如果仍然没有获得指定数量的令牌直接返回 `false`。

`tryAcquire`对应的方法是：

```java
private <T> RFuture<T> tryAcquireAsync(RedisCommand<T> command, Long value) {
        return commandExecutor.evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
                "local rate = redis.call('hget', KEYS[1], 'rate');"
              + "local interval = redis.call('hget', KEYS[1], 'interval');"
              + "local type = redis.call('hget', KEYS[1], 'type');"
              + "assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')"
              
              + "local valueName = KEYS[2];"
              + "local permitsName = KEYS[4];"
              + "if type == '1' then "
                  + "valueName = KEYS[3];"
                  + "permitsName = KEYS[5];"
              + "end;"

              + "assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); "

              + "local currentValue = redis.call('get', valueName); "
              + "if currentValue ~= false then "
                     + "local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); "
                     + "local released = 0; "
                     + "for i, v in ipairs(expiredValues) do "
                          + "local random, permits = struct.unpack('fI', v);"
                          + "released = released + permits;"
                     + "end; "

                     + "if released > 0 then "
                          + "redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); "
                          + "currentValue = tonumber(currentValue) + released; "
                          + "redis.call('set', valueName, currentValue);"
                     + "end;"

                     + "if tonumber(currentValue) < tonumber(ARGV[1]) then "
                         + "local nearest = redis.call('zrangebyscore', permitsName, '(' .. (tonumber(ARGV[2]) - interval), '+inf', 'withscores', 'limit', 0, 1); "
                         + "return tonumber(nearest[2]) - (tonumber(ARGV[2]) - interval);"
                     + "else "
                         + "redis.call('zadd', permitsName, ARGV[2], struct.pack('fI', ARGV[3], ARGV[1])); "
                         + "redis.call('decrby', valueName, ARGV[1]); "
                         + "return nil; "
                     + "end; "
              + "else "
                     + "redis.call('set', valueName, rate); "
                     + "redis.call('zadd', permitsName, ARGV[2], struct.pack('fI', ARGV[3], ARGV[1])); "
                     + "redis.call('decrby', valueName, ARGV[1]); "
                     + "return nil; "
              + "end;",
                Arrays.asList(getRawName(), getValueName(), getClientValueName(), getPermitsName(), getClientPermitsName()),
                value, System.currentTimeMillis(), ThreadLocalRandom.current().nextLong());
    }
```

> `evalWriteAsync` 和 `writeAsync`的区别是什么呢 :question:
>
> `evalWriteAsync` 和 `writeAsync` 是两个不同的方法，可能来自于特定的 Redis 客户端库或框架，其具体行为和区别取决于该库的设计和实现。一般情况下，它们可能具有以下区别：
>
> 1. 功能目的：`writeAsync` 方法通常用于执行常规的写操作，如设置键值对、执行单个命令等。它们直接发送指定的写命令到 Redis 服务器，并返回执行结果或响应。而 `evalWriteAsync` 方法则用于执行复杂的脚本或 Lua 表达式。它通过将脚本发送到 Redis 服务器并在服务器端执行来完成相关的操作。
> 2. 参数传递：`writeAsync` 方法一般会以明确的参数形式传递 Redis 命令和相应的参数，可以直接指定命令和参数的值。而 `evalWriteAsync` 方法通常接受 Lua 脚本作为参数，并允许通过占位符或参数列表的方式将参数传递给脚本。
> 3. 执行方式：`writeAsync` 方法通常是直接将命令发送到 Redis 服务器执行，并返回结果。它一般是单个命令的执行。而 `evalWriteAsync` 方法将 Lua 脚本发送到 Redis 服务器，然后在服务器端进行解析和执行。这使得可以在脚本中进行更复杂的逻辑和操作，甚至可以使用 Redis 的事务和 Lua 脚本语言的特性。

我们先看看执行 lua 脚本时，所有要传入的参数内容：

- `KEYS[1]`：hash key name
- `KEYS[2]`：全局 string(value) key name
- `KEYS[3]`：单机 string(value) key name
- `KEYS[4]`：全局 zset(permits) key name
- `KEYS[5]`：单机 zset(permits) key name
- `ARGV[1]`：当前请求令牌数量
- `ARGV[2]`：当前时间
- `ARGV[3]`：8位随机字符串

然后，我们再将其中的lua部分提取出来，我再根据自己的理解，在其中各段代码加上了注释。

```java
-- rate：间隔时间内产生令牌数量
-- interval：间隔时间
-- type：类型：0-全局限流；1-单机限流
local rate = redis.call('hget', KEYS[1], 'rate');
local interval = redis.call('hget', KEYS[1], 'interval');
local type = redis.call('hget', KEYS[1], 'type');

-- 如果3个参数存在空值，错误提示初始化未完成
assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')

local valueName = KEYS[2];
local permitsName = KEYS[4];

-- 如果是单机限流，在全局key后拼接上机器唯一标识字符
if type == '1' then
    valueName = KEYS[3];
    permitsName = KEYS[5];
end;

-- 如果：当前请求令牌数 < 窗口时间内令牌产生数量，错误提示请求令牌不能超过rate
assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate');

-- currentValue = 当前剩余令牌数量
local currentValue = redis.call('get', valueName);

-- 非第一次访问，存储剩余令牌数量的 string(value) key 存在，有值（包括 0）
if currentValue ~= false then
    -- 当前时间戳往前推一个间隔时间，属于时间窗口以外。时间窗口以外，签发过的令牌，都属于过期令牌，需要回收回来
    local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval);

    -- 统计可以回收的令牌数量
    local released = 0;
    for i, v in ipairs(expiredValues) do
        -- lua struct的pack/unpack方法，可以理解为文本压缩/解压缩方法
        local random, permits = struct.unpack('fI', v);
        released = released + permits;
    end;

    -- 移除 zset(permits) 中过期的令牌签发记录
    -- 将过期令牌回收回来，重新更新剩余令牌数量
    if released > 0 then
        redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval);
        currentValue = tonumber(currentValue) + released;
        redis.call('set', valueName, currentValue);
    end;

    -- 如果 剩余令牌数量 < 当前请求令牌数量，返回推测可以获得所需令牌数量的时间
    -- （1）最近一次签发令牌的释放时间 = 最近一次签发令牌的签发时间戳 + 间隔时间(interval)
    -- （2）推测可获得所需令牌数量的时间 = 最近一次签发令牌的释放时间 - 当前时间戳
    -- （3）"推测"可获得所需令牌数量的时间，"推测"，是因为不确定最近一次签发令牌数量释放后，加上到时候的剩余令牌数量，是否满足所需令牌数量
    if tonumber(currentValue) < tonumber(ARGV[1]) then
        local nearest = redis.call('zrangebyscore', permitsName, '(' .. (tonumber(ARGV[2]) - interval), '+inf', 'withscores', 'limit', 0, 1);
        return tonumber(nearest[2]) - (tonumber(ARGV[2]) - interval);
    -- 如果 剩余令牌数量 >= 当前请求令牌数量，可直接记录签发令牌，并从剩余令牌数量中减去当前签发令牌数量
    else
        redis.call('zadd', permitsName, ARG这段代码实现了一个基于Redis的分布式限流器。它使用了令牌桶算法来控制请求的速率。
代码中的`rate`表示在`interval`时间内产生的令牌数量，`type`表示限流的类型（全局限流或单机限流）。代码使用Redis的哈希表来存储限流器的配置信息，包括`rate`、`interval`和`type`。
```

代码的主要逻辑如下：

1. 首先，通过Redis的`hget`命令获取限流器的配置信息，如果配置信息为空，则抛出错误提示初始化未完成。
2. 然后，根据限流器的类型确定存储剩余令牌数量的键名和令牌桶存储的键名。
3. 如果是非第一次访问，即存储剩余令牌数量的键存在，获取当前剩余令牌数量。
4. 如果存在过期的令牌，即时间窗口以外的令牌，将其回收并更新剩余令牌数量。
5. 如果剩余令牌数量小于请求的令牌数量，返回预计可获得所需令牌数量的时间。
6. 如果剩余令牌数量大于等于请求的令牌数量，记录签发的令牌，并从剩余令牌数量中减去请求的令牌数量。

这段代码的作用是在分布式环境中实现请求的限流，确保请求的速率不超过预设的限制。它使用了Redis的哈希表和有序集合来存储和管理令牌桶的状态，并使用Lua脚本在原子操作中完成限流逻辑，保证了并发环境下的一致性和效率。





## 2、依照员工id分组向topic发送打卡数据，批量加锁落库



先来看看核心代码：

（1）mobile服务中向指定topic发送数据

```java
public class HcmAbsenceMsgProducer {

    @Value("${topic.hcm_abs_batch_sync_channel_record}")
    private String topic;
    
      /**
     * 推送打卡数据
     * 1.基于员工进行分组推送
     * 2.每个消息最多600条 ； 循环推送
     * 3.key 使用员工ID
     *
     * @param resList
     * @param entId
     */
    public void pushMsg(List<ChannelCheckInDataRes> resList, Long entId) {
        if (CollectionUtils.isEmpty(resList)) {
            return;
        }
         // 将打卡数据列表按员工号进行分组
        Map<Long, List<ChannelCheckInDataRes>> employeeDataMap = resList.stream().collect(Collectors.groupingBy(ChannelCheckInDataRes::getEmployeeId));
        for (Long employeeId : employeeDataMap.keySet()) {
            List<ChannelCheckInDataRes> employeeCheckInDataList = employeeDataMap.get(employeeId);
          
       //遍历每个员工的打卡数据，将数据按配置的每个消息的最大条数进行分割，生成多个消息   
        List<List<ChannelCheckInDataRes>> checkInLists = ListUtils.partition(employeeCheckInDataList, partitionRecord);
            
            for (List<ChannelCheckInDataRes> singleEmployeeDataList : checkInLists) {
                AbsClockCheckInDataDto absClockCheckInDataDto = new AbsClockCheckInDataDto();
                absClockCheckInDataDto.setEntId(entId);
                absClockCheckInDataDto.setBuId(singleEmployeeDataList.get(0).getBuId());
                absClockCheckInDataDto.setEmployeeId(employeeId);
                absClockCheckInDataDto.setCheckInDataResDtoList(singleEmployeeDataList);

                Message message = Message.builder().sendTime(new Date())
                        .msgId(employeeId)
                        .isSequential(false)
                        .topic(getTopic(entId))
                        .body(JacksonUtils.toJson(absClockCheckInDataDto))
                        .build();
                log.info("HcmAbsenceMsgProducer pushMsg -{}", JacksonUtils.toJson(message));
                defaultNoLogMsgProducer.pushMsg(message); //发送消息
            }
        }

    }

    /*
     * 根据企业ID、环境和灰度发布状态决定是否使用灰度发布的消息主题。如果需要切换到灰度主题，将消息主题的前缀修改为 "gray_"
     */
    public String getTopic(Long entId) {
        String newTopic = topic;
        boolean shouldChange2GrayTopic = GrayUtils.shouldChange2GrayTopic(entId, instanceProfileConfig.getEnv(), instanceProfileConfig.getGrayReleased());
        if (shouldChange2GrayTopic) {
            newTopic = "gray_" + topic;
            log.info("getTopic should be replace to new topic:{}, originTopic:{}", newTopic, topic);
        }
        return newTopic;
    }
}
```

这段代码是一个 Kafka 消息生产者类，用于向 Kafka 队列推送打卡数据消息。下面对代码中与 Kafka 相关的细节进行解释：

1. 首先，通过 `@Value("${topic.hcm_abs_batch_sync_channel_record}")` 注解将 Kafka 主题（topic）名称从配置文件中注入到 `topic` 字段中。这个字段表示要向哪个 Kafka 主题发送消息。
2. `pushMsg` 方法用于推送打卡数据消息到 Kafka 队列。它接收打卡数据列表 `resList` 和企业ID `entId` 作为参数。
3. 首先，检查 `resList` 是否为空，如果为空，则直接返回，不进行消息推送。
4. 接下来，将打卡数据列表按照员工号进行分组，使用 `resList.stream().collect(Collectors.groupingBy(ChannelCheckInDataRes::getEmployeeId))` 将数据按员工ID进行分组，得到一个 `Map`，键为员工ID，值为对应的打卡数据列表。
5. 针对每个员工，遍历其打卡数据列表。将数据按照配置的每个消息的最大条数进行分割，使用 `ListUtils.partition` 方法将数据列表分割为多个子列表，每个子列表的大小最多为 `partitionRecord`。
6. 针对每个子列表，创建一个 `AbsClockCheckInDataDto` 对象，将企业ID、员工ID、子列表中的打卡数据等信息设置到对象中。
7. 创建一个 Kafka 消息对象 `Message`，设置消息的发送时间、消息ID、是否按顺序发送、主题名称和消息体。消息体使用 `JacksonUtils.toJson` 方法将 `absClockCheckInDataDto` 对象转换为 JSON 字符串。
8. 最后，通过调用 `defaultNoLogMsgProducer.pushMsg(message)` 方法将消息推送到 Kafka 队列中。
9. `getTopic` 方法根据企业ID `entId` 决定使用哪个 Kafka 主题。根据一些条件判断，如果需要使用灰度发布的消息主题，会将主题名称的前缀修改为 "gray_"，然后返回新的主题名称。

总体而言，这段代码主要完成了将打卡数据按照员工ID进行分组，并将每个员工的打卡数据拆分为多个消息，并推送到 Kafka 队列中。通过控制每个消息的最大条数，可以控制消息的大小，确保消息在传输和处理过程中的合理性和效率。



> 关于顺序消费的问题：可以看出这里设置了 `isSequential(false)`，即消息不按顺序发送。实际上，是否需要保证消息的顺序消费是根据具体业务需求而定的，这里根据实际情况选择了不保证顺序发送。
>
> 通常情况下，确保消息的顺序消费是一个重要的考虑因素，特别是对于涉及到相关业务逻辑或依赖前后顺序的消息处理。然而，有时候在实际业务中，需要在追求顺序消费的同时，也需要一定的并发性能来提高系统的吞吐量。
>
> 需要注意的是，设置为不保证顺序发送并不意味着消息一定会乱序到达。在 Kafka 中，消息的分区和消费者的分组机制可以保证同一分区内的消息是有序的，只有跨多个分区的消息才可能存在乱序的情况。因此，在设计和配置 Kafka 主题、分区和消费者时，需要根据实际需求来选择适当的设置，以平衡性能和顺序性的要求。



（2）clock服务作为消息的消费者，监听kafka队列

```java
public class BatchSyncClockInRecordListener {
    
       @Resource
    private BatchSyncChannelRecordService batchSyncChannelRecordService;


    @KafkaListener(topics = {"${topic.hcm_abs_batch_sync_channel_record}"},
            groupId = "${group.hcm_abs_batch_sync_channel_record}",
            containerFactory = "batchConsumeListenContainerFactory")
    public void handleClockRecord(List<ConsumerRecord<String, String>> records, Acknowledgment ack) {

        ack.acknowledge();

        try {
            log.info("当前批次处理的消息: {}", records);
            Map<String, List<ChannelCheckInDataResDto>> employeeUniqkeyChannelRecordsMap = buildEmployeeUniqKeyChannelRecordsFromMsgs(records);

            batchSyncChannelRecordService.handleChannelRecords(employeeUniqkeyChannelRecordsMap);
        } catch (Exception e) {
            log.error("打卡记录同步消费失败 messages: {}", JacksonUtils.toJson(records), e);
        }
    }
}
```

这段代码是一个 Kafka 消费者的监听器类，用于监听 Kafka 主题中的消息并进行处理。下面是对代码的解释：

1. `@Component`: 标识该类为 Spring 组件，可以被自动扫描和注入到 Spring 容器中。
2. `@KafkaListener`: 用于定义 Kafka 消费者的监听器注解，指定了监听的主题、消费者组和消息监听容器工厂等信息。
3. `handleClockRecord` 方法: 作为 Kafka 消费者的消息处理方法，在接收到消息时被调用。
4. `ack.acknowledge()`: 调用 `Acknowledgment` 对象的 `acknowledge()` 方法，表示消息已被成功消费，向 Kafka 提交消费确认。
5. `buildEmployeeUniqKeyChannelRecordsFromMsgs` 方法: 用于解析消息内容，构建员工唯一键与打卡记录之间的映射关系。
6. 解析消息：通过遍历传入的 `records` 列表，逐个解析消息内容。首先将消息内容反序列化为 `Message` 对象，然后从 `Message` 对象中获取 `BatchSyncClockInRecordResDto` 对象，该对象包含了打卡记录的相关信息。
7. 构建员工唯一键与打卡记录的映射：根据 `BatchSyncClockInRecordResDto` 对象中的信息，构建租户、员工ID等参数，并生成一个表示员工唯一键的字符串 `employeeUniqKey`。然后根据 `employeeUniqKey` 从 `employeeUniqKeyChannelRecordsMap` 中获取对应的打卡记录列表，将当前解析的打卡记录添加到列表中，最后将列表放回到 `employeeUniqKeyChannelRecordsMap` 中。
8. 返回 `employeeUniqKeyChannelRecordsMap`：将构建好的员工唯一键与打卡记录的映射返回。

**这段代码的主要作用是将 Kafka 主题中的消息解析为打卡记录，并按照员工唯一键将这些记录进行分组。然后调用 `batchSyncChannelRecordService` 的 `handleChannelRecords` 方法来处理这些打卡记录。**

针对这个方法 ：**batchSyncChannelRecordService.handleChannelRecords(employeeUniqkeyChannelRecordsMap)**

```java
 public void handleChannelRecords(Map<String, List<ChannelCheckInDataResDto>> employeeUniqkeyChannelRecordsMap) {
 
  stopWatch.start("处理当前同步的数据以及状态");
        // 处理当前同步的数据以及状态
        handleChannelSyncTaskData(employeeUniqkeyChannelRecordsMap);
        stopWatch.stop();

        stopWatch.start("处理原始同步的打卡数据");
        // 处理同步的打卡数据
        Map<String, List<ClockInRecord>> employeeUniqKeyRecordsMap = Maps.newHashMap();
        for (String employeeUniqKey : employeeUniqkeyChannelRecordsMap.keySet()) {
            List<ChannelCheckInDataResDto> channelRecords = employeeUniqkeyChannelRecordsMap.get(employeeUniqKey);
            Long employeeId = ClockRecordUtil.parseEmployeeIdFromEmployeeUniqKey(employeeUniqKey);

            if (CollectionUtils.isEmpty(channelRecords)) {
                log.info("{}没有需要同步的数据", employeeId);
                continue;
            }
            String key = SYNC_CHANNEL_DATA_KEY_PREFIX + employeeId;
            RLock lock = redissonClient.getLock(key);
            try {
                boolean lockFlag = lock.tryLock(10, TimeUnit.SECONDS);
                if (!lockFlag) {
                    log.error("当前员工正在同步打卡数据: {}", employeeId);
                    continue;
                }

                Map<SyncTypeEnum, List<ChannelCheckInDataResDto>> channelRecordsMap = channelRecords.stream().collect(Collectors.groupingBy(ChannelCheckInDataResDto::getChannel));
                Map<SyncTypeEnum, ChannelOriginRecordService> channelServiceMap = syncClockInRecordFactory.getSyncChannelRecordsByChannels(channelRecordsMap.keySet());
                List<ClockInRecord> clockInRecords = Lists.newArrayList();
                for (SyncTypeEnum channel : channelServiceMap.keySet()) {
                    ChannelOriginRecordService originRecordService = channelServiceMap.get(channel);
                    if (Objects.isNull(originRecordService)) {
                        log.error("没有找到渠道对应的原始记录处理方式: {}, employeeUniqKey: {}", channel, employeeUniqKey);
                        continue;
                    }
                    // 处理原始数据
                    List<ClockInRecord> curChannelRecords = originRecordService.batchHandleOriginRecords(channelRecordsMap.getOrDefault(channel, Lists.newArrayList()));
                    clockInRecords.addAll(curChannelRecords);
                }
                employeeUniqKeyRecordsMap.put(employeeUniqKey, clockInRecords);
            } catch (Exception e) {
                log.error("{}员工打卡数据同步失败", employeeId, e);
            } finally {
                if (lock.isHeldByCurrentThread() && lock.isLocked()) {
                    lock.unlock();
                }
            }
        }
        stopWatch.stop();

        stopWatch.start("保存数据");
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
                List<ClockInRecord> totalRecords = employeeUniqKeyRecordsMap.values().stream().flatMap(Collection::stream).collect(Collectors.toList());
                batchSaveClockRecords(totalRecords);
            }
        });
        stopWatch.stop();

        stopWatch.start("推送消息");
        // 推送相关消息处理后续业务
        TransactionAsyncExecutor.asyncExecute(() -> postHandleResult(employeeUniqKeyRecordsMap));
        stopWatch.stop();

        log.info("数据同步耗时：{}", stopWatch.prettyPrint());
 
 }
```

注意这一行代码,它会**获取通道对应的原始打卡记录列表，这里获取的就是飞书**

```java
List<ClockInRecord> curChannelRecords = originRecordService.batchHandleOriginRecords(channelRecordsMap.getOrDefault(channel, Lists.newArrayList()));
```

![](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403290026097.png)

最后调用`larkService`下的`batchSave`方法，完成打卡记录的同步：

![image-20231123203142111](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403290026276.png)

对应的`mapper`文件为：

```xml
<insert id="batchInsert" keyColumn="id" keyProperty="id" parameterType="map" useGeneratedKeys="true">
  insert into hcm_abs_lark_origin_clock_record
  (ent_id, bu_id, user_id, corp_id, employee_id, detail, lark_record_id, check_time, batch_number)
  values
  <foreach collection="list" item="item" separator=",">
    (#{item.entId}, #{item.buId}, #{item.userId}, #{item.corpId}, #{item.employeeId},
    #{item.detail}, #{item.larkRecordId}, #{item.checkTime}, #{item.batchNumber})
  </foreach>
</insert>
```



数据库中的记录为：

![image-20231123203601172](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403290026890.png)

在 `handleChannelRecords` 方法中使用了 Redisson 提供的分布式锁这个方法的主要目的是处理批量同步的渠道打卡数据。在处理过程中，它需要访问和更新一些共享的数据结构，比如 `employeeUniqkeyChannelRecordsMap`，由于此方法可能会被多个线程同时调用，而且会涉及到共享数据的读写操作，因此需要使用分布式锁来确保线程安全性。而**handleChannelSyncTaskData 方法**并不直接涉及到共享数据的读写操作，而是根据输入的数据进行一些逻辑处理，并更新任务状态。由于其操作不涉及到共享数据的并发读写，因此不需要加锁。那么我们接下来就来看看Redisson分布式锁的底层原理！

### 分布式锁 Redisson-Lock

使用redisson实现分布式锁的代码十分简单，如下所示：

```java
RLock lock = redisson.getLock("anyLock");
lock.lock();
lock.unlock();
```

redisson具体的执行加锁逻辑都是通过lua脚本来完成的，lua脚本能够保证原子性。

先看下RLock初始化的代码：

```java
//Redisson.java
public class Redisson implements RedissonClient {

　　 //RLock是可重入锁的实现， redis命令都是通过CommandExecutor来发送到redis服务执行的
    @Override
    public RLock getLock(String name) {
    return new RedissonLock(connectionManager.getCommandExecutor(), name);
    }

}

--------
Redisson 中所有 Redis 命令都是通过 …Executor 执行的
获取到默认的同步执行器后, 就要初始化 RedissonLock
--------
    
//RedissonLock.java
public class RedissonLock extends RedissonBaseLock {
    public RedissonLock(CommandAsyncExecutor commandExecutor, String name) {
    // 父类的构造方法，最终是RedissonObject
    super(commandExecutor, name);
    // 初始化命令执行器
    this.commandExecutor = commandExecutor;
    // 初始化唯一ID，用于锁的前缀
    this.id = commandExecutor.getConnectionManager().getId();
    // 初始化锁的默认时间，这里采用的是看门狗的时间，默认30s
    this.internalLockLeaseTime = commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout();
    // 锁的标识
    this.entryName = id + ":" + name;
    // 初始化锁的监听器，用于在锁被占用时，使用Redis的pub/sub功能订阅锁释放消息
    this.pubSub = commandExecutor.getConnectionManager().getSubscribeService().getLockPubSub();
    }
}
```

#### 加锁 lock

然后我们来看一下 `RLock#lock()` 底层是如何获取锁的，加锁有lock和tryLock两类方法，区别在于tryLock方法会有一个等待时间，如果超过等待时间未获取到锁就会返回false，表示获取锁失败，而lock方法会一直等待，直到获取到锁，两者的源码相差不大，这里主要分析tryLock方法的源码。源码分析如下：

```java
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long time = unit.toMillis(waitTime);
    long current = System.currentTimeMillis();
    // 获取当前线程id
    long threadId = Thread.currentThread().getId();
    
    ------------
    // 获取锁，底层实现是lua脚本，具体参考下面的tryAcquireAsync源码（这部分包括lua脚本加锁和加锁时间续期）
    Long ttl = tryAcquire(leaseTime, unit, threadId);
    ------------
   
    
    // 锁获取成功，返回true
    if (ttl == null) {
        return true;
    }

    // 后面是锁获取失败的处理流程
     
 
    time -= System.currentTimeMillis() - current;
    // 等待时间用完，获取锁失败
    if (time <= 0) {
        acquireFailed(threadId);
        return false;
    }
     
    current = System.currentTimeMillis();
    // 订阅锁释放的通知
    RFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
    // 剩余时间内未订阅成功
    if (!subscribeFuture.await(time, TimeUnit.MILLISECONDS)) {
        // 尝试取消订阅过程，若无法取消，则会取消订阅
        if (!subscribeFuture.cancel(false)) {
            subscribeFuture.onComplete((res, e) -> {
                if (e == null) {
                    unsubscribe(subscribeFuture, threadId);
                }
            });
        }
        // 获取锁失败
        acquireFailed(threadId);
        return false;
    }

    try {
        time -= System.currentTimeMillis() - current;
        // 等待时间用完，加锁失败
        if (time <= 0) {
            acquireFailed(threadId);
            return false;
        }
     
        // 自旋获取锁
        while (true) {
            // 尝试获取锁
            long currentTime = System.currentTimeMillis();
            ttl = tryAcquire(leaseTime, unit, threadId);
            // 获取成功
            if (ttl == null) {
                return true;
            }
             
            // 未获取成功，若等待时间用完，则加锁失败
            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(threadId);
                return false;
            }

            // 通过java.util.concurrent包的Semaphore（信号量）挂起线程等待锁释放
            currentTime = System.currentTimeMillis();
            if (ttl >= 0 && ttl < time) {
                getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                getEntry(threadId).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
            }
             
            // 等待时间已用完，获取锁失败
            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(threadId);
                return false;
            }
        }
    } finally {
        // 取消订阅锁释放
        unsubscribe(subscribeFuture, threadId);
    }
//        return get(tryLockAsync(waitTime, leaseTime, unit));
}
private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, long threadId) {
    //  指定加锁时间
    if (leaseTime != -1) {
        // lua脚本加锁，详细见tryLockInnerAsync源码
        return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    }
    // 未指定加锁时间，则异步执行尝试获取锁，加锁时间是默认的30s
    RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
    ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
        if (e != null) {
            return;
        }
 
        // 获取锁成功，开启时间轮任务，用于锁续期
        if (ttlRemaining == null) {
            scheduleExpirationRenewal(threadId);
        }
    });
     
    return ttlRemainingFuture;
}
 
private void scheduleExpirationRenewal(long threadId) {
    ExpirationEntry entry = new ExpirationEntry();
    // 获取当前线程的锁续期任务节点
    ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
    if (oldEntry != null) {
        // 若当前线程的锁续期任务任务节点已存在，将节点计数加1
        oldEntry.addThreadId(threadId);
    } else {
        // 不存在，说明是新的线程获取锁
        // 将节点计数加1
        entry.addThreadId(threadId);
        // 开始时间轮调度任务
        // 其会每10s执行一次调度任务，给锁续期
        renewExpiration();
    }
}

```

这里可以看到**在不指定锁的过期时间时，Redisson会自动给锁续期（看门狗机制）**，锁每次重入都会将锁续期任务节点的计数加1，是为了在释放锁的时候判断释放几次后将锁续期任务停止。下面看一下锁存入redis的lua脚本：

```lua
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);
    // KEYS[1]为 Collections.<Object>singletonList(getName())的第一个元素，即锁名
    // ARGV[1]是internalLockLeaseTime，即为过期时间
    // ARGV[2]是getLockName(threadId)，为锁的唯一标识
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
               // 判断锁是否存在
              "if (redis.call('exists', KEYS[1]) == 0) then " +
                  // 锁不存在，通过hash结果进行存储，hash-key是前缀+线程id，value是1，表示第一次加锁
                  "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                  // 设置过期时间
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  // 返回null，加锁成功
                  "return nil; " +
              "end; " +
              // 锁已存在，判断是否是当前线程获取到锁
              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                  // 是当前线程获取到锁，将hash的value加1，表示锁的进入次数加1
                  "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                  // 设置过期时间
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  // 返回null，加锁成功
                  "return nil; " +
              "end; " +
              // 返回锁的过期时间，表示获取锁失败
              "return redis.call('pttl', KEYS[1]);",
                Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
}

```

通过`tryLockInnerAsync`的源码不难分析`Redisson`锁的存储结构，`Redisson`锁采用的是`hash`结构，`Redis` 的`Key`为锁名，也就是初始化时传入的`name`参数，**`hash`的`key`为锁的id属性(uuid)+线程id**，`hash`的`value`为加锁次数，不难看出`Redisson`分布式锁是可重入的。

这段 Lua 脚本是用于在 Redis 中执行的，主要用于实现 Redisson 分布式锁的尝试获取操作。让我逐步解释每个参数的含义：

- `KEYS[1]`：锁的名称，由 `getName()` 方法返回。这是分布式锁在 Redis 中的存储键（key）。
- `ARGV[1]`：锁的过期时间（以毫秒为单位）。在 Lua 脚本中，`ARGV` 表示命令的参数数组，而在这里，`ARGV[1]` 是 `internalLockLeaseTime`，即指定的锁的持续时间。
- `ARGV[2]`：锁的唯一标识。在这里，它是由 `getLockName(threadId)` 方法生成的，用于标识获取锁的线程。
- `redis.call('exists', KEYS[1])`：检查键 `KEYS[1]` 是否存在。如果锁不存在，说明当前没有其他线程持有该锁，可以尝试获取锁。
- `redis.call('hset', KEYS[1], ARGV[2], 1)`：如果锁不存在，则通过哈希结构存储锁的信息。这里将哈希的字段名设置为 `ARGV[2]`，即锁的唯一标识，值为 1，表示第一次加锁。
- `redis.call('pexpire', KEYS[1], ARGV[1])`：设置锁的过期时间。这里使用了 `pexpire` 命令，以毫秒为单位设置锁的过期时间。
- `redis.call('hexists', KEYS[1], ARGV[2])`：检查哈希结构中是否存在指定字段（即锁的唯一标识）。如果该字段存在，表示当前线程已经持有了锁。
- `redis.call('hincrby', KEYS[1], ARGV[2], 1)`：如果锁已经存在且是当前线程持有，将哈希结构中指定字段的值增加 1。这是为了支持锁的重入功能，即同一个线程多次获取同一个锁。
- `redis.call('pttl', KEYS[1])`：获取锁的剩余过期时间（以毫秒为单位）。如果获取到锁失败，Lua 脚本返回锁的剩余过期时间，否则返回 nil 表示获取锁成功。

总的来说，这段 Lua 脚本的作用是尝试获取 Redisson 分布式锁，并且支持了锁的重入功能。根据不同的情况，它会返回不同的结果：如果获取锁成功，返回 nil；如果获取锁失败，返回剩余过期时间。

再来加深一下理解:

> 假设前面获取锁时传的name是“abc”，假设调用的线程ID是Thread-1，假设成员变量UUID类型的id是6f0829ed-bfd3-4e6f-bba3-6f3d66cd176c
>
> 那么KEYS[1]=abc，ARGV[2]=6f0829ed-bfd3-4e6f-bba3-6f3d66cd176c:Thread-1
>
> 因此，这段脚本的意思是:
>
> 　　1、判断有没有一个叫“abc”的key
>
> 　　2、如果没有，则在其下设置一个字段为“6f0829ed-bfd3-4e6f-bba3-6f3d66cd176c:Thread-1”，值为“1”的键值对 ，并设置它的过期时间
>
> 　　3、如果存在，则进一步判断“6f0829ed-bfd3-4e6f-bba3-6f3d66cd176c:Thread-1”是否存在，若存在，则其值加1，并重新设置过期时间
>
> 　　4、返回“abc”的生存时间（毫秒）
>
> 这里用的数据结构是hash，hash的结构是： key 字段1 值1 字段2 值2 。。。
>
> 用在锁这个场景下，key就表示锁的名称，也可以理解为临界资源，字段就表示当前获得锁的线程
>
> 所有竞争这把锁的线程都要判断在这个key下有没有自己线程的字段，如果没有则不能获得锁，如果有，则相当于重入，字段值加1（次数）
>
> 

综上所述，加锁的流程如下：

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403291927857.png)



#### 看门狗 watchdog

再来回顾一下加锁的核心操作：

![f2a17e07363807624568a75d21fd396](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403292008574.png)

**当leaseTime == -1时，才会启动看门狗机制**，它的主要步骤如下：

- 在获取锁的时候，不能指定leaseTime或者只能将leaseTime设置为-1，这样才能开启看门狗机制。
- 在tryLockInnerAsync方法里尝试获取锁，如果获取锁成功调用scheduleExpirationRenewal执行看门狗机制
- 在scheduleExpirationRenewal中比较重要的方法就是renewExpiration，当线程第一次获取到锁（也就是不是重入的情况），那么就会调用renewExpiration方法开启看门狗机制。
- 在renewExpiration会为当前锁添加一个延迟任务task，这个延迟任务会在10s后执行，执行的任务就是将锁的有效期刷新为30s（这是看门狗机制的默认锁释放时间）
- 并且在任务最后还会继续递归调用renewExpiration。

总的流程就是，首先获取到锁（这个锁30s后自动释放），然后对锁设置一个延迟任务（10s后执行），延迟任务给锁的释放时间刷新为30s，并且还为锁再设置一个相同的延迟任务（10s后执行），这样就达到了如果一直不释放锁（程序没有执行完）的话，看门狗机制会每10s将锁的自动释放时间刷新为30s。

而当程序出现异常，那么看门狗机制就不会继续递归调用renewExpiration，这样锁会在30s后自动释放。或者，在程序主动释放锁后，流程如下：

- 将锁对应的线程ID移除
- 接着从锁中获取出延迟任务，将延迟任务取消
- 在将这把锁从EXPIRATION_RENEWAL_MAP中移除。
  



#### 解锁 unlock

再来看看解锁的流程，锁一般在finally代码块里执行unlock方法，核心方法是unlockAsync，其源码如下：

```java
@Override
public RFuture<Void> unlockAsync(long threadId) {
    RPromise<Void> result = new RedissonPromise<Void>();
    // 释放锁
    RFuture<Boolean> future = unlockInnerAsync(threadId);
 
    future.onComplete((opStatus, e) -> {
        if (e != null) {
            // 释放异常，取消锁续期任务
            cancelExpirationRenewal(threadId);
            result.tryFailure(e);
            return;
        }
         
        // 返回为null，表示该线程未拥有该锁，抛出异常
        if (opStatus == null) {
            IllegalMonitorStateException cause = new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                    + id + " thread-id: " + threadId);
            result.tryFailure(cause);
            return;
        }
         
        // 取消锁续期任务
        cancelExpirationRenewal(threadId);
        result.trySuccess(null);
    });
 
    return result;
}
 
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    //keys[1]]为 Arrays.<Object>asList(getName(), getChannelName())的第一个元素，即锁名
    //keys[2]为Arrays.<Object>asList(getName(), getChannelName())的第二个元素，为锁释放通知的通道，格式为：redisson_lock__channel:{lockName}
    //ARGV[1]为LockPubSub.UNLOCK_MESSAGE，是发布锁释放事件类型
    //ARGV[2]为internalLockLeaseTime，即过期时间
    //ARGV[3]为getLockName(threadId)，是锁的唯一标识
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                // 锁不存在，返回null
                "return nil;" +
            "end; " +
            // 加锁次数减1
            "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
            "if (counter > 0) then " +
                // 剩余次数大于0
                // 设置过期时间，时间为30s
                "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                // 返回0
                "return 0; " +
            "else " +
                // 剩余次数小于等于0
                // 删除缓存
                "redis.call('del', KEYS[1]); " +
                // 发布锁释放通知
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                // 返回1
                "return 1; "+
            "end; " +
            "return nil;",
            Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));
}
```

流程图如下：

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/amonstercat/PicGo@master/202403291956972.png)





### 整长型累加器（LongAdder）

