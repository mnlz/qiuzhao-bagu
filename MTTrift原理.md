# MTThrift原理


1. 概述
1.1 Thrift 架构
1.2 通信整体流程
2. 数据封装

2.1 请求数据封装
2.1.1 包头
包头组成
TMessage 序列化
2.1.2 请求参数
请求参数结构
请求参数的序列化
2.1.3 包尾
2.2 统一 OCTO 协议
3. IO 模型
4. 线程模型
4.1 传统线程模型
4.2 Reactor 模型
4.2.1 单线程 Reactor 模型
4.2.2 多线程 Reactor 模型
4.2.3 主从多线程 Reactor 模型
4.3 MTThrift 线程模型
5. case 分析
6. 参考
7. 附件
技术团队博客/.../『每日一粒』苦练基本功-.../第二篇 MTThrift 通信的实现

## 1.概述

### 1.1 Thrift 架构

MTThrift 使用 Thrift 框架实现，Thrift 包含一个完整的堆栈结构用于构建客户端和服务器端，下图描绘了 Thrift 的整体架构。


各模块说明如下：

底层IO模块：负责实际的数据传输，包括Socket，文件，或者压缩数据流等；

TTransport：负责以字节流方式发送和接收 Message，是底层 IO 模块在 Thrift 框架中的实现，每一个底层IO模块都会有一个对应 TTransport 来负责 Thrift 的字节流( Byte Stream )数据在该 IO 模块上的传输。例如 TSocket 对应 Socket 传输，TFileTransport 对应文件传输；

TProtocol：将一个有类型的数据转化为字节流以交给 TTransport 进行传输，或者从 TTransport 中读取一定长度的字节数据转化为特定类型的数据。如 int32 会被 TBinaryProtocol Encode 为一个四字节的字节数据，或者 TBinaryProtocol 从 TTransport 中取出四个字节的数据 Decode 为 int32 ；

TServiceClient：负责定义请求数据结构，发送 Client 的请求；

TServer：负责接收 Client 的请求，并将请求转发到 Processor 进行处理；

TProcessor：负责对 Client 的请求做出相应，包括 RPC 请求转发，调用参数解析和用户逻辑调用，返回值写回等处理步骤。Processor 是服务器端从 Thrift 框架转入用户逻辑的关键流程。Processor 同时也负责向 Message 结构中写入数据或者读出数据。 

### 1.2 通信整体流程

假设客户端调用：

```java
helloService.sayHello("world"); 
```

服务端的实现为：

```java
public class HelloServiceImpl implements HelloService.Iface {
    public String sayHello(String username) throws TException {
        System.out.println(username);
        return "hello, " + username;
    }
}
```

客户端向服务端发送请求的通信过程如下：


（1）客户端调用 helloService 的 sayHello 方法，交给 TServiceClient 处理；

（2）TServiceClient 定义包含请求信息的数据结构，包括函数名、请求类型、seqid、参数类型、参数值等数据；

（3）TProtocol 将 TServiceClient 定义的请求数据结构序列化为二进制数，交给 TTransport；

（4）TTransport 对 TProtocol 层传来的数据再次封装，比如增加分割字段防止黏包，增加业务特殊字段，如 trance 信息等；

（5）使用高效的 IO 模型和线程模型，将 TTransport 层封装的数据发送给服务端。

服务端接收数据是客户端的逆过程，不再累述。

通过上面的过程可以看出，通信过程主要的核心点是数据封装、IO 模型和线程模型，本文从这三个角度展开介绍。

## 2.数据封装

系统间数据通信需按统一的封装协议，否则无法正常解析和处理接收的数据。以客户端请求为例，主要有两层封装：

**请求数据封装：定义请求的方法名、参数类型、参数值的结构，并使用序列化协议将请求数据结构序列化成二进制数，以便于在互联网中传输；**

MTThrift 仅支持 thrift 序列化，后面的序列化也仅介绍 thrift 的序列化过程

