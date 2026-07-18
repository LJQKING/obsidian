MQTT 是 Android 中**物联网（IoT）开发、智能家居、车联网、即时通信（IM）、消息推送**最常见的协议之一。

如果你准备 **Android 中高级（3~5年）面试**，不仅要知道如何使用 MQTT，更需要理解：

- MQTT 协议原理
    
- QoS 工作机制
    
- KeepAlive 心跳
    
- Retain、Will Message
    
- Android 如何集成 MQTT
    
- Eclipse Paho 源码
    
- MQTT 收发流程
    
- 与 WebSocket、HTTP、Socket 对比
    

下面按照**原理 → Android 实现 → 源码**来讲。

---

# 一、MQTT 是什么

MQTT（Message Queuing Telemetry Transport）

是一种

> 基于 TCP 的轻量级发布/订阅（Publish/Subscribe）消息协议。

特点：

```
Client
      \
       \
        Broker
       /
      /
Client
```

它不像 HTTP：

```
Client ---> Server
```

而是：

```
Client <-----> Broker <-----> Client
```

Broker 就是消息服务器。

例如：

```
Android

      |
 Publish

      |

 Mosquitto

      |

 Subscribe

      |

 另一台Android
```

Broker 常见：

- EMQX
    
- Mosquitto
    
- HiveMQ
    
- RabbitMQ(MQTT Plugin)
    

---

# 二、为什么 MQTT 很省流量

HTTP：

```
GET

↓

Headers

↓

Body

↓

Response
```

HTTP Header 非常大。

例如：

```
User-Agent

Accept

Cookie

Host

Authorization

...
```

可能几百字节。

MQTT：

```
固定头

可变头

Payload
```

最小只有：

```
2 Byte
```

所以：

```
HTTP：

100 Byte Header

MQTT：

2 Byte Header
```

非常适合：

- 车联网
    
- 智能家居
    
- 传感器
    
- Android IoT
    

---

# 三、MQTT 工作流程

例如：

```
Android A

↓

Publish

↓

Broker

↓

Android B

↓

Receive
```

完整流程：

```
Connect

↓

Subscribe Topic

↓

Publish

↓

Broker 转发

↓

Disconnect
```

---

# 四、Publish / Subscribe 原理

例如聊天室：

```
Topic:

chat/room1
```

A：

```
Publish

chat/room1

Hello
```

Broker：

```
Topic:

chat/room1

↓

找到所有订阅者
```

B：

```
Subscribe

chat/room1
```

收到：

```
Hello
```

所以：

```
发布者

↓

不知道谁接收

↓

Broker负责路由
```

这就是：

```
解耦
```

---

# 五、Topic 原理

Topic：

```
home/light

home/temp

car/speed

car/location
```

支持层级：

```
/

例如：

china/beijing/car1
```

支持：

```
+

#
```

例如：

```
home/+/temp
```

匹配：

```
home/a/temp

home/b/temp
```

```
#

home/#
```

匹配：

```
home/a

home/a/b

home/a/b/c
```

---

# 六、QoS（重点）

MQTT 面试最高频。

MQTT 有：

```
QoS0

QoS1

QoS2
```

---

## QoS0

```
最多一次

At most once
```

流程：

```
Publish

↓

结束
```

不确认。

可能丢。

适合：

```
GPS

天气

温度
```

---

## QoS1

```
至少一次

At least once
```

流程：

```
Client

↓

Publish

↓

Broker

↓

PUBACK

↓

结束
```

如果：

```
没收到 PUBACK
```

再次发送：

```
Publish
```

所以：

```
可能重复
```

例如：

```
支付通知
```

需要：

```
去重
```

---

## QoS2

最高等级：

```
Exactly Once
```

流程：

```
PUBLISH

↓

PUBREC

↓

PUBREL

↓

PUBCOMP
```

四次握手。

保证：

```
只收到一次
```

---

# 七、Keep Alive

MQTT 长连接。

如果：

```
10分钟

没有数据
```

怎么知道连接还活着？

MQTT：

```
PINGREQ

↓

PINGRESP
```

例如：

```
60秒
```

发送：

```
PINGREQ
```

Broker：

```
PINGRESP
```

Android：

源码：

```
AlarmManager

↓

Wakeup

↓

Ping
```

新版：

```
JobScheduler

WorkManager
```

---

# 八、Retain Message

例如：

```
Topic:

temperature
```

发送：

```
30℃
```

设置：

```
retain=true
```

以后：

```
新的客户端

Subscribe
```

立即收到：

```
30℃
```

Broker 保存最后一条。

---

# 九、Will Message

客户端异常退出：

```
手机断电

Crash

网络断开
```

Broker 自动发送：

```
offline
```

例如：

```
user/online

↓

offline
```

聊天室：

```
XXX 已离线
```

---

# 十、Android 集成 MQTT

最常用：

```
implementation "org.eclipse.paho:org.eclipse.paho.client.mqttv3"
implementation "org.eclipse.paho:org.eclipse.paho.android.service"
```

初始化：

```kotlin
val client = MqttAndroidClient(
    context,
    "tcp://broker.emqx.io:1883",
    "android_client"
)
```

连接：

```kotlin
client.connect()
```

订阅：

```kotlin
client.subscribe("chat",1)
```

发送：

```kotlin
client.publish(
    "chat",
    MqttMessage("Hello".toByteArray())
)
```

---

# 十一、源码分析（Paho）

Android：

```
MqttAndroidClient
```

↓

```
MqttAsyncClient
```

↓

```
ClientComms
```

↓

```
ClientState
```

↓

```
NetworkModule
```

↓

```
Socket
```

---

## 第一层

```
MqttAndroidClient
```

