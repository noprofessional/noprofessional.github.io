# Webrtc REMB-接收方带宽估算

remb(Receiver Estimated Maximum Bitrate)，是一种基于协议的接收方的带宽估算

### 一句话

remb假设 传输时间 跟 size 成线性关系，同时收到 网络拥塞 影响 的模型，利用卡尔曼滤波检测网络拥塞增大的时刻，并以此刻的带宽作为预估带宽

### 原理

考虑当链路瓶颈为1000kbps时，并且当前链路只有你一个人发送时，如下图。你每个包的消耗的时间为

$$
T=\cfrac{size}{ bandwidth}
$$

![没有阻塞的情况-4s](<.gitbook/assets/remb.drawio (1).png>)

但是当链路有多个人一起发送的时候，作为瓶颈就要同时发送多方的数据，导致原始数据的间隔增加（增加的是发送其他使用者的数据的消耗），如下图。

![有阻塞的情况-8s](<.gitbook/assets/remb\_block.drawio (1).png>)

此时，每个包通过的时间还要等待链路其他使用者的包（图中间隔的部分）通过，即

$$
T'=\cfrac{size+othersize}{ bandwidth}=\cfrac{size}{bandwidth}+\cfrac{othersize}{bandwidth}= T + T_{other}
$$

以上都是简化之后的情况，实际上用户发送的每个包大小不一样，链路其他用户同时间占用的带宽也未知,链路带宽也可能随时间变化。其他用户占用的带宽跟瓶颈（中间路由器）的调度策略，跟每个用户采用的阻塞算法（是否公平）都有关。

不过可以确定的是，阻塞越严重的情况 $$T_{other}$$​的值越大，