统一 OCTO 协议：增加包长、tranceInfo、全链路消息上下文、校验码等信息，便于拆包、校验、链路监测等。

服务端和客户端的处理使用同一种协议，数据解码是数据编码的逆过程，本节以客户端请求数据为例进行分析。

### 2.1 请求数据封装

本节内容依赖 IDL 相关知识，可参考：Thrift IDL 开发规范 和 2.MTthrift 开发指南

请求数据封装主要是为了保证本次请求的相关信息，包括调用类型、方法名、参数类型、参数值等数据，其结构定义在 TServiceClient 中：

- 包头：包头由版本号、方法名和请求序号组成

- 请求参数结构：请求参数和值封装的结构体

- 包尾：未实现，待扩展

```java
Java
    protected void sendBase(String methodName, TBase args) throws TException {
        this.oprot_.writeMessageBegin(new TMessage(methodName, (byte)1, ++this.seqid_));// 写 message 头
        args.write(this.oprot_); // 写参数
        this.oprot_.writeMessageEnd();
        this.oprot_.getTransport().flush();
    }

```

#### 2.1.1 包头

包头组成 4B 4B length B 4B version name length name seqid 版本号 方法名长度 方法名 请求序号

包头由 TMessage 组成，由3个数据组成：

- version：由 type 生成，type 包括4种类型：

```java
    public static final byte CALL = 1;// 请求
    public static final byte REPLY = 2;// 响应
    public static final byte EXCEPTION = 3;// 异常
    public static final byte ONEWAY = 4;// 单向请求
```

- name：本次请求的方法名

- seqid：请求序号

TMessage 序列化
MTThrift 序列化使用 MtraceClientTBinaryProtocol，继承自 TBinaryProtocol，使用的是 thrift 二进制序列化协议，TMessage 的序列化实现过程如下：

```java
    public void writeMessageBegin(TMessage message) throws TException {
        if (this.strictWrite_) {
            int version = -2147418112 | message.type;// 版本号由 type 生成
            this.writeI32(version);
            this.writeString(message.name);// 方法名
            this.writeI32(message.seqid);// 请求序号
        } else {
            this.writeString(message.name);
            this.writeByte(message.type);
            this.writeI32(message.seqid);
        }
    }
```



#### 2.1.2 请求参数

请求参数结构
主要目的是封装请求的数据类型，是一个 Struct 结构，Struct 由多个 field 组成。请求参数的实现由 IDL 编译器自动生成，参数结构如下：

- 结构（Struct）头：默认未实现

- 参数（field）：

  - 参数头：由3个部分组成

  - name：参数名

  - type：参数类型，包括 IDL 支持的所有类型

```java
public final class TType {
    public static final byte STOP = 0;
    public static final byte VOID = 1;
    public static final byte BOOL = 2;
    public static final byte BYTE = 3;
    public static final byte DOUBLE = 4;
    public static final byte I16 = 6;
    public static final byte I32 = 8;
    public static final byte I64 = 10;
    public static final byte STRING = 11;
    public static final byte STRUCT = 12;
    public static final byte MAP = 13;
    public static final byte SET = 14;
    public static final byte LIST = 15;
    public static final byte ENUM = 16;
    public TType() {
  	}
 }
```


id：字段在函数中定义的顺序，从1开始

参数值：具体的 String、int、Map 等值，使用序列化协议进行序列化（MTThrift 仅支持 thrift 序列化协议），下一节会详细介绍序列化过程

参数尾：默认未实现

参数结束符：1个字节，固定为0

结构（Struct）尾：默认未实现

以 sayHello("world") 为例（源码详见文末附件），其调用的是 IDL 自动生成的 sayHello_args#write 方法：

```java
    public void write(org.apache.thrift.protocol.TProtocol oprot) throws org.apache.thrift.TException {
      schemes.get(oprot.getScheme()).getScheme().write(oprot, this);
    }
```

