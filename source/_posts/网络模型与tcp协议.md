---
title: 网络模型与tcp协议
categories: Hexo教程
date: 2019-03-14 15:25:33
tags: ["tcp"]
description: 网络模型与tcp协议
---


OSI模型是ISO制定的网络模型规范，而TCP/IP模型是实际中更通用的协议族。

![图片描述](/tfl/pictures/201901/tapd_20091751_1547276540_10.png)


![图片描述](/tfl/pictures/201901/tapd_20091751_1547276563_65.png)

一个网络请求的顺序如下:
即IP路由寻址，再根据MAC地址确认目标机器，然后通过TCP/UDP进行端口数据传输，最后通过应用层协议解析数据。
MAC地址可以使用`ifconfig`，osx系统可以用`networksetup -listallhardwareports`,ether即MAC地址。

![图片描述](/tfl/pictures/201901/tapd_20091751_1547276575_7.png)


## 以太帧
在TCP/IP模型中，数据是自顶向下包装的。最后的一个以太网帧如下结构


![图片描述](/tfl/pictures/201901/tapd_20091751_1547276585_16.png)

## IP
IP是无连接，无状态，不保证发送的包是否有序到达。
```$xslt
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Version|  IHL  |Type of Service|          Total Length         |
   |版本号  | 头长度 |  服务类型(8位) |           总长度(16位)
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |         Identification        |Flags   |    Fragment Offset   |
   |            标识(16位）         |标志(3位)|      分偏移段(13位)
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Time to Live |    Protocol   |         Header Checksum       |
   |  生存时间(8位) | 挂载协议标识(8位)|            校验和(16位)
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                       Source Address                          |
   |                        源IP地址(16位）                          |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Destination Address                        |
   |                      目标IP地址(16位)                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   |                      选项                      |     填充      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                             Data                              |
   |                          TCP/UDP数据                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
         
- Version: IPv4/IPv6
- Total Length: IP报文头与数据总长度，<=MTU(Maximum Transmission Unit,以太网一般1500字节)
- Identification：分片后+1，用于分片重组
- Flags：3bit, 第一位0，第二位0可以分片1不可分片，第三位0最后一片，1不是最后一片
- Fragment Offset: 分片后再原组中的相对偏移量
- Time to Live: 可经过最大的路由器数
- Protocol: 下一层协议，ICMP/TCP/UDP    
```
      
## TCP

> Transmission Control Protocol (TCP): TCP provides reliable, ordered, and error-checked delivery of a stream of octets (bytes) between applications running on hosts communicating via an IP network
>
TCP是面向连接、确保数据在端对端可靠传输的协议。
```$xslt
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-------------------------------+-------------------------------+
   |          Source Port          |       Destination Port        |
   |        源机器端口号(16位)        |      目标机器端口号(16位)       |
   +-------------------------------+-------------------------------+
   |                        Sequence Number                        |
   |                        序列号(32位）seq=？                      |
   +---------------------------------------------------------------+
   |                    Acknowledgment Number                      |
   |                     确认序号(32位）ack=？                       |
   +-------+-----------+-+-+-+-+-+-+-------------------------------+
   |  Data |           |U|A|P|R|S|F|                               |
   | Offset| Reserved  |R|C|S|S|Y|I|            Window             |
   |       |           |G|K|H|T|N|N|                               |
   |  头部  |           |           |                               |
   |  长度  |   保留    |    FALG   |          滑动窗口大小           |
   | (4位) |   (6位）   |           |           （16位）             |
   +-------+-----------+-+-+-+-+-+-+-------------------------------+
   |           Checksum            |         Urgent Pointer        |
   |           校验和(16位)         |          紧急指针(16位）        |
   +-------------------------------+---------------+---------------+
   |                    Options                    |    Padding    |
   +-----------------------------------------------+---------------+
   |                             data                              |
   +---------------------------------------------------------------+
- Sequence Number: seq,本报文段所发送的数据的第1个字节的序号。解决的是包乱序的问题
- Acknowledgment Number: ack,接受的seq+1。即解决是否丢包的问题
- ACK(Acknowledgement Number): 根据ack确认收到的数据
- SYN(Synchronize Sequence Numbers): 建立连接时的同步信息
- FIN(Finish): 意味将要关闭连接
- Window:TCP的流量控制，这个值是接收端期望接收的字节数。睡着ack一起发给发送方。
```

