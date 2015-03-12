title: wireshark网络包分析
date: 2015-02-11 08:08:08
tags: 
- 网络抓包
- 移动端
category: 网络
---

本文的前提是，我们已经知道抓包方法或者手头有个pcap包。不了解抓包的，可以移步上一篇文章：[移动web前端抓包][1]

关于wireshark分析方法，网上教程不少，基于此说一些个人的理解。

本文以小度wifi为热点，让iPhone连接对应的热点。

<!-- more -->

## 启动监听

启动监听时，需要选择监听的设备，只要针对性的选择小度的网络连接（或者说端口）就可以了。

## 主界面

wireshark的界面可以自定，本人常用的如下：

<img src="/images/wireshark-home-view.png" alt="wireshark home view" width="800" />

## 协议分析

1）TCP

开启监听后，会抓取所有协议的网络包，这时候就需要用到过滤了。
如前面主界面视图所说，包详情的面板项可以对应过滤输入框的一个对象。每个对象`可过滤的属性对应面板详情的数据项`。

实际开发中，如果只需要关注HTTP的请求和返回。这个时候可以用http对象，如：
```
http.request.full_uri contains "weixin"
http.request.method == "POST"
...
```
其他过滤方式自行挖掘。以第一个过滤规则为例，过滤结果：

<img src="/images/wireshark-packet-tcp-filter.png" alt="wireshark packet tcp filter" width="800" />

如果需要查看第一个POST请求，选择后右键`Follow TCP Stream`就可以看到整个请求的情况。
`Follow TCP Stream`操作实际也是执行了一次过滤，可以看到过滤输入框的内容。

<img src="/images/wireshark-packet-tcp-stream.png" alt="wireshark packet tcp stream" width="800" />

关于tcp.stream的值，官方说法是：

```
the stream index is an internal Wireshark mapping to: 
[IP address A, TCP port A, IP address B, TCP port B]
```

实际中，我们不用过分关注。

分析整个请求，可以得到你想要的信息，如连接建立的三次握手、请求和返回、四次挥手等。
分析时，注意按照ack和seq的值来对应。

2）ARP

主机（iPhone手机）发送信息时将包含目标（小度路由）IP地址的ARP请求`广播`到网络上的所有主机，并接收返回消息，以此确定目标的物理地址。
对比以下的网络包截图，可以很好的理解ARP的意图。

<img src="/images/wireshark-packet-arp.png" alt="wireshark packet arp" width="800" />

小度路由返回mac地址等信息：

<img src="/images/wireshark-packet-arp-reply.png" alt="wireshark packet arp reply" width="400" />

3）DNS

iPhone向DNS服务器解析域名（short.weixin.qq.com）的ip：

<img src="/images/wireshark-packet-dns.png" alt="wireshark packet dns" width="800" />

DNS服务器解析域名，并应答：

<img src="/images/wireshark-packet-dns-reply.png" alt="wireshark packet dns reply" width="400" />

域名解析结果，跟命令行下运行nslookup的返回是一致的。

另外，也可以从面板中看到，DNS查询和返回是按照`UDP协议`。
在网络包列表中，选中其中一个帧，右键`Follow UDP Stream`可以在网络列表中只显示该数据流。


[1]: /2015/02/10/network-packet-trace/