最后调用的是 sayHello_argsStandardScheme#write 方法：

```java
public void write(org.apache.thrift.protocol.TProtocol oprot, sayHello_args struct) throws org.apache.thrift.TException {
        struct.validate();
        oprot.writeStructBegin(STRUCT_DESC);
          if (struct.username != null) {
            oprot.writeFieldBegin(USERNAME_FIELD_DESC);
            oprot.writeString(struct.username);
            oprot.writeFieldEnd();
          }
          oprot.writeFieldStop();
          oprot.writeStructEnd();
        }
}
```



请求参数的序列化
从上一小节可以看出，请求参数结构主要由 struct 和 field 组成，filed 又由 IDL 规定的基础类型组成。

struct 的序列化：一个struct就是由多个field编码而成，最后一个field排列完成之后是一个stop field，这个field是一个8bit全为0的字节，它标志着一条Thrift消息的结束；上一节 sayHello_argsStandardScheme#write 中即是 struct 对应的结构

field 的序列化：长度为 1+2+X，filed不是一个实际存在的类型，而是一个抽象概念。field不独立出现，而是出现在struct内部，其中field-type可以是任何其他的类型，field-id就是定义IDL时该field在struct的编号，field-value是对应类型的值的序列化结果（参见基础类型序列化）


参数开始、参数结束和参数结束符的实现如下：

代码块
Java
// 写入 field-type 和 field-id    
public void writeFieldBegin(TField field) throws TException {
        this.writeByte(field.type);// 字段类型
        this.writeI16(field.id);// 字段 id，字段在函数中定义的顺序，从1开始
    }

    public void writeFieldEnd() { // 默认未实现，MTThrift 未扩展
    }
    
    public void writeFieldStop() throws TException {
        this.writeByte((byte)0);// 参数结束符固定为0
    }
基础类型序列化

定长编码： bool, byte, short, int, long, double采用的都是固定字节数编码，各类型占用的字节数如下：

类型名

idl类型名

占用字节数

类型ID

bool

bool

1

2

byte

byte

1

3

short

i16

2

6

int

i32

4

8

long

i64

8

10

double

double

8

4

 举例说明如下：

代码块
Java
// bool 实现
    public void writeBool(boolean b) throws TException {
        this.writeByte((byte)(b ? 1 : 0));
    }

// byte 实现
    public void writeByte(byte b) throws TException {
        this.bout[0] = b;
        this.trans_.write(this.bout, 0, 1);
    }

// i16 实现
    public void writeI16(short i16) throws TException {
        this.i16out[0] = (byte)(255 & i16 >> 8);
        this.i16out[1] = (byte)(255 & i16);
        this.trans_.write(this.i16out, 0, 2);
    }

// 其他详见 TBinaryProtocol 中的实现
长度前缀编码：长度为4+N，string, byte array 采用的是长度前缀编码，前四个字节(无符号四字节整数)表示长度，后面跟着的就是实际的内容


代码块
Java
    public void writeString(String str) throws TException {
        try {
            byte[] dat = str.getBytes("UTF-8");
            this.writeI32(dat.length);
            this.trans_.write(dat, 0, dat.length);
        } catch (UnsupportedEncodingException var3) {
            throw new TException("JVM DOES NOT SUPPORT UTF-8");
        }
    }
map的编码：长度为1+1+4+NX+NY


list和set的编码：长度为1+4+N*X


2.1.3 包尾
默认未实现。

thrift 序列化在性能和空间开销上的表现都非常好，适用于高负载、高并发的场景：https://tech.meituan.com/2015/02/26/serialization-vs-deserialization.html

2.2 统一 OCTO 协议
主要是为了调用链追踪、数据统计等，并保证协议稳定可扩展。

1B

1B

1B

1B

4B

2B

header length

total length - 2B - header lentgh (-4B)

4B(可选)

0xAB

0xBA

version

protocol

