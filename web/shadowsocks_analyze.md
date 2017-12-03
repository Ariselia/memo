[original](https://loggerhead.me/posts/shadowsocks-yuan-ma-fen-xi-xie-yi-yu-jie-gou.html)
# Shadowsocks 源码分析——协议与结构

[Shadowsocks](https://github.com/shadowsocks/shadowsocks/tree/master) 是一款著名的 SOCKS5 代理工具，深受人民群众喜爱。它的源码工程质量很高，十分便于研究。不过当你真正开始读源码的时候，会有一种似懂非懂的感觉，因为虽然它的大体框架容易理解，但是其中的诸多细节却不是那么简单明了。

本文将基于 [2.9.0 版本的源码](https://github.com/loggerhead/shadowsocks/tree/8e8ee5d490ce319b8db9b61001dac51a7da4be63)对 shadowsocks 进行分析，希望读者看完以后能对 shadowsocks 的原理有个大体上的认识。为了行文简洁，在示例中我们用 ss 指代 shadowsocks。

---

* [SOCKS5 协议](#SOCKS5 协议)
  *[握手阶段](#握手阶段)
  *[建立连接](#建立连接)
  *[传输阶段](#传输阶段)
*[整体结构](#整体结构)
*[事件处理](#事件处理)
  *[eventloop.py](#eventloop.py)
  *[tcprelay.py](#tcprelay.py)
  *[udprelay.py](#udprelay.py)
  *[asyncdns.py](#asyncdns.py)
*[总结](#总结)
---

# SOCKS5 协议
无论是实现什么网络应用，首当其冲的就是确定通讯协议。SOCKS5 协议作为一个同时支持 TCP 和 UDP 的应用层协议（[RFC](https://www.ietf.org/rfc/rfc1928.txt) 只有短短的 7 页），因为其简单易用的特性而被 shadowsocks 青睐。我们先从 SOCKS5 协议入手，一点一点剖析 shadowsocks。

## 握手阶段
客户端和服务器在握手阶段协商认证方式，比如：是否采用用户名/密码的方式进行认证，或者不采用任何认证方式。

客户端发送给服务器的消息格式如下（数字表示对应字段占用的字节数）

+----+----------+----------+
|VER | NMETHODS | METHODS  |
+----+----------+----------+
| 1  |    1     |  1~255   |
+----+----------+----------+



