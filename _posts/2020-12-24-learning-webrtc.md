---
layout: post
title:  "从Chrome源码学习Webrtc"
date:   2020-12-24 14:23:48 +0800
categories: webrtc
---
### 什么是webrtc

webrtc的全程是 **web real-time communications** ，从名字中我们可以知道webrtc是提供网络实时通信的，但是我们并不知道它究竟是什么。

其实，webrtc就是通过集合一系列技术以提供上述功能，形成的一套技术标准。

对于普通的用户而言，webrtc就是一个个 **API** ，能够方便的调用，轻松实现高质量网络实时通信，包括：
- 本地媒体捕获
- 对等连接 多端通信
- 媒体流
- 数据流
- 媒体广播
- ...

而对于我们程序员而言就需要了解webrtc中各种技术，以实现对于各种场景的优化

### 起点：webrtc对等连接

#### 1. 对等连接之SDP
SDP, Session Description Protocol, 是一种会议(session)协商协议

webrtc使用SDP协商 确保会议各方都支持要传输的数据类型

SDP的作用就是统一格式，统一描述，这样各端就能用统一的处理方式，方便协商。

SDP的格式是 多行UTF8文字，每行都是 `单个字母+'='+'具体值/描述'`，详见 [rfc4556](https://tools.ietf.org/html/rfc4566)

比如 rfc 中提供的SDP例子：
```txt
v=0
o=jdoe 2890844526 2890842807 IN IP4 10.47.16.5
s=SDP Seminar
i=A Seminar on the session description protocol
u=http://www.example.com/seminars/sdp.pdf
e=j.doe@example.com (Jane Doe)
c=IN IP4 224.2.17.12/127
t=2873397496 2873404696
a=recvonly
m=audio 49170 RTP/AVP 0
m=video 51372 RTP/AVP 99
a=rtpmap:99 h263-1998/90000
```
其中 
1. v行 => version. 即SDP的协议版本
1. s行 => session. 是session块的开始<br>
    session块会一直延续到第一个media块，这之间的行都是描述session的属性，比如

    ```txt
    * i行 表示 session info
    * u行 表示 uri
    * c行 表示 connection
    * t行 表示 time 即开始session的时间
    ```
1. m行 => media. 是media块的开始<br>
    media块会一直延续到下一个media块，这之间的行都是描述该media块的属性

    同时media行会提供一些传输相关信息，具体格式如下<br>
     `m=<media> <port>/<number of ports> <proto> <fmt> ... `<br>

    |---|---|
    |media|视频还是音频|
    |port|传输端口|
    |number of ports|端口可以选择一个范围|
    |proto|传输协议|
    |fmt|在不同proto情况下表示不同的涵义，具体见rfc|

1. a行 => attribute. 即各种属性及其具体描述。

    a行如果在session块中就描述session的属性，在media块中就描述media的属性

    同时sesion块中的属性可以看成所有media块的默认属性，除非media块中有定义这个属性

    所以<br>
    `a=recvonly` 表示后面的video和audio都是只收不发的<br>
    `a=rtpmap:99 h263-1998/90000` 只表示video的编码格式是h263

#### 2. 对等连接之ICE
webrtc设计之初就支持端对端的即时通信，要实现端对端（p2p）的通信，关键在创建端对端的直接连接。

由于大部分端没有可用的公网ip，处于层层NAT和防火墙的后面，因此需要一种技术能够找到端与端之间的通路，而不需要走中心服务器，以降低成本提高速度。

webrtc采用的技术即 ICE **Interactive Connectivity Establishment** 详细技术标准见 [rfc8445](https://tools.ietf.org/html/rfc8445)

简单概括如下：
1. signal server 创建初始连接，用于信令交换，candidate交换等
2. 用户收集所有的 **address-pair(candidate)**, 包括本地host的，也包括向 stun server 查询的NAT的公网ip-port
1. 用户尝试匹配双方的 **address-pair**，如果可以匹配成功，就尝试创建连接
3. 如果连接失败，则退化使用turn server中继，如果turn也不存在则认为连接失败
4. signal server, stun server, turn server 可以为同一个server

创建连接之后，signal server的连接就可以放弃了，此时双方是直接互相通信的，可以极大程度的节约服务器的资源

ICE中并不指定signal server的协议，因为signal server创建初始连接，依赖于具体业务的判断，所以协议也是用户自行决定

#### 3. webrtc 创建p2p流程简介
webrtc 为了建立连接，需要
1. 生成sdp信息，总结本地支持的数据格式，传输方式
1. 交换sdp信息，以协商后续媒体数据的格式，传输方式等

还需要
1. 获取所有的ice candidate，包括本地(host), stun反馈的NAT的公网地址(reflex), turn server中继地址(relay)
1. 交换ice candidate信息，以建立ice连接

其中ice candidate信息可以放在sdp信息中，也可以使用单独的信令交换

但是一般来说：sdp offer生成是很快的，而部分candidate的获取需要向服务器请求，时间消耗未知，

如果candidate信息放入sdp offer中，则需要等待所有的candidate收集完成，

但是实际应用场景中，是存在host层的candidate就可以建立连接，没必要请求reflex candidate的情况的

此时等待所有candidate收集，就会有额外的等待时间。

因此，chrome 生成的sdp中是不带candidate的，而是额外提供了
1. `onicecandidate` event回调，来通知新candidate获取
2. `onicegatheringstatechange` event回调，通知所有的candidate获取完毕
3. 且sdp可以被用户修改然后通过 `setLocalDescription` 设置

方便用户自行决定，是否要等待所有的candidate

#### 4. chrome 创建p2p对等连接——代码分析

##### sdp交换-js代码
首先我们从webrtc的用户角度出发，看一下js的调用逻辑，以下示例详见[文档](https://webrtc.org/getting-started/peer-connections)

```js
//主动发起连接
const configuration = {'iceServers': [{'urls': 'stun:stun.l.google.com:19302'}]}
const peerConnection = new RTCPeerConnection(configuration);
signalingChannel.addEventListener('message', async message => {
    if (message.answer) {
        const remoteDesc = new RTCSessionDescription(message.answer);   //4.收到remote SDP回复(answer)
        await peerConnection.setRemoteDescription(remoteDesc);          //5.保存remote answer
    }
});
const offer = await peerConnection.createOffer();   //1.生成local SDP(offer)
await peerConnection.setLocalDescription(offer);    //2.保存local offer
signalingChannel.send({'offer': offer});            //3.通过signalingChannel 将local SDP 发送给对端
```

其中
1. `signalingChannel` 是 两端 通过 signal server 创建的信令通道
2. 可以看到这本质就是一个 SDP 的异步交换过程，交换使用的就是 `signalingChannel` 信令通道<br>
   因为此时p2p的连接还没有建立，因此需要一个信令连接
3. 代码中会常常见到 offer， answer， local， remote这几个命名关键词，理解意义有助于看懂代码：
    * 主动发送的 SDP 称为 offer
    * 对应的返回就是 answer
    * 本地生成的就是 local
    * 对端生成的就是 remote，


接着，我们看看接收端的js，在某些场景，接收端可以是服务器，因此不一定使用js，但是逻辑是类似的：
```js
//被动接受连接
const peerConnection = new RTCPeerConnection(configuration);
signalingChannel.addEventListener('message', async message => {
    if (message.offer) {    //2. 接收到SDP消息(offer)
        peerConnection.setRemoteDescription(new RTCSessionDescription(message.offer));  //3. 保存对端offer
        const answer = await peerConnection.createAnswer(); //4. 根据offer 生成SDP回复(answer)
        await peerConnection.setLocalDescription(answer);   //5. 保存自己的answer
        signalingChannel.send({'answer': answer});  //6. 使用信令通道发送answer
    }
}); //1. 监听信令通道
```

这里 `setRemoteDescription` 和 `setRemoteDescription` 的作用，就是将signal server交换的sdp传入内部， 方便 webrtc 判断 session 能否成立

可以看到，两个调用的顺序不重要，因为逻辑就是，创建连接只是需要双方的SDP，顺序并不重要。

chrome内部只需要记录状态，就能正确的处理
1. stable --setLocal()--->  haveLocalOffer  ---setRemote()--> stable
1. stable --setRemote()-->  haveRemoteOffer  --setLocal()---> stable



##### candidate收集--c++代码

Candidate的收集是可以通过由RTCPeerConnection创建，设定configuration中的iceCandidatePoolSize 提前分配<br>
会放在pool中，后续的再触发gather的时候可以直接使用，某些应用场景可以使连接更快建立

也可以通过 `setLocalDescription` 触发的，此时收集发生在sdp offer生成之后。

两种情况，用户都可以自行决定要不要等待candidate，合并到sdp

收集完成的candidate由两个event通知上层
1. `onicecandidate` event回调，来通知新candidate获取
2. `onicegatheringstatechange` event回调，通知所有的candidate获取完毕

如果是提前分配流程如下

|接口|下一层调用|
|---|---|
|`RTCPeerConnection::Create`|RTCPeerConnection* peer_connection= MakeGarbageCollected\<RTCPeerConnection\>(...);|
|`RTCPeerConnection::RTCPeerConnection`|if (!peer_handler_->Initialize(...)) {|
|`RTCPeerConnectionHandler::Initialize`|native_peer_connection_ = dependency_factory_->CreatePeerConnection(...);|
|`PeerConnectionDependencyFactory::CreatePeerConnection`|auto pc_or_error = GetPcFactory()->CreatePeerConnectionOrError(...);|
|`PeerConnectionFactory::CreatePeerConnectionOrError`|auto result = PeerConnection::Create(...);|
|`PeerConnection::Create`|RTCError init_error = pc->Initialize(...);|
|`PeerConnection::Initialize`|const auto pa_result = network_thread()->Invoke\<...\>(...,rtc::Bind(&PeerConnection::InitializePortAllocator_n,...));|
|`PeerConnection::InitializePortAllocator_n`|port_allocator_->SetConfiguration(|
|`PortAllocator::SetConfiguration`|pooled_session->StartGettingPorts();|
|`BasicPortAllocatorSession::StartGettingPorts`|network_thread_->Post(RTC_FROM_HERE, this, MSG_CONFIG_START);|
|`BasicPortAllocatorSession::OnMessage`|case MSG_CONFIG_START:<br>&nbsp;&nbsp;&nbsp;&nbsp;GetPortConfigurations();|
|`BasicPortAllocatorSession::GetPortConfigurations`|ConfigReady(config);|
|`BasicPortAllocatorSession::ConfigReady`|network_thread_->Post(RTC_FROM_HERE, this, MSG_CONFIG_READY, config);|
|`BasicPortAllocatorSession::OnMessage`|case MSG_CONFIG_READY:<br>&nbsp;&nbsp;&nbsp;&nbsp;OnConfigReady(static_cast\<PortConfiguration*\>(message->pdata));|
|`BasicPortAllocatorSession::OnConfigReady`|AllocatePorts();|
|`BasicPortAllocatorSession::AllocatePorts`|network_thread_->Post(RTC_FROM_HERE, this, MSG_ALLOCATE);|
|`BasicPortAllocatorSession::ConfigReady`|case MSG_ALLOCATE:<br>&nbsp;&nbsp;&nbsp;&nbsp;OnAllocate();|
|`BasicPortAllocatorSession::OnAllocate`|DoAllocate(disable_equivalent_phases);|
|`BasicPortAllocatorSession::DoAllocate`|sequence->Start();|
|`AllocationSequence::Start`|session_->network_thread()->Post(RTC_FROM_HERE, this, MSG_ALLOCATION_PHASE);|
|`AllocationSequence::OnMessage`|case PHASE_UDP:<br>&nbsp;&nbsp;&nbsp;&nbsp;CreateUDPPorts();<br>&nbsp;&nbsp;&nbsp;&nbsp;CreateStunPorts();|

之后AllocationSequence 会获取 host， relex 和 relay 三种candidate，且每个media类型都要

如果是 `setLocalDescription` 触发，前面的流程有不同之处

|接口|下一层调用|注释|
|---|---|---|
|`RTCPeerConnection::setLocalDescription`|peer_handler_->SetLocalDescription(request, std::move(parsed_sdp));||
|`RTCPeerConnectionHandler::SetLocalDescription`|if (!peer_handler_->Initialize(...)) {||
|`RTCPeerConnectionHandler::Initialize`| CrossThreadBindOnce(..., &webrtc::PeerConnectionInterface::SetLocalDescription, ..., parsed_sdp.release(), ...);||
|`PeerConnection::SetLocalDescription`|sdp_handler_->SetLocalDescription(std::move(desc), observer);||
|`SdpOfferAnswerHandler::SetLocalDescription`|this_weak_ptr->DoSetLocalDescription(std::move(desc), observer);||
|`SdpOfferAnswerHandler::DoSetLocalDescription`|dtls->ice_transport()->MaybeStartGathering();|这里每一种media都有自己的dtls transport|
|`P2PTransportChannel::MaybeStartGathering`|allocator_sessions_.back()->StartGettingPorts();||
|`BasicPortAllocatorSession::StartGettingPorts`|network_thread_->Post(RTC_FROM_HERE, this, MSG_CONFIG_START);||

之后就和提前分配的流程一致。