total length

header length

header

body

checksum

包头

消息头

消息体

校验码

0xAB 和 0xBA ：固定字段

version：目前固定为1

protocol：按位分割，含义如下：

7

6

5

4

3

2

1

0

含义

是否校验

压缩类型

预留

序列化类型

作用

0，不需要校验

1，需要检验

00，不压缩

01，采用Snappy压缩

10，采用Gzip压缩

0

0

0

只支持thrift序列化，值为1

HeadInfo：一些标识信息，如消息类型、压缩类型、traceInfo、请求信息、请求类型等，详细可参考：服务框架统一协议(OCTO协议)设计

代码块
Java
enum MessageType { // 消息类型
    Normal = 0,             // 正常消息
    NormalHeartbeat = 1,    // 端对端心跳消息
    ScannerHeartbeat = 2    //scanner心跳消息
}

enum CompressType { // 压缩类型 
    None = 0,        // 不压缩 
    Snappy = 1,      // Snappy 
    Gzip = 2         // Gzip 
}

struct RequestInfo {                // 请求信息 
    1: required string serviceName; // 服务名 
    2: required i64 sequenceId;     // 消息序列号 
    3: required byte callType;      // 调用类型  
    4: required i32 timeout;        // 请求超时时间 
}

enum CallType {   // 调用类型 
    Reply = 0,     // 需要响应 
    NoReply = 1,   // 不需要响应 
}

struct ResponseInfo {           // 响应信息 
    1: required i64 sequenceId; // 消息序列号 
    2: required byte status;    // 消息返回状态
    3: optional string message; //异常消息
}

enum StatusCode{  
    Success = 0,              // 成功  
    ApplicationException = 1, // 业务异常，业务接口方法定义抛出的异常  
    RuntimeException = 2,     // 运行时异常，一般由业务抛出 
    RpcException = 3,         // 框架异常，包含没有被下列异常覆盖到的框架异常 
    TransportException = 4,   // 传输异常 
    ProtocolException = 5,    // 协议异常 
    DegradeException = 6,     // 降级异常 
    SecurityException = 7,    // 安全异常 
    ServiceException = 8,     // 服务异常，如服务端找不到对应的服务或方法 
    RemoteException = 9,      // 远程异常 
}

当前基于thrift异常序列化实现，除了ApplicationException异常是由用户定义的exception类进行序列化，其他的所有异常的都通过TApplicationException类进行序列化。
struct TraceInfo {                        // Mtrace 跟踪信息，原 MTthrift 中的 RequestHeader 
    1: required string clientAppkey;      // 客户端应用名 
    2: optional string traceId;           // Mtrace 的 traceId 
    3: optional string spanId;            // Mtrace 的 spanId 
    4: optional string rootMessageId;     // Cat 的 rootMessageId 
    5: optional string currentMessageId;  // Cat 的 currentMessageId 
    6: optional string serverMessageId;   // Cat 的 serverMessageId 
    7: optional bool debug;               // 是否强制采样 
    8: optional bool sample;              // 是否采样 
    9: optional string clientIp;              // 客户端IP
}


struct LoadInfo{
     1: optional double averageLoad;
     2: optional i32 oldGC;
     3: optional i32 threadNum;                      //默认线程池
     4: optional i32 queueSize;                      //主IO线程队列长度
     5: optional map<string, double> methodQpsMap;   //key为ServiceName.methodName，value为1分钟内的对应的qps值（key为all，value则为所有方法的qps）
}

struct HeartbeatInfo {
    1:  optional string appkey;      // 解决重复注册，修改错误appkey状态的问题
    2:  optional i64 sendTime;       // 发送心跳时间，微秒，方便业务剔除历史心跳
    3:  optional LoadInfo loadInfo;  // 负载信息
    4:  required i32 status;         // 0：DEAD（未启动）， 2：ALIVE（正常），4：STOPPED（禁用）
}

