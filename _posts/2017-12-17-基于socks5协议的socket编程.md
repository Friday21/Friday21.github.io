---
layout:     post
title:      基于socks5协议的socket编程
date:       2017-12-17
author:     "FridayLi"
catalog: true
tags:
  - Python
  - Socket
---


我们平时写代码时虽然很少直接接触socket编程， 但这部分其实应该算是基本功，一个做web编程的人应该至少要熟悉这方面的知识。这篇文章先是介绍了Python下socket编程用到的模块， 然后讲解了sock5协议通讯的具体步骤， 最后解读了shadowsocks组早版本的代码， 研究了socks5在Python下的具体实现。
## 1. Python中的socket模块
#### Python中提供了两个相关的模块
* Socket 它提供了标准的BSD Socket API。
* SocketServer 它提供了服务器重心，可以简化网络服务器的开发。

#### Socket 类型
套接字格式：socket(family, type[,protocal]) 使用给定的套接族，套接字类型，协议编号（默认为0）来创建套接字

socket 类型 | 描述 
:----------- |:-------------
socket.AF_UNIX | 用于同一台机器上的进程通信（既本机通信）
socket.AF_INET | 用于服务器与服务器之间的网络通信
socket.AF_INET6 | 基于IPV6方式的服务器与服务器之间的网络通信
socket.SOCK_STREAM	| 基于TCP的流式socket通信
socket.SOCK_DGRAM | 基于UDP的数据报式socket通信

创建TCP Socket：
```sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)```

创建UDP Socket：
```sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)```

#### Socket 函数
* 服务器端 Socket 函数

socket 函数 | 描述    
:----------- |:-------------
s.bind(address) | 将套接字绑定到地址，在AF_INET下，以tuple(host, port)的方式传入，如s.bind((host, port))
s.listen(backlog) | 开始监听TCP传入连接，backlog指定在拒绝链接前，操作系统可以挂起的最大连接数，该值最少为1，大部分应用程序设为5就够用了
s.accept() | 接受TCP链接并返回（conn, address），其中conn是新的套接字对象，可以用来接收和发送数据，address是链接客户端的地址。

* 客户端 Socket 函数

socket 函数 | 描述    
:----------- |:-------------
s.connect(address) | 链接到address处的套接字，一般address的格式为tuple(host, port)，如果链接出错，则返回socket.error错误
s.connect_ex(address) | 功能与s.connect(address)相同，但成功返回0，失败返回errno的值

* 公用函数

socket 函数 | 描述    
:----------- |:-------------
s.recv(bufsize[, flag]) | 接受TCP套接字的数据，数据以字符串形式返回，buffsize指定要接受的最大数据量，flag提供有关消息的其他信息，通常可以忽略
s.send(string[, flag]) | 发送TCP数据，将字符串中的数据发送到链接的套接字，返回值是要发送的字节数量，该数量可能小于string的字节大小
s.recvfrom(bufsize[, flag]) | 接受UDP套接字的数据u，与recv()类似，但返回值是tuple(data, address)。其中data是包含接受数据的字符串，address是发送数据的套接字地址
s.sendto(string[, flag], address) | 发送UDP数据，将数据发送到套接字，address形式为tuple(ipaddr, port)，指定远程地址发送，返回值是发送的字节数
s.close() | 关闭套接字
s.getpeername() | 返回套接字的远程地址，返回值通常是一个tuple(ipaddr, port)
s.getsockname() | 返回套接字自己的地址，返回值通常是一个tuple(ipaddr, port)


#### 一个完整的例子


* 服务器端代码
```
import socket

HOST = '127.0.0.1'
PORT = 8001

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind((HOST, PORT))
s.listen(5)
print 'Server start at: %s:%s' %(HOST, PORT)
print 'wait for connection...'

while True:
    conn, addr = s.accept()
    print 'Connected by ', addr
    while True:
        data = conn.recv(1024)
        print data
		conn.send("server received you message.")
```
* 客户端代码
```
import socket
HOST = '127.0.0.1'
PORT = 8001

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((HOST, PORT))

while True:
    cmd = raw_input("Please input msg:")
    s.send(cmd)
    data = s.recv(1024)
    print data
```
#### SocketServer
SocketServer是对上面的过程进行了封装，使得网络开发更简单
```
from socketserver import BaseRequestHandler, TCPServer

class CustomHandler(BaseRequestHandler, TCPServer):
    def handler(self):
	    print('Got connection from', self.client_address)
	    while True:
	        msg = self.request.recv(8192)
	        print(msg)
	        self.request.send(msg)
  
serv = TCPServer(('', 2000), CustomHandler)
serv.serve_forever()
```

