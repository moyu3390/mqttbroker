# mqttBroker
基于netty springboot实现的mqtt 服务端
1.遗嘱消息
2.保留消息
3.消息质量 0/1/2
4.集群部署

加入自定义的自动订阅配置，如果只是设备和broker通信可以不用订阅
自定义管理协议

针对标准的mqtt协议有2点还没有实现
1.topic的通配符过滤，mqtt协议允许客户端订阅 topic 使用#/+$ 符号来过滤主题
2.在服务端发布qos 为1或者2 的消息的时候，在未收到 客户端响应时 需要重试
    解决：目前发布qos 为1或者2消息时 存redis。可以基于redis 过期通知的机制完成 对客户端的 超时重发
    
# mqtt概述
MQTT是一个客户端服务端架构的发布/订阅模式的消息传输协议。它的设计思想是轻巧、开放、简单、规范，易于实现。这些特点使得它对很多场景来说都是很好的选择，特别是对于受限的环境如机器与机器的通信（M2M）以及物联网环境（IoT）。
# 主要特性
MQTT协议工作在低带宽、不可靠的网络的远程传感器和控制设备通讯而设计的协议，它具有以下主要的几项特性：
- 使用发布/订阅消息模式，提供一对多的消息发布，解除应用程序耦合。
- 使用TCP/IP提供网络连接。
- 有三种消息发布服务质量：
*    "至多一次"，  消息发布完全依赖底层TCP/IP网络。会发生消息丢失或重复。这一级别可用于如下情况，环境传感器数据，丢失一次读记录无所谓，因为不久后还会有第二次发送。这一种方式主要普通APP的推送，倘若你的智能设备在消息推送时
                                  未联    网推送过去没收到，再次联网也就收不到了。
*    "至少一次"，确保消息到达，但消息重复可能会发生。
*    "只有一次"，确保消息到达一次。在一些要求比较严格的计费系统中，可以使用此级别。在计费系统中，消息重复或丢失会导致不正确的结果。这种最高质量的消息发布服务还可以用于即时通讯类的APP的推送，确保用户收到且只会处理一次。
- 小型传输，开销很小（固定长度的头部是2字节），协议交换最小化，以降低网络流量。           
           这就是为什么在介绍里说它非常适合"在物联网领域，传感器与服务器的通信，信息的收集"，要知道嵌入式设备的运算能力和带宽都相对薄弱，使用这种协议来传递消息再适合不过了。
- 使用Last Will特性通知有关各方客户端异常中断的机制。
          * Last Will：即遗言机制，用于通知同一主题下的其他设备发送遗言的设备已经断开了连接。
- 保留消息，每一个topic可以存储一个消息在服务端，当有其他客户端订阅时，发送该消息给这个客户端
# 协议原理
## 实现方式
实现MQTT协议需要客户端和服务器端通讯完成，在通讯过程中，MQTT协议中有三种身份：发布者（Publish）、代理（Broker）（服务器）、订阅者（Subscribe）。其中，消息的发布者和订阅者都是客户端，消息代理是服务器，消息发布者可以同时是订阅者。
MQTT传输的消息分为：主题（Topic）和负载（payload）两部分：
- topic，可以理解为消息的类型，订阅者订阅（Subscribe）后，就会收到该主题的消息内容（payload）；
- payload，可以理解为消息的内容，是指订阅者具体要使用的内容。

## 网络传输
MQTT会构建底层网络传输：它将建立客户端到服务器的连接，提供两者之间的一个有序的、无损的、基于字节流的双向传输。
当应用数据通过MQTT网络发送时，MQTT会把与之相关的服务质量（QoS）和主题名（Topic）相关连。
### mqtt客户端
一个使用MQTT协议的应用程序或者设备，它总是建立到服务器的网络连接。客户端可以：
- 订阅其它客户端发布的消息；
- 退订其它客户端发布的消息；
- 断开与服务器连接。
### mqtt服务器
MQTT服务器以称为"消息代理"（Broker），可以是一个应用程序或一台设备。它是位于消息发布者和订阅者之间，它可以：
- 接受来自客户的网络连接；
- 接受客户发布的应用信息；
- 处理来自客户端的订阅和退订请求；
- 向订阅的客户转发应用程序消息。
## mqtt协议术语 
- 一、订阅（Subscription）
订阅包含主题筛选器（Topic Filter）和最大服务质量（QoS）。订阅会与一个会话（Session）关联。一个会话可以包含多个订阅。每一个会话中的每个订阅都有一个不同的主题筛选器。
- 二、会话（Session）
每个客户端与服务器建立连接后就是一个会话，客户端和服务器之间有状态交互。会话存在于一个网络之间，也可能在客户端和服务器之间跨越多个连续的网络连接。
- 三、主题名（Topic Name）
连接到一个应用程序消息的标签，该标签与服务器的订阅相匹配。服务器会将消息发送给订阅所匹配标签的每个客户端。
- 四、主题筛选器（Topic Filter）
一个对主题名通配符筛选器，在订阅表达式中使用，表示订阅所匹配到的多个主题。
- 五、负载（Payload）
消息订阅者所具体接收的内容。
## mqtt协议数据包结构
在MQTT协议中，一个MQTT数据包由：固定头（Fixed header）、可变头（Variable header）、消息体（payload）三部分构成。MQTT数据包结构如下：
- 固定头（Fixed header）。存在于所有MQTT数据包中，表示数据包类型及数据包的分组类标识。
- 可变头（Variable header）。存在于部分MQTT数据包中，数据包类型决定了可变头是否存在及其具体内容。
- 消息体（Payload）。存在于部分MQTT数据包中，表示客户端收到的具体内容。
## 消息质量
- qos=0：最多一次
发送者
发送消息时qos=0，dup=0
接收者
无
解释：
为什么叫最多一次？因为这个消息，发送者发完后，自己也不能确定接收者一定是收到了，并且如果这个消息丢了，接收者再也不会收到这个消息了，如果不丢，接收者也只能获取到一次这个消息 所以叫最多一次。
开销最少，最不可靠
- qos=1:最少一次
发送者：
为消息分配一个唯一标示id（2个字节）放入可变头
发送消息时qos=1,dup=0
等待收到一个pubask 类型消息的应答
如果在一定时间没有收到应答 就重新发送该消息 此时dup=1
接受者：
收到消息后 返回一个pubask
解释：
为什么叫至少一次，因为消息在一定时间没有得到确认的应答会重新发送，就有可能接受方已经收到了但是 应答丢了或者回复的应答慢了一点 发送方又会重复发送该消息，导致这个消息至少会被发送一次
开销增加，有重复消费的风险
- qos=2:只有一次
发送者：
为消息分配一个唯一标示id（2个字节）放入可变头
发送消息时qos=2,dup=0
等待收到一个pubrec 类型消息的应答
如果在一定时间没有收到应答 就重新发送该消息 此时dup=1
收到pubrec后 返回一个pubrel报文，包含同一个表示id ，不在重试发送消息
接受者
接受到消息后返回一个pubrec，该报文包含消息的唯一id
等待收到一个pubrel的保报文
如果一定时间没有收到pubrel报文，则重发pubrec
收到pubrel报文后不在重发pubrec
解释：
为什么叫只有一次，因为在q o s=1 的基础之上又多了一个应答的 应答，在q o s=1 中因为发送发有可能接收不到确认会重复发送，因此qos=2中对应答在进行一次确认，确保发送发不会在重复发了才完成。接收方在第一次收到这个消息的时候会保存消息唯一id 当在收到重复的消息，不会处理，因此该消息只会被处理一次