typedef map<string, string> Context // 消息上下文，用于传递自定义数据

struct Header {                                                // 消息头
    1: optional byte messageType = MessageType.Normal;         // 消息类型，必填
    2: optional RequestInfo requestInfo;                       // 请求信息，必填
    3: optional ResponseInfo responseInfo;                     // 响应信息
    4: optional TraceInfo traceInfo;                           // 跟踪信息
    5: optional Context globalContext;                         // 全链路消息上下文，总大小不超过 512 Bytes
    6: optional Context localContext;                          // 单次消息上下文，总大小不超过 2K Bytes
    7: optional HeartbeatInfo heartbeatInfo;                   // 心跳信息
}
3. IO 模型
操作系统提供的网络 IO 模型主要分为阻塞 IO、非阻塞 IO、IO 复用、信号驱动 IO 和异步 IO（代码安全第一篇 - Java NIO）。Java 提供了阻塞 IO、非阻塞 IO 和异步 IO 的实现，三者对比如下：

类型

概念

同步/异步

阻塞/非阻塞

client:I/O线程数

使用难度

适用场景

使用模式

BIO

Blocking IO

阻塞IO

同步、阻塞

1:1

或M:N (线程池)

简单

在活动连接数不是特别高（小于单机1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的 I/O 并且编程模型简单，也不用过多考虑系统的过载、限流等问题。线程池本身就是一个天然的漏斗，可以缓冲一些系统处理不了的连接或请求。但是，当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。因此，我们需要一种更高效的 I/O 处理模型来应对更高的并发量。

Thrift IO

NIO

Non-Blocking IO

非阻塞IO

同步、非阻塞

M:1

复杂

对于高负载、高并发的（网络）应用，应使用 NIO 的非阻塞模式来开发。

Netty IO

AIO

Asynchronous IO

异步IO

异步、非阻塞

M:0

复杂

使用未成熟

未支持

目前 MTThrift 提供 thrift IO（BIO）和 Netty IO（NIO）两种方式，官方推荐使用 Netty IO 的方式。在使用时，只需将 Netty IO 设置为 true 即可。

代码块
Java
    <bean id="clientProxy" class="com.meituan.service.mobile.mtthrift.proxy.ThriftClientProxy" destroy-method="destroy">
        <property name="serviceInterface" value="com.meituan.mtthrift.test.HelloService"/> <!-- 接口名 -->
        <property name="appKey" value="com.sankuai.inf.mtthrift.testClient"/>  <!-- 本地appkey -->
        <property name="remoteAppkey" value="com.sankuai.inf.mtthrift.testServer"/>  <!-- 目标 Server Appkey  -->
        <property name="nettyIO" value="true"/><!-- 开启 Netty IO  -->
        <property name="timeout" value="1000"/>
    </bean>
在使用 Netty IO 时有几点需要注意（MTthrift Netty IO 说明）：

调用端使用 Netty IO 需要将 MTThrift 升级到 1.8.4.2版本

在调用端使用Netty IO需要服务端的支持，即需要服务端 MTThrift 版本在1.8.0及以上

MTThrift 在实际运行时会根据服务端 MTThrift 版本进行检查，判断是否使用 Netty IO，如果发现服务端 MTThrift 版本较低会自动关闭 Netty IO 选项，使用 Thrift IO

如果调用端采用直连的方式（即设置了localServerPort或serverIpPorts属性），除了要将 Netty IO属性设置为true，还需要将remoteUniProto属性设置为true

如果是多端口服务不要使用Netty 4.1.26.Final、4.1.27.Final、4.1.28.Final版本，会引发对堆内存溢出，4.1.29.Final已修复（<=4.1.25.Final没有问题）https://github.com/netty/netty/issues/8288

如果服务端是whale，则需要 MTThrift 升级到1.8.7.5+版本

