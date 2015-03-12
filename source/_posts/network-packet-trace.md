title: 移动web前端抓包
date: 2015-02-10 08:08:08
tags: 
- 调试
- 移动端
category: 编码
---
数据包是最贴近真实的场景还原，可以作为技术人员间解决问题的可信依据（出问题，先给我包！）。

移动端抓包调试是重要技能。浏览器自带的开发者工具，如chrome F12就是大家都熟悉的抓包工具。抓包是门学问，不过通常情况下，我们只需要关注发送和接收的数据包内容就可以解决问题了。

本文主要总结本人在移动web前端开发中的抓包方法，不涉及原理分析，主要用来解决一些关乎网络状况、手机特性等线下不好复现的问题，比方说：在微信中出现不符合预期的页面返回、或者在某网络下出现劫持或乱码、或者某android手机偶发登录问题。

<!-- more -->

## 1. 热点监听

通过监听网卡的出入流量，来捕获数据包。实施方法为：

1）手机连wifi热点
在pc上设置热点，手机连该热点。个人喜欢用`小度wifi`即插即用的方式。

2）Wireshark监听热点
`Wireshark`是学习网络协议的好工具。有人说用fiddler，在Wireshark面前，这方面它还是逊色了。

## 2. 本地监听

本地监听，需要有设备的`root权限`，使用`tcpdump`来抓取网络包，并用`Wireshark`进行分析。

1）Android手机

i）必要的`安装工具`[下载][1]。

ii）在pc上安装`adb`（android debug bridge）命令行工具，将adb命令配置到系统环境变量。
打开手机的usb开发调试开关，用数据线连接，进入命令行就可以adb，其用法可以自己另查。

提示：adb的默认端口是5037，如不能adb，很可能是端口被占用，可以试着找出该进程kill掉。试试以下命令：
```
netstat -ano|findstr "5037"
tasklist /fi "pid eq {pid}"
taskkill /pid {pid} /f
```

iii）将tcpdump push到手机
目录可以自己选，我一般push到/sdcard目录
```
adb root
adb remount
adb push {where you put tcpdump} /sdcard
adb push {where you put tcpdump.sh} /sdcard
```

iv）进入adb shell
```
adb shell
cd /sdcard
```
提示：可能需要修改tcpdump的权限，可以看命令行的错误提示。

v）开始抓包
可以直接运行tcpdump命令抓包。为了方便，把命令写到脚本了。
```
sh tcpdump.sh
```
可以看到，我们将抓包内容存到/sdcard下的capture.pcap文件。

vi）pc下获取pcap文件
```
adb pull /sdcard/capture.pcap {where you put capture.pcap}
```

vii）Wireshark打开网络包分析



2）iPhone

iOS本地抓包也是需要root权限，方法跟android近似。不过这个方法代价还是不小（mac+iphone），不实用。
有了前面提到的方法，也足可以解决iPhone的抓包了。



## 3. 怎么选择

1）普通的问题，采用热点监听；

2）移动网络才能复现的问题，采用本地监听；

3）无root权限的又想用移动网络的，找一个有root的手机做热点，对该手机抓包；

以上不论哪种方法，都只提到了抓包，包的分析参考[wireshark网络包分析][2]

[1]: http://pan.baidu.com/s/1qWofHHq
[2]: /2015/02/11/wireshark-network-analyzer/
