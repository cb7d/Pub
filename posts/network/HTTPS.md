---
title: "HTTPS"
date: 2019-03-27T10:46:42+08:00
showDate: true
draft: false
tags: ["blog","HTTP"]
---

# HTTPS

最近又看了一遍 [HTTP权威指南]，每次想写个总结的时候都会拖延症爆发，今天决定简单总结下我们日常使用的的网络传输协议和TLS相关。

## OSI (Open System Interconnect)

先列一张能够体现不同协议在OSI七层模型中的表格

| 层级 | 层级名称   | 应用                                                         |
| ---- | ---------- | ------------------------------------------------------------ |
| 7    | 应用层     | 例如HTTP、SMTP、SNMP、FTP、Telnet、SIP、SSH、NFS、RTSP、XMPP、Whois、ENRP、TLS |
| 6    | 表示层     | 例如XDR、ASN.1、SMB、AFP、NCP                                |
| 5    | 会话层     | 例如ASAP、ISO 8327 / CCITT X.225、RPC、NetBIOS、ASP、IGMP、Winsock、BSD sockets |
| 4    | 传输层     | 例如TCP、UDP、RTP、SCTP、SPX、ATP、IL                        |
| 3    | 网络层     | 例如IP、ICMP、IPX、BGP、OSPF、RIP、IGRP、EIGRP、ARP、RARP、X.25 |
| 2    | 数据链路层 | 例如以太网、令牌环、HDLC、帧中继、ISDN、ATM、IEEE 802.11、FDDI、PPP |
| 1    | 物理层     | 例如线路、无线电、光纤                                       |


### 先从 TCP/IP 说起

> IPS（Internet Protocol Suite）又叫做互联网协议套件、是一套网络传输协议家族，也就是我们熟悉的TCP/IP协议族，（又因为TCP、IP为不同层级的协议，当多个层级的协议共同工作时类似计算机科学中的堆栈、所以又叫做TCP/IP协议栈）

TCP/IP 中包含一系列用于处理数据通信的协议

- TCP (传输控制协议) - 应用程序之间通信
- UDP (用户数据报协议) - 应用程序之间的简单通信
- IP (网际协议) - 计算机之间的通信
- ICMP (因特网消息控制协议) - 针对错误和状态
- DHCP (动态主机配置协议) - 针对动态寻址

这里要说一下TCP、HTTP、socket的关系

> 实际上socket只是一个Api，方便我们去调用底层封装好的TCP/IP协议，但是socket与TCP/IP并没有必然的关系，socket接口在设计的时候就希望也能够适应其他协议，而HTTP是基于TCP之上的应用层协议，有一个很有趣的比喻就是如果TCP是高速路，那么HTTP就是高速路上的货车。

好，下面重头戏来了，TCP三次握手(建立连接)四次挥手(断开连接)

客户端 - Client、 服务端 - Server  
两个整型变量 J、K

### 三次握手(建立连接)

1. Server socket绑定端口，等待消息
2. (first) Client 打开socket(connect) 、发送 SYN J
3. (second) Server 收到消息，向客户端发送 SYN K、 ack J+1
4. (third) Client 收到应答、发送 ack K+1
5. Server 开始读取Client发送的信息

至此、连接建立完成、这里有一个问题，为什么建立连接不是2次握手、为什么不是4次握手？

恰好是三次的原因是三次刚刚好，4次浪费网络资源没必要，而如果改为2次，那么client在向server发送第一次握手的时候因为某些原因舍弃了这次连接(或是因其他原因被迫强制关闭)而并没有通知Server，那么如果只需要2次握手无疑会造成server会一直处于盲等状态。

### 四次挥手(连接拆除)

三个整型变量 x、y、z

与建立连接不同的是、断开连接可以由任意一方发起

1. 主动方发送fin=1、ack=z、seq=x
2. 被动方发送ack=x+1、seq=z
3. 被动方发送fin=1、ack=x、seq=y
4. 主动方发送ack=y、seq=x

那么，为何挥手要挥4次呢，原因在于拆除连接时收到fin报文，仅仅代表对方不再发送数据，而接收到fin报文的一方可能还有数据没发送完，所以被动方的fin和ack要分开发送。

### HTTP 连接

那么什么是HTTP连接呢，所谓HTTP，就是基于TCP/IP协议的上层应用层协议，
所以我们可以理解HTTP连接就是以TCP/IP为传输层协议，以HTTP协议作为应用层协议的连接

就比如我们在浏览器输入 https://k.felixplus.top/ 、会先通过 TCP握手建立连接，这个时候传输的数据会通过HTTP协议进行解析，最后到达我们的浏览器进行绘制展示

### HTTPS

HTTPS = HTTP + (TLS/SSL)

其中，TLS/SSL 为 HTTP 提供了安全加密传输的功效

相比之下，HTTPS比HTTP多提供了以下几点：

1. 数据完整性：内容传输经过完整性校验
2. 数据隐私性：内容经过对称加密，每个连接生成一个唯一的加密密钥
3. 身份认证：第三方无法伪造服务端(客户端)身份

那么，HTTPS的一次传输过程是怎样的呢，先说结论:

1. Server 被动监听
2. Client 发送连接请求
3. Server 返回公钥
4. Client 生成随机值（对称加密的私钥）通过Server返回的公钥加密
5. Client 将密文发送回Server 并等待确认
6. 双方开始使用客户端生成的对称加密密钥进行通讯

在继续之前我们需要先来回顾一下什么是对称加密，什么是非对称加密:

| 对称方式 | 加密方法                   | 风险     | 加解密速度   |
| -------- | -------------------------- | -------- | ------------ |
| 非对称   | 数据的加解密由一个密钥解决 | 风险较高 | 加解密速度快 |
| 对称     | 数据使用公钥加密、私钥解密 | 风险较小 | 加解密速度慢 |

从这里不难看出，HTTPS同时使用到对称加密和非对称加密的原因是如果数据传输量很大，一直使用非对称加密会消耗更多资源，产生不必要的浪费，而在传输数据之前先通过非对称加密发送客户端生成的私钥在安全性能上足以胜任，所以这里并没有完全使用非对称加密。

下面是使用 `curl` 命令debug HTTPS 请求的输出

```bash
curl -v https://aground.felixplus.top
```

```bash
* Rebuilt URL to: https://aground.felixplus.top/
*   Trying 47.75.98.166...
* TCP_NODELAY set
* Connected to aground.felixplus.top (47.75.98.166) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-CHACHA20-POLY1305
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=aground.felixplus.top
*  start date: Feb 15 10:01:34 2019 GMT
*  expire date: May 16 10:01:34 2019 GMT
*  subjectAltName: host "aground.felixplus.top" matched cert's "aground.felixplus.top"
*  issuer: C=US; O=Let's Encrypt; CN=Let's Encrypt Authority X3
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7feef980b800)
> GET / HTTP/2
> Host: aground.felixplus.top
> User-Agent: curl/7.54.0
> Accept: */*
>
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 200
< server: nginx/1.14.0 (Ubuntu)
< date: Mon, 25 Mar 2019 07:13:40 GMT
< content-type: text/html
< content-length: 2121
< last-modified: Tue, 19 Mar 2019 09:57:48 GMT
< vary: Accept-Encoding
< etag: "5c90bd1c-849"
< accept-ranges: bytes
<
* Connection #0 to host aground.felixplus.top left intact
<!doctype html><html lang="en"><head><meta charset="utf-8"/><link rel="shortcut icon" href="/panda.png"/><meta name="viewport" content="width=device-width,initial-scale=1,shrink-to-fit=no"/><meta name="theme-color" content="#000000"/><link rel="manifest" href="/manifest.json"/><title>AGround</title><link href="/static/css/2.4126af51.chunk.css" rel="stylesheet"><link href="/static/css/main.85ae910b.chunk.css" rel="stylesheet"></head><body><noscript>You need to enable JavaScript to run this app.</noscript><div id="root"></div><script>!function(l){function e(e){for(var r,t,n=e[0],o=e[1],u=e[2],f=0,i=[];f<n.length;f++)t=n[f],p[t]&&i.push(p[t][0]),p[t]=0;for(r in o)Object.prototype.hasOwnProperty.call(o,r)&&(l[r]=o[r]);for(s&&s(e);i.length;)i.shift()();return c.push.apply(c,u||[]),a()}function a(){for(var e,r=0;r<c.length;r++){for(var t=c[r],n=!0,o=1;o<t.length;o++){var u=t[o];0!==p[u]&&(n=!1)}n&&(c.splice(r--,1),e=f(f.s=t[0]))}return e}var t={},p={1:0},c=[];function f(e){if(t[e])return t[e].exports;var r=t[e]={i:e,l:!1,exports:{}};return l[e].call(r.exports,r,r.exports,f),r.l=!0,r.exports}f.m=l,f.c=t,f.d=function(e,r,t){f.o(e,r)||Object.defineProperty(e,r,{enumerable:!0,get:t})},f.r=function(e){"undefined"!=typeof Symbol&&Symbol.toStringTag&&Object.defineProperty(e,Symbol.toStringTag,{value:"Module"}),Object.defineProperty(e,"__esModule",{value:!0})},f.t=function(r,e){if(1&e&&(r=f(r)),8&e)return r;if(4&e&&"object"==typeof r&&r&&r.__esModule)return r;var t=Object.create(null);if(f.r(t),Object.defineProperty(t,"default",{enumerable:!0,value:r}),2&e&&"string"!=typeof r)for(var n in r)f.d(t,n,function(e){return r[e]}.bind(null,n));return t},f.n=function(e){var r=e&&e.__esModule?function(){return e.default}:function(){return e};return f.d(r,"a",r),r},f.o=function(e,r){return Object.prototype.hasOwnProperty.call(e,r)},f.p="/";var r=window.webpackJsonp=window.webpackJsonp||[],n=r.push.bind(r);r.push=e,r=r.slice();for(var o=0;o<r.length;o++)e(r[o]);var s=n;a()}([])</script><script src="/static/js/2.5167ae78.chunk.js"></script><script src="/static/js/main.05967284.chunk.js"></script></body></html>%
```