MTThrift 在使用 Netty 提供的 nio 时做了优化，如果操作系统支持 epoll 就使用 EpollServerSocketChannel，否则使用 NioServerSocketChannel，epoll 模式可以提升系统性能（https://my.oschina.net/u/992559/blog/1787817）。

代码块
Java
// Epoll.isAvailable() 由 netty 提供的判断方法
if (Epoll.isAvailable()) {
  // 系统支持 epoll 模式
    bossGroup = new EpollEventLoopGroup(IO_BOSS_THREADS, new DefaultThreadFactory("MtthriftServerBossGroup"));
    workerGroup = new EpollEventLoopGroup(selectorThreads, new DefaultThreadFactory("MtthriftServerWorkerGroup"));
} else {
  // 系统不支持 epoll 模式
    bossGroup = new NioEventLoopGroup(IO_BOSS_THREADS, new DefaultThreadFactory("MtthriftServerBossGroup"));
    workerGroup = new NioEventLoopGroup(selectorThreads, new DefaultThreadFactory("MtthriftServerWorkerGroup"));
}

ServerBootstrap b = new ServerBootstrap();
   b.group(bossGroup, workerGroup)
            // 如果操作系统支持 epoll 模式，使用 EpollServerSocketChannel，否则使用 NioServerSocketChannel
           .channel(workerGroup instanceof EpollEventLoopGroup ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
           .childHandler(new NettyServerInitiator(serviceProcessorMap, tprocessor, this, (int) this.maxReadBufferBytes))
           .option(ChannelOption.SO_BACKLOG, this.backlog)
           .option(ChannelOption.SO_REUSEADDR, true)
           .childOption(ChannelOption.SO_KEEPALIVE, true)
           .childOption(ChannelOption.TCP_NODELAY, true);
4. 线程模型
4.1 传统线程模型
一个服务器在处理网络请求时，每一个handler都会开启一个线程（或者使用线程池）来处理请求。


传统线程模型实现简单，但每个请求都由一个单独的线程处理，线程数将会是支撑高并发的瓶颈。

4.2 Reactor 模型
4.2.1 单线程 Reactor 模型
由一个线程来接收客户端的连接，并将该请求分发到对应的事件处理 handler 中，整个过程完全是异步非阻塞的，并且完全不存在线程上线文切换带来的问题。


由于是一个线程，对多核 CPU 利用率不高，一旦有大量的客户端连接上来性能必然下降，甚至会有大量请求无法响应。最坏的情况是一旦这个线程哪里没有处理好进入了死循环那整个服务都将不可用！

4.2.2 多线程 Reactor 模型
Rector 多线程模型与单线程模型最大的区别就是有一组 NIO 线程处理 IO 操作：


其实最大的改进就是将原有的事件处理改为了多线程。可以基于 Java 自身的线程池实现，这样在大量请求的处理上性能提升是巨大的。

虽然如此，但理论上来说依然有一个地方是单点的，如例如并发百万客户端连接，或者服务端需要对客户端握手进行安全认证，但是认证本身非常损耗性能。

4.2.3 主从多线程 Reactor 模型
该模型将客户端连接处理和 IO 读写分离出来，同时也是多个子线程来处理事件响应，这样无论是连接、读写、事件处理都是高性能的。


4.3 MTThrift 线程模型
MTThrift 目前主要使用的是 Netty IO，我们这里主要分析 Netty 的线程模型。

Netty是一款高效的NIO框架和工具，基于JAVA NIO提供的API实现。在JAVA NIO方面Selector给Reactor模式提供了基础，Netty结合Selector和Reactor模式设计了高效的线程模型。


（1）client 连接请求过来，从 Boss NioEventLoopGroup 中选择一个 EventLoop，建立连接；

（2）建立连接成功后，把 channel 交给 Worker NioEventLoopGroup 其中一个 EventLoop 处理处理后续的读写操作，之后这个客户端的 channel 的生命周期都由这个 EventLoop 管理，Boss NioEventLoop 继续处理其他的连接请求；

（3）client 的读写请求发送到步骤（2）注册的 EventLoop 处理。

MTThrift 中的配置

服务端：

boss 默认为 4

worker 默认为 selectorThreads 配置逻辑如下：

代码块
Java
    private void calculateIOThreadIfNoConfig() {
        if (selectorThreads != -1) {
            return;
        }
        if (serviceProcessorMap == null || serviceProcessorMap.isEmpty() || serviceProcessorMap.size() == 1) {
            // 单端口单服务，默认为 4
            selectorThreads = Consts.DEFAULT_MIN_IO_WORKER_THREAD_COUNT;
        } else {
            try {
                int multiServiceCount = serviceProcessorMap.size();
                // cpu * 2, 最大32
                int maxIOThreads = Math.min(Runtime.getRuntime().availableProcessors() * 2, 32);
                // 单个端口服务树达到10个则默认使用最大线程数
                if (multiServiceCount >= 10) {
                    selectorThreads = maxIOThreads;
                } else {
                    Map<MathUtil.LinearFactor, Double> linearFactor = MathUtil.calculateLinearFactor(1, Consts.DEFAULT_MIN_IO_WORKER_THREAD_COUNT, 10, maxIOThreads);
                    long calThreadCount = Math.round(multiServiceCount * linearFactor.get(MathUtil.LinearFactor.K) + linearFactor.get(MathUtil.LinearFactor.B));
                    if (calThreadCount > maxIOThreads) {
                        selectorThreads = maxIOThreads;
                    } else if (calThreadCount < Consts.DEFAULT_MIN_IO_WORKER_THREAD_COUNT) {
                        selectorThreads = Consts.DEFAULT_MIN_IO_WORKER_THREAD_COUNT;
                    } else {
                        selectorThreads = (int) calThreadCount;
                    }
                }
            } catch (Throwable e) {
                selectorThreads = Consts.DEFAULT_MIN_IO_WORKER_THREAD_COUNT;
                logger.warn("Calculate selectorThreads failed, selectorThreads set to default={}", Consts.DEFAULT_MIN_IO_WORKER_THREAD_COUNT, e);
            }
        }
    }
工作线程：默认最小为 10，最大为 256

客户端：

连接池默认为 1

worker 默认为 CPU 数 * 2

代码块
Java
 DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));

protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
    }
5. case 分析
RejectedExecutionException排查

分析：服务端线程被占满，且等待队列也被占满，导致剩下的客户端请求被拒绝。潜在的解决方法：

（1）适当增加服务端线程池大小；

（2）服务线程池隔离，对某些方法单独设置线程池：MTthrift 单端口多服务使用说明

配送调度骑手路径规划任务计算服务东方set故障，影响配送运单调度

分析：主要由两个原因导致：

（1）客户端使用的是 thrift io，每个请求会独占连接到直到请求返回才释放，在满足 MTthrift Netty IO 说明 中条件的情况下，建议使用 netty io，会复用连接

（2）请求数据量较大，每次请求达到 2M，可在调用端开启压缩（2.MTthrift 开发指南 中的 gzip 或 snappy 压缩方式）

6. 参考
https://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/index.html

https://tech.meituan.com/2015/02/26/serialization-vs-deserialization.html

代码安全第一篇 Java NIO

https://www.jianshu.com/p/df1d6d8c3f9d

Scalable IO in Java 个人理解

https://blog.csdn.net/lijinqi1987/article/details/77896140

https://www.cnblogs.com/zaizhoumo/p/8206591.html

https://andrewpqc.github.io/2019/02/24/thrift/

https://www.infoq.cn/article/netty-threading-model

http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf

7. 附件
仅供内部使用，未经授权，切勿外传
评论(1)
浏览 11553 次  共 3890 人浏览

王春生(Spring Bourne) 06-07 08:26
里面贴的Mtthrift的源码，注明是哪个类就更好了。


写点你要说的