### 可靠传输保证

#### seq+ack
seq的编号可以保障报传递时候的顺序
ack=seq+1，可以保证是否丢包，同时期望下次请求的seq=ack

##### 三次握手与四次挥手

![图片描述](/tfl/pictures/201901/tapd_20091751_1547276603_53.jpg)
我们通过WireShark抓包验证下：

3次握手抓包
![图片描述](/tfl/pictures/201901/tapd_20091751_1547276617_99.png)
4次挥手抓包
![图片描述](/tfl/pictures/201901/tapd_20091751_1547276624_40.png)

#### TCP重传机制
seq+ack可以保证理想情况下的可靠和有序传播，如果遇到丢包或者网络超时这种非理想情况下，就需要重传机制来进行补偿。

重传触发有两种机制：
1. 时间维度(超时重传)。通过RTT(Round Trip Time)推算出Timeout，RTO（Retransmission TimeOut）。
2. 数据维度(快速重传)。发送方连续收到3次相同的ack，就重传。
无论是哪种机制，都需要做一个选择，即丢包之后的数据如何处理？
例如发送端发送了1,2,3,4,5
其中3丢包了。
接收端实际接受了1,2,4,5，同时ack返回了3，但是4,5的ack并没有返回，发送方无法判断4,5是否丢包。
一种是仅重传timeout的包。也就是第3份数据。
另一种是重传timeout后所有的数据，也就是第3，4，5这三份数据。

##### SACK(Selective Acknowledgment)
在TCP头里加一个SACK,SACK存储的是收到的数据碎片，如下图所示。
![图片描述](/tfl/pictures/201901/tapd_20091751_1547276654_94.jpg)
   
### 流量控制
在保证可靠传输的情况下，我们还需要根据网络拥堵情况与数据处理速度来进行流量控制，防止出现阻塞丢包的情况。
#### 滑动窗口
滑动窗口类似`背压`的作用，接收端通过传给发送端window大小，从而控制发送端发送数据的大小。
滑动窗口分为接受窗口，发送窗口。
对于TCP字节流，分为4部分:  
1. 已发送且已ack
2. 已发送但未ack
3. 未发送但对方可接受
4. 对方未允许发送  

其中2和3即是发送窗口。
![图片描述](/tfl/pictures/201901/tapd_20091751_1547276663_14.png)

下面是一个完整的请求实例：
![图片描述](/tfl/pictures/201901/tapd_20091751_1547276671_32.png)

#### 拥塞控制

滑动窗口只解决的客户端处理数据的能力，TCP协议还对拥堵控制进行一些管控。
TCP通过通信双方维护一个拥塞窗口（cwnd，congesion window）值来决定发送速率。网络越阻塞，cwnd值越小，发送越缓慢。

拥塞控制一般以慢热启动，即刚连接时，速率较慢，根据网络情况，指数级的增加cwnd值。
当到达ssthresh（slow start threshold，一般65535字节）后会运用拥塞避免算法进行一个线性增加，直至一个最佳的速率。
当然还有很多算法没有介绍，但最终的目的都是为了结合网络情况找到一个合适的传输速率、


----
## 参考文献
1. [TCP 的那些事儿](https://coolshell.cn/articles/11564.html) 
2. https://zh.wikipedia.org/wiki/TCP/IP%E5%8D%8F%E8%AE%AE%E6%97%8F
3. https://support.huawei.com/hedex/pages/EDOC100010596730006905/04/EDOC100010596730006905/04/resources/message/cd_feature_cover.html
4. [码出高效](https://www.amazon.cn/dp/B07HCDC3C2/ref=sr_1_1?s=books&ie=UTF8&qid=1546455236&sr=1-1&keywords=%E7%A0%81%E5%87%BA%E9%AB%98%E6%95%88)

