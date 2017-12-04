[original](https://loggerhead.me/posts/shadowsocks-yuan-ma-fen-xi-xie-yi-yu-jie-gou.html)
# Shadowsocks 源码分析——协议与结构

[Shadowsocks](https://github.com/shadowsocks/shadowsocks/tree/master) 是一款著名的 SOCKS5 代理工具，深受人民群众喜爱。它的源码工程质量很高，十分便于研究。不过当你真正开始读源码的时候，会有一种似懂非懂的感觉，因为虽然它的大体框架容易理解，但是其中的诸多细节却不是那么简单明了。

本文将基于 [2.9.0 版本的源码](https://github.com/loggerhead/shadowsocks/tree/8e8ee5d490ce319b8db9b61001dac51a7da4be63)对 shadowsocks 进行分析，希望读者看完以后能对 shadowsocks 的原理有个大体上的认识。为了行文简洁，在示例中我们用 ss 指代 shadowsocks。

---

* [SOCKS5 协议](#SOCKS5 协议)<br>
  *[握手阶段](#握手阶段)<br>
  *[建立连接](#建立连接)<br>
  *[传输阶段](#传输阶段)<br>
*[整体结构](#整体结构)<br><br>
*[事件处理](#事件处理)<br>
  *[eventloop.py](#eventloop.py)<br>
  *[tcprelay.py](#tcprelay.py)<br>
  *[udprelay.py](#udprelay.py)<br>
  *[asyncdns.py](#asyncdns.py)<br>
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


* VER 字段是当前协议的版本号，也就是 5；<br>
* NMETHODS 字段是 METHODS 字段占用的字节数；<br>
* METHODS 字段的每一个字节表示一种认证方式，表示客户端支持的全部认证方式。<br>

服务器在收到客户端的协商请求后，会检查是否有服务器支持的认证方式，并返回客户端如下格式的消息：


+----+--------+
|VER | METHOD |
+----+--------+
| 1  |   1    |
+----+--------+


对于 shadowsocks 而言，返回给客户端的值只有两种可能：

`0x05 0x00`：告诉客户端采用无认证的方式建立连接；
`0x05 0xff`：客户端的任意一种认证方式服务器都不支持。


举个例子，就 shadowsocks 而言，最简单的握手可能是这样的：

```
client -> ss: 0x05 0x01 0x00
ss -> client: 0x05 0x00
```

如果客户端**还**支持用户名/密码的认证方式，那么握手会是这样子：

```
client -> ss: 0x05 0x02 0x00 0x02
ss -> client: 0x05 0x00
```
如果客户端**只**支持用户名/密码的认证方式，那么握手会是这样子：

```
client -> ss: 0x05 0x01 0x02
ss -> client: 0x05 0xff
```
+----+-----+-------+------+----------+----------+
|VER | CMD |  RSV  | ATYP | DST.ADDR | DST.PORT |
+----+-----+-------+------+----------+----------+
| 1  |  1  |   1   |  1   | Variable |    2     |
+----+-----+-------+------+----------+----------+

* `CMD` 字段：command 的缩写，shadowsocks 只用到了：
		* `0x01：建立 TCP 连接
		* `0x03`：关联 UDP 请求
* `RSV`字段：保留字段，值为 0x00；
* `ATYP` 字段：address type 的缩写，取值为：
	* `0x01`：IPv4
	* `0x03`：域名
	* `0x04`：IPv6
* `DST.ADDR` 字段：destination address 的缩写，取值随 ATYP 变化：
	* `ATYP == 0x01`：4 个字节的 IPv4 地址
	* `ATYP == 0x03`：1 个字节表示域名长度，紧随其后的是对应的域名
	* `ATYP == 0x04`：16 个字节的 IPv6 地址
* `DST.PORT` 字段：目的服务器的端口。

在收到客户端的请求后，服务器会返回如下格式的消息


+----+-----+-------+------+----------+----------+
|VER | REP |  RSV  | ATYP | BND.ADDR | BND.PORT |
+----+-----+-------+------+----------+----------+
| 1  |  1  |   1   |  1   | Variable |    2     |
+----+-----+-------+------+----------+----------+

* REP 字段：用以告知客户端请求处理情况。在请求处理成功的情况下，shadowsocks 将这个字段的值设为 0x00，否则，shadowsocks 会直接断开连接；<br>
* 其它字段和请求中字段的取值类型一样。<br>

举例来说，如果客户端通过 shadowsocks 代理 `127.0.0.1:8000` 的请求，那么客户端和 shadowsocks 之间的请求和响应是这样的


```
#    request: VER  CMD  RSV  ATYP DST.ADDR            DST.PORT
client -> ss: 0x05 0x01 0x00 0x01 0x7f 0x00 0x00 0x01 0x1f 0x40
#   response: VER  REP  RSV  ATYP BND.ADDR            BND.PORT
ss -> client: 0x05 0x00 0x00 0x01 0x00 0x00 0x00 0x00 0x10 0x10
```

这里 `0x7f 0x00 0x00 0x01 0x1f 0x40` 对应的是 `127.0.0.1:8000`。需要注意的是，当请求中的 `CMD == 0x01` 时，绝大部分 SOCKS5 客户端的实现都会忽略 SOCKS5 服务器返回的 `BND.ADDR` 和 `BND.PORT` 字段，所以这里的 `0x00 0x00 0x00 0x00 0x10 0x10` 只是 shadowsocks 返回的一个无意义的地址和端口[1](https://loggerhead.me/posts/shadowsocks-yuan-ma-fen-xi-xie-yi-yu-jie-gou.html#fn:bnd.addr)。


## 传输阶段

SOCKS5 协议只负责建立连接，在完成握手阶段和建立连接之后，SOCKS5 服务器就只做简单的转发了。假如客户端通过 shadowsocks 代理 `google.com:80`（用 `remote` 表示），那么整个过程如图所示：




![socks5-example.svg](./socks5-example.svg)



整个过程中发生的传输可能是这样的：

```
# 握手阶段
client -> ss: 0x05 0x01 0x00
ss -> client: 0x05 0x00
# 建立连接
client -> ss: 0x05 0x01 0x00 0x03 0x0a b'google.com'  0x00 0x50
ss -> client: 0x05 0x00 0x00 0x01 0x00 0x00 0x00 0x00 0x10 0x10
# 传输阶段
client -> ss -> remote
remote -> ss -> client
...
```

`b'google.com'` 表示 `google.com` 对应的 ASCII 码

## 整体结构

在进一步了解 shadowsocks 的内部构造之前，我们粗略的看一下各个模块分别做了些什么：


* tcprelay.py：核心部分，整个 SOCKS5 协议的实现都在这里。负责 TCP 代理的实现；
* udprelay.py：负责 UDP 代理的实现；
* asyncdns.py：实现了简单的异步 DNS 查询；
* eventloop.py：封装了三种常见的 IO 复用函数——epoll、kqueue 和 select，提供统一的接口；
* encrypt.py：提供统一的加密解密接口；
* crypto：封装了多种加密库的调用，包括 OpenSSL 和 libsodium；
* daemon.py：用于实现守护进程；
* shell.py：读取命令行参数，检查配置；
* common.py：提供一些工具函数，比如：将 bytes 转换成 str、解析 SOCKS5 请求；
* lru_cache.py：实现了 LRU 缓存；
* local.py：shadowsocks 客户端（即 sslocal 命令）的入口；
* server.py：shadowsocks 服务器（即 ssserver 命令）的入口。

























































