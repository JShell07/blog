---

title: WebRTC_Best_Relay_Network_Selection
date: 2018-06-22 10:04:39
tags:
    - WebRTC
categories: WebRTC    
---

## 1. Prepare

![](https://raw.githubusercontent.com/JShell07/images/master/webRTC/RelayNetworkSelection/WebRTC_Best_Relay_Network_Sort_Itself.png)

<!--more-->

Local 端与remote 连接前，local 端会收取本地的SDP、Candidate。Candidate 其包含了一些网络信息，类似于

```json
{
   "candidate" : "candidate:3724402988 1 udp 41885695 172.28.28.24 51570 typ relay raddr 101.206.166.96 rport 23166 generation 0 ufrag 8B5q network-id 3 network-cost 50",
   "id" : "sdparta_0",
   "label" : 0,
   "type" : "candidate"
}
```

收集到的Candidates，会依据：

- IPv6 > IPv4

- UDP > TCP

以上的准则进行自我的排序准备动作。

## 2. Connection

![](https://raw.githubusercontent.com/JShell07/images/master/webRTC/RelayNetworkSelection/WebRTC_Best_Relay_Network_Get_Candidate.png)

在收到remote端的candidate后，会进行如下动作：

| 序号  | 动作                                                                                                                                                                                                                                                                                  |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | 将新收到的candidate网络与local 端的每一个candidate都创建一个connection                                                                                                                                                                                                                                |
| 2   | 将connection 连接对，更新到预备队列中并且排序。排序的依据是：1）**writable, receiving**; 2) **estimated latency is lowest**;                                                                                                                                                                                  |
| 3   | MaybeSwitchSelectedConnection()，可能现在到达的remote candidate 是在有了选择连接之后，如果新到的remote candidate 更好，则将此candidate 替换selected_。那怎样是好的呢？1)**wirtable、receiving、connected states**; 2)**nomination state**; 3) **last data received time**; 4)**lower cost、higher priority**;5)**rtt(往返时间间隔)**; |
| 4   | 开始尝试stun ping 动作                                                                                                                                                                                                                                                                    |
| 5   | 从之前拍寻过的队列中选择下一个后背ping item。当然，这里也是有些策略：1)**unpinged connections have priority over pinged ones**; 2)**select best connections from every network, the one with the earliest last-ping-sent time**                                                                                   |

## 3. Selection

![](https://raw.githubusercontent.com/JShell07/images/master/webRTC/RelayNetworkSelection/WebRTC_Best_Relay_Network_Get_Ping_Response.png)

在收到stun ping的响应之后，我们可以理解为，这条线路是通的。