负责：

- Android 生命周期
    
- Service
    
- BroadcastReceiver
    

发送：

```java
publish(...)
```

↓

实际上：

```java
myClient.publish(...)
```

myClient：

```
MqttAsyncClient
```

---

## 第二层

```
MqttAsyncClient
```

真正客户端。

里面：

```java
ClientComms comms;
```

发送：

```java
comms.sendNoWait(...)
```

---

## 第三层

ClientComms

源码：

```java
sendNoWait()
```

里面：

```java
clientState.send()
```

负责：

```
发送消息

维护状态

Reconnect
```

---

## 第四层

ClientState

这里最重要。

维护：

```
QoS

Inflight

Token

MessageId
```

例如：

```
QoS1
```

发送：

```
Pending

↓

Wait PUBACK

↓

Complete
```

这里维护：

```
Hashtable
```

保存：

```
正在发送
```

---

## 第五层

CommsSender

线程：

```
while(true){

write(socket)
}
```

不断：

```
取队列

↓

发送Socket
```

源码：

```
run()
```

里面：

```java
clientState.get()
```

↓

```java
out.write(...)
```

---

## 第六层

CommsReceiver

线程：

```
while(true){

read(socket)
}
```

收到：

```
Publish

PubAck

PingResp
```

交给：

```
ClientState
```

处理。

---

## 第七层

ClientState

收到：

```
PUBACK
```

更新：

```
Inflight--
```

通知：

```
Delivery Complete
```

---

# 十二、发送流程源码

```
publish()

↓

MqttAndroidClient

↓

MqttAsyncClient

↓

ClientComms

↓

ClientState

↓

CommsSender

↓

Socket OutputStream

↓

TCP

↓

Broker
```

---

# 十三、接收流程源码

```
Broker

↓

Socket

↓

CommsReceiver

↓

ClientState

↓

MqttCallback

↓

messageArrived()
```

Android：

```kotlin
override fun messageArrived(
    topic: String?,
    message: MqttMessage?
)
```

最终回调：

```
UI
```

---

# 十四、线程模型

```
Main Thread

↓

MqttAndroidClient

↓

CommsSender(Thread)

↓

Socket
```

还有：

```
CommsReceiver(Thread)
```

以及：

```
Callback Thread
```

整个模型：

```
          UI

           |

MqttAndroidClient

      |

ClientComms

 /      |      \

Sender Receiver Callback
```

---

# 十五、MQTT 与 HTTP、WebSocket、TCP 对比

|特性|MQTT|HTTP|WebSocket|TCP Socket|
|---|---|---|---|---|
|底层协议|TCP|TCP|TCP|TCP|
|通信方式|发布/订阅|请求/响应|全双工|全双工|
|长连接|是|默认否（HTTP/1.1 可 Keep-Alive）|是|是|
|消息头|极小（最小约 2 字节）|较大|较小|无应用层协议头|
|服务端推送|原生支持|不支持（需 SSE、轮询等）|支持|支持|
|QoS|内置 QoS0/1/2|无|无|无|
|典型场景|IoT、IM、车联网|REST API|聊天、实时协作|自定义协议|

---

# 十六、Android 面试高频问题

1. **MQTT 为什么适合物联网？**
    
    - 轻量级协议，报文头小，带宽占用低。
        
    - 长连接减少频繁握手开销。
        
    - 发布/订阅模型降低客户端耦合。
        
    - 内置 QoS、遗嘱消息（Will）、保留消息（Retain）等机制。
        
2. **QoS1 为什么会出现重复消息？**
    
    - 发送方未收到 `PUBACK` 时会重发 `PUBLISH`，如果第一次实际上已到达 Broker，就可能导致接收方收到重复消息，因此业务侧通常需要通过消息 ID 或业务唯一标识做幂等处理。
        
3. **MQTT 如何保持长连接？**
    
    - 客户端按 Keep Alive 周期发送 `PINGREQ`，Broker 回复 `PINGRESP`。若连续超时未收到响应，客户端会认为连接断开并触发重连。
        
4. **Paho 中负责真正 Socket 收发的是哪些类？**
    
    - `CommsSender`：负责从发送队列读取报文并写入 `Socket`。
        
    - `CommsReceiver`：负责监听 `Socket` 输入流并解析 MQTT 报文。
        
    - `ClientState`：负责消息状态管理、QoS 流程、消息重发及确认。
        
5. **为什么 MQTT 比 HTTP 更适合实时通信？**
    
    - MQTT 使用长连接，消息可由 Broker 主动推送；HTTP 通常采用请求/响应模式，需要轮询或其他扩展机制才能实现近实时推送，因此 MQTT 在低延迟和网络开销方面更有优势。
        

---

## 总结

掌握 MQTT 建议按照以下路线深入学习：

1. **协议层**：报文格式（固定头、可变头、Payload）、Topic、QoS、Retain、Will。
    
2. **网络层**：基于 TCP 的长连接、Keep Alive、心跳与断线重连机制。
    
3. **Android 实现**：`MqttAndroidClient`、生命周期管理、线程模型、消息回调。
    
4. **源码层（Paho）**：重点理解 `MqttAsyncClient → ClientComms → ClientState → CommsSender/CommsReceiver` 的调用链，以及 QoS1/QoS2 的状态流转和消息重发逻辑。
    
5. **项目实践**：结合 Kotlin 协程、Flow、MVI/MVVM、自动重连、离线缓存、消息幂等处理等，实现稳定可靠的实时消息系统。
    

对于 Android 中高级岗位，能够解释 **MQTT 协议原理 + Paho 源码架构 + QoS 状态机 + Android 长连接管理**，通常已经能够覆盖这一知识点的大部分面试深度。