默认情况下上边的服务器是单线程的， 一次只能处理一个客户单。 如果要实现多线程， 可以使用TheadingTCPServer

```
from socketserver import ThreadingTCPServer
...

serv = ThreadingTCPServer(('', 2000), CustomHandler)
serv.serve_forever()
```

其实， 这样使用多线程有个问题。它会针对每一个客户端连接创建一个新的进程或线程。 但是允许连接的客户端数量是没有上限的， 这样服务器遭受恶意连接的时候会挂掉。

可以在代码里给线程数加个上限。

```
from threading import Thread
NWORKERS = 16
serv = TCPServer(('', 2000), CustomHandler)
for n in range(NWORKERS):
    t = Thread(target=serv.serv_forever)
    t.daemon = True
    t.start()
serv.serve_forever()
```
## 2. socks5 协议
SOCKS是一种网络传输协议，主要用于客户端与外网服务器之间通讯的中间传递。SOCKS是"SOCKetS"的缩写。

当防火墙后的客户端要访问外部的服务器时，就跟SOCKS代理服务器连接。这个代理服务器控制客户端访问外网的资格，允许的话，就将客户端的请求发往外部的服务器。

根据OSI模型，SOCKS是会话层的协议，位于表示层与传输层之间。
####一个完整的socks5 通信过程
*  握手阶段
* 建立连接
* 传输阶段

#####1. 握手阶段
1.客户端和服务器在握手阶段协商认证方式

 VER|NMETHODS | METHODS    
:----------- |:------------- |:-----------
1 | 1 | 1 - 255 

VER 表示版本号:sock5 为 X'05'
NMETHODS（方法选择）中包含在METHODS（方法）中出现的方法标识的数据（用字节表示）

  目前定义的METHOD有以下几种:
 > X'00'  无需认证
  X'01'  通用安全服务应用程序(GSSAPI)
  X'02'  用户名/密码 auth (USERNAME/PASSWORD)
  X'03'- X'7F' IANA 分配(IANA ASSIGNED) 
  X'80'- X'FE' 私人方法保留(RESERVED FOR PRIVATE METHODS) 
  X'FF'  无可接受方法(NO ACCEPTABLE METHODS) 

2.服务器在收到客户端的协商请求后，会检查是否有服务器支持的认证方式，并返回选中的认证方式
  服务器从客户端发来的消息中选择一种方法作为返回
  服务器从METHODS给出的方法中选出一种，发送一个METHOD（方法）选择报文：

VER | METHOD  
:----------- |:-------------
1 | 1

例如：对于shadowsocks
```
client -> ss: 0x05 0x01 0x00
ss -> client: 0x05 0x00
```


#####2. 建立连接

1.完成握手后，客户端会向服务器发起请求，请求的格式如下：

VER | CMD | RSV | ATYP | DST.ADDR | DST.PORT
:----------- |:------------- |:----------- |:------------- |:----------- |:-----------
1 | 1 | 0x00 | 1 | 动态 | 2

VER是SOCKS版本，这里应该是0x05；
CMD是SOCK的命令码
> 0x01表示CONNECT请求
0x02表示BIND请求
0x03表示UDP转发

RSV 0x00，保留
ATYP DST.ADDR类型
> 0x01 IPv4地址，DST.ADDR部分4字节长度
	0x03域名，DST ADDR部分第一个字节为域名长度，DST.ADDR剩余的内容为域名
	0x04 IPv6地址，16个字节长度。

DST.ADDR 目的地址
DST.PORT 网络字节序表示的目的端口


2. 服务器按以下格式回应客户端的请求（以字节为单位）：

VER | REP | RSV | ATYP | BND.ADDR | BND.PORT
:----------- |:------------- |:----------- |:------------- |:----------- |:-----------
1 | 1 | 0x00 | 1 | 动态 | 2

VER是SOCKS版本，这里应该是0x05；
REP应答字段
> 0x00表示成功
	0x01普通SOCKS服务器连接失败
	0x02现有规则不允许连接
	0x03网络不可达
	0x04主机不可达
	0x05连接被拒
	0x06 TTL超时
	0x07不支持的命令
	0x08不支持的地址类型
	0x09 - 0xFF未定义

RSV 0x00，保留
ATYP BND.ADDR类型
> 0x01 IPv4地址，DST.ADDR部分4字节长度
0x03域名，DST.ADDR部分第一个字节为域名长度，DST.ADDR剩余的内容为域名
0x04 IPv6地址，16个字节长度。

BND.ADDR 服务器绑定的地址
BND.PORT 网络字节序表示的服务器绑定的端口

##### 3. 传输阶段
SOCKS5 协议只负责建立连接，在完成握手阶段和建立连接之后，SOCKS5 服务器就只做简单的转发了。

## 3. shadowsocks 最早版本代码解析
shadowsocks 是非常有名的基于socks5 协议进行通讯的一个项目， 我们从技术的角度来分析下它的代码。 最新的代码加入了很多功能和优化， 不太容易理解， 所以我们从它最早的一个版本来入手。

最早版本的项目只有三个文件， readme、server.py, local.py 加起来也就两百行代码。很适合分析socks5的完整实现。

readme 介绍了使用方法， 我们略去。 
#### 先看server端的代码

```
PORT = 8499  # 绑定的端口号（地址是本机地址）
KEY = "foobar!"   # 最早版本的密码

import socket
import select
import SocketServer
import struct  # 用于二进制转换
import string
import hashlib

# get_table 函数用来实现加密和解密， 可以不用管实现细节。
def get_table(key):
    m = hashlib.md5.new()
    m.update(key)
    s = m.digest()
    (a, b) = struct.unpack('<QQ', s)
    table = [c for c in string.maketrans('', '')]
    for i in xrange(1, 1024):
        table.sort(lambda x, y: int(a % (ord(x) + i) - a % (ord(y) + i)))
    return table


# 常用的TCPServer实现方式
class ThreadingTCPServer(SocketServer.ThreadingMixIn, SocketServer.TCPServer):
    pass


# 核心部分
class Socks5Server(SocketServer.StreamRequestHandler):
    def handle_tcp(self, sock, remote):
        # 两个socket， 一个与客户端通讯（你电脑上的ss客户端）， 一个与目标服务器通讯（ex:google）
        fdset = [sock, remote]
        while True:
            r, w, e = select.select(fdset, [], [])  # select 是常用的异步socket 处理方法， 此处用来监听socket的读
            # 如果接收到客户端发送的数据， 读取并解密数据， 然后用remote socket发送到目标服务器
            if sock in r:
                if remote.send(self.decrypt(sock.recv(4096))) <= 0: break
            # 如果接收到目标服务器的数据， 加密发送给客户端
            if remote in r:
                if sock.send(self.encrypt(remote.recv(4096))) <= 0: break

    def encrypt(self, data):
        return data.translate(encrypt_table)

    def decrypt(self, data):
        return data.translate(decrypt_table)

    def send_encrpyt(self, sock, data):
        sock.send(self.encrypt(data))

    def handle(self):
        try:
            print 'socks connection from ', self.client_address
            sock = self.connection  # 获取已经建立的连接
            sock.recv(262)  # 接受客户端发来的认证方式的请求
            self.send_encrpyt(sock, "\x05\x00")  # socks5, 无认证
            data = self.decrypt(self.rfile.read(4))  # 读取四个字节（VER、CMD、RSV、ATYP）
            mode = ord(data[1])  # CMD
            addrtype = ord(data[3])  # ATYP
            if addrtype == 1:  # ipv4
                addr = socket.inet_ntoa(self.decrypt(self.rfile.read(4)))  # ipv4 地址占4个字节
            elif addrtype == 3:  # 域名
                addr = self.decrypt(self.rfile.read(ord(self.decrypt(sock.recv(1))))) # 域名的话， 第一个字节表示域名的长度， 后面几个字节是真正的域名的地址
            else:
                # not support
                return
            port = struct.unpack('>H', self.decrypt(self.rfile.read(2)))  # 两个字节的端口号
            reply = "\x05\x00\x00\x01"  # socks5, 成功连接， 保留， IPV4
            try:
                if mode == 1:  # connect
                    remote = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # 与目标服务器进行通讯的TCP socket
                    remote.connect((addr, port[0]))
                    local = remote.getsockname()
                    reply += socket.inet_aton(local[0]) + struct.pack(">H", local[1])  # 指明返回的地址和端口
                    print 'Tcp connect to', addr, port[0]
                else:
                    reply = "\x05\x07\x00\x01"  #socks5， 不支持请求包中的CMD， 保留， IPV4
                    print 'command not supported'
            except socket.error:
                # Connection refused
                reply = '\x05\x05\x00\x01\x00\x00\x00\x00\x00\x00'  # socks5, 连接拒绝
            self.send_encrpyt(sock, reply)
            if reply[1] == '\x00':  # 连接成功
                if mode == 1:
                    self.handle_tcp(sock, remote)
        except socket.error:
            print 'socket error'


def main():
    server = ThreadingTCPServer(('', PORT), Socks5Server)
    server.allow_reuse_address = True  # 允许服务器重新对之前使用过的端口号进行绑定
    print "starting server at port %d ..." % PORT
    server.serve_forever()

# 程序入口
if __name__ == '__main__':
    # 加密的table
    encrypt_table = ''.join(get_table(KEY))
    # 解密的table
    decrypt_table = string.maketrans(encrypt_table, string.maketrans('', ''))
    main()
``` 

逻辑非常清晰， 首先有两个加密解密的函数负责客户端和服务器的数据加密通讯， 然后handle函数负责客户端和服务器socks5 认证， 认证通过后建立连接， 并交给handle_tcp 函数进行传输数据的处理。
 
#### 接下来我们看看local.py 的代码
```
class Socks5Server(SocketServer.StreamRequestHandler):
    def encrypt(self, data):
        return data.translate(encrypt_table)

    def decrypt(self, data):
        return data.translate(decrypt_table)

    def handle_tcp(self, sock, remote):
        fdset = [sock, remote]
        counter = 0
        while True:
            r, w, e = select.select(fdset, [], [])  # 监听两个socket的读事件
            if sock in r:
            # 本地的socket的数据加密后发送给ss的服务器
                r_data = sock.recv(4096)
                if counter == 1:
                    try:
                        lock_print("Connecting " + r_data[5:5 + ord(r_data[4])])
                    except Exception:
                        pass
                if counter < 2:
                    counter += 1
                if remote.send(self.encrypt(r_data)) <= 0: break
            if remote in r:
            # ss服务器发送过来的数据解密后发送给浏览器的socks5代理客户端。
                if sock.send(self.decrypt(remote.recv(4096))) <= 0:
                    break

    def handle(self):
        try:
            sock = self.connection  # 和浏览器等进行通讯的socket
            remote = socket.socket() # 和ss服务器进行通讯的socket
            remote.connect((SERVER, REMOTE_PORT))
            self.handle_tcp(sock, remote) # 交给handle_tcp处理
        except socket.error:
            lock_print('socket error')
```

可以看出local的代码只是简单的做下数据的加密和转发，socks5 协议的认证和连接实际发生在你的浏览器和ss服务器之间。
另外， 能科学上网的socks5 通讯和普通的socks5 通讯的差别在于多了数据加密的那一步。

![socks5](/img/old-post/e1a9a11341c628164d32f8046b37214d8290.PNG)  

> shadowsocks最新版本的代码还在研读中，有机会的话继续在技术层面上分享。
##参考文章
[shadowsocks早版本代码](https://github.com/shadowsocks/shadowsocks/commit/833bd3de41024a70fe5be4c5480ae6fbce7fc0fa)
[python socket模块](https://gist.github.com/kevinkindom/108ffd675cb9253f8f71)
[shadowsocks源码结构](https://loggerhead.me/posts/shadowsocks-yuan-ma-fen-xi-xie-yi-yu-jie-gou.html)
[socks5 协议介绍](http://blog.csdn.net/suifengdeshitou/article/details/48782667)
[socks 协议](https://en.wikipedia.org/wiki/SOCKS)
Python CookBook 第三版