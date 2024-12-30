---
layout:     post
title:      shadowsocks源码阅读一整体流程
date:       2019-12-08
author:     "FridayLi"
catalog: true
tags:
  - 源码
  - Python
---

最近刚读完《Unix 网络编程 卷1》， 对HTTP的更底层——TCP及socket编程有了更多的了解， 在此基础上重新开始研究起shadowsocks的源码， 之前一知半解的地方现在看来豁然开朗， 感觉打通了任督二脉一样，忍不住想写篇blog来分享下我对其整个工作原理的理解。当然， 任何源码阅读都是有门槛的，想要真正读懂shadowsocks源码， 最起码要具备以下知识背景：

1. Python 基础语法
2. socket 基本编程知识， 懂得几个相关的基础函数用法（listen、bind、accept等）—— [Python socket 基础](https://gist.github.com/kevinkindom/108ffd675cb9253f8f71)
3. 了解I/O 多路复用 （select、poll、epoll）
4. 了解socks 5协议， 知道socks 5协议传输数据的几个步骤 ——[socks5 协议介绍](https://jiajunhuang.com/articles/2019_06_06-socks5.md.html)

## shadowsocks 和普通socks 5 server的区别
首先， shadowsocks是基于socks 5协议开发的一个服务， 那么他和普通的socks 5 server有什么区别呢？ 
要回答这个问题， 我们首先要了解socks 5协议出现的背景，解决了一个怎样的问题。
根据维基百科： “SOCKS工作在比HTTP代理更低的层次：SOCKS使用握手协议来通知代理软件其客户端试图进行的连接SOCKS，然后尽可能透明地进行操作。” 
 “Bill希望通过互联网与Jane沟通，但他们的网络之间存在一个防火墙，Bill不能直接与Jane沟通。所以，Bill连接到他的网络上的SOCKS代理，告知它他想要与Jane创建连接；SOCKS代理打开一个能穿过防火墙的连接，并促进Bill和Jane之间的通信。”

所以， socks协议出现的本身就是为了突破防火墙对客户端A访问某个服务端B的限制，做法是通过第三方服务器C，来打开一个透明代理， 来达到间接访问服务器B的目的。这里有个前置条件， 那就是A能顺畅的访问C， C也能顺畅的访问B。所以， 如果防火墙只是单纯的切断你直接访问B的所有请求的话， 普通的socks 代理服务器也是可以达到翻墙的效果的， 但随着GFW的发展， 防火墙不仅墙掉了直接访问B的请求， 甚至像VPN、socks 代理这种能够根据代理请求特征数据判断出你最终想要访问的服务器B， 也一样会切断你访问C的所有请求。

那么， shadowsocks是如何做到不被检测到代理请求的地址的呢？请看下图：

![描述](/img/old-post/6496463e500457711f058d077fdd9f489972.PNG)

shadowsocks 和普通socks代理相比多了一步， 把socks代理服务器拆分成了ss客户端和ss服务端， ss客户端负责和本地浏览器（其它客户端也可以）进行常规的socks 5通信， 同时， ss客户端把socks 5 通信内容通过对称加密， 把数据传输到远端服务器的ss服务端， ss服务端解密数据后解析出要连接的远端地址， 连接要访问的服务器， 成功后， ss客户端和ss服务端整体作为一个透明代理， 搭建起本地浏览器和目标服务器之间的桥梁。

## 整体运转流程
服务端同学面试时有时会被问到这样一个问题： 当你在浏览器打开一个网页时， 其背后发生了哪些事情？
在这里， 我们也从同样的层面来探讨下当用户在浏览器打开google网站时， 数据在shadowsocks内部是怎么传输的呢？

首先， 我们先看下shadowsocks 源代码的几个重要文件：
```Python
├── __init__.py
├── asyncdns.py    #DNS 解析
├── common.py  
├── cryptor.py  # 加密解密
├── daemon.py
├── eventloop.py  # socket 读写事件监控
├── local.py  # 本地服务启动点
├── manager.py
├── server.py  # 远端服务启动点
├── shell.py
├── tcprelay.py  # TCP数据传输核心处理逻辑
└── udprelay.py
```
我们以shadowsocks 服务端为例， 我们启动的命令是 `ssserver -c config.json start` , 其实就是执行的server.py 的main函数， 我们看看这个函数都做了些什么(省略掉了部分不重要的代码)：

```Python
def main():
    config = shell.get_config(False)  # 根据配置文件生成配置选项
    ...
    tcp_servers = []
    udp_servers = []
    port_password = config['port_password']
    del config['port_password']
    for port, password in port_password.items():
        a_config = config.copy()
        a_config['server_port'] = int(port)
        a_config['password'] = password
        logging.info("starting server at %s:%d" %
                     (a_config['server'], int(port)))
        logger.info('[流程观察server-main]创建TCPRelay, UDPRelay, EventLoop')
        tcp_servers.append(tcprelay.TCPRelay(a_config, dns_resolver, False))  # 根据配置创建了TCPServer
        udp_servers.append(udprelay.UDPRelay(a_config, dns_resolver, False))  ## 根据配置创建了UDPServer

    def run_server():
        ...
        try:
            loop = eventloop.EventLoop()  # 创建主循环loop
            dns_resolver.add_to_loop(loop)
            list(map(lambda s: s.add_to_loop(loop), tcp_servers + udp_servers))  # 将之前创建的TCPServer 和 UDPServer 所监听的socket加入loop循环， 监听对应的读事件（对应的socket已经调用了listen方法）
            logger.info('[流程观察server-main]loop run, 阻塞在监听socket读写事件')
            loop.run()
        except Exception as e:
            shell.print_exception(e)
            sys.exit(1)
        ...  
       run_server()
```

所以， 进程最终落脚在loop.run() 函数里， 我么进入eventloop.py 看看里边发生了什么。
eventloop里封装了几个I/O多路复用的方法：Select、Kqueue、epoll。会根据当前的系统环境来选取最合适的方法。优先级依次是：epoll > Kqueue > Select 。其中epoll是linux系统专用， windows和mac都没有，select是跨平台的， 但性能不如epoll。kqueue 实现和epoll类似， 而且支持mac（FreeBSD）。

这里先不展开讲他们之间的区别， 我们先把他们看成一个统一的select模块， select模块在这里的作用是能够在单进程单线程的条件下， 同时监控多个socket的读写事件， 当某个socket可读或可写时， 就激活这个socket，执行对应的处理函数。我们看下对应的run函数：
```Python
  def run(self):
        events = []
        while not self._stopping:
            asap = False
            try:
                events = self.poll(TIMEOUT_PRECISION)  # 主循环阻塞在这里
            except (OSError, IOError) as e:
                ...

            for sock, fd, event in events:
                handler = self._fdmap.get(fd, None)  # 取出对应的描述符（fd)对应的处理函数handler
                if handler is not None:
                    handler = handler[1]
                    try:
                        handler.handle_event(sock, fd, event)  # 调用函数处理激活的socket
                    except (OSError, IOError) as e:
                        shell.print_exception(e)
```

这里可以看出， eventloop做的工作就是注册要监听的socket（并建立描述符和回调函数的映射关系）， 监听socket的读写事件， 有活动了， 就找到对应socket的处理函数，处理对应事件。
这里， sock是我们要监听的socket， fd是socket对应的描述符， event是对应的读写事件。

这次， 落脚点在handler.handle_event 这里， 对tcp连接来说，其实就是tcprelay.py文件里的TCPRelay.handle_event， 我们继续往下挖。

我们知道， 在最开始的server.py 的main函数中，会根据配置创建TCPServer， 其实就是实例化tcprelay.py中的一个TCPRelay， 我们先看看TCPRelay的`__init__`函数.

```Python
class TCPRelay(object):

    def __init__(self, config, dns_resolver, is_local, stat_callback=None):
        ...  # 配置处理
        server_socket = socket.socket(af, socktype, proto)  # 创建一个 server socket
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server_socket.bind(sa)  # 绑定端口
        server_socket.setblocking(False)  # socket设置为非阻塞
        server_socket.listen(1024)  # 开始listen客户端的连接
        self._server_socket = server_socket
        self._stat_callback = stat_callback
        logger.info('[流程观察TCPRelay-init]创建TCPRelay, 监听（listen）{}'.format(sa))
```

可以看出来， TCPServer示例化后已经创建了一个server socket， 并处于监听连接的状态了。接下来我们看下handle_event里发生了什么：

```Python
    def handle_event(self, sock, fd, event):
        # handle events and dispatch to handlers
        if sock:
            logging.log(shell.VERBOSE_LEVEL, 'fd %d %s', fd,
                        eventloop.EVENT_NAMES.get(event, event))
        if sock == self._server_socket:
            try:
                logging.debug('accept')  # server socket 可读代表有连接过来， 这时候调用accpet方法与客户端建立连接。
                conn = self._server_socket.accept()
                # 对每一个新连接， 实例化一个TCPRelayHandler，并加入到eventloop中
                h = TCPRelayHandler(self, self._fd_to_handlers,
                                    self._eventloop, conn[0], self._config,
                                    self._dns_resolver, self._is_local)
                logger.info('[流程观察TCPRelay-handle_event]accept 客户端连接， 创建TCPRelayHandler:{}'.format(id(h)))
            except (OSError, IOError) as e:
                ...
        else:
            if sock:  # 非server socket的活动， 代表是已经建立连接的socket， 这时候要调用对应的处理方法去处理， 也就是TCPRelayHandler的handle_event方法。
                handler = self._fd_to_handlers.get(fd, None)
                logger.info('[流程观察TCPRelay-handle_event]socket:{}有活动，交给handler——TCPRelayHandler:{}处理'.format(
                    id(sock), id(handler)))
                if handler:
                    handler.handle_event(sock, event)
            else:
                logging.warn('poll removed fd')
```

可以看到， 对于socket 的读写事件的处理， TCPRelay是区别对待的，如果是自己的server socket可读了（只监听了读事件）， 就代表有新的客户想要建立连接， 这时候就调用accpet方法建立连接， 连接建立后会生成一个新的socket —— conn， 用来处理和客户端的数据传输， 并示例化一个TCPRelayHandler， 把conn对应的事件处理都交给TCPRelayHandler。 如果sock不是自己的server socket， 其实就代表sock是之前创建的conn中的一个， 直接交给对应实例的TCPRelayHandler.handel_event处理就好。

这里有个需要注意的事情是：TCPRelay在整个服务中只有一个实例， 负责处理监听事件； 而TCPRelayHandler有多个， 每有一个新的连接创建， 就会实例化一个TCPRelayHandler， 绑定对应的socket，并负责之后的数据传输。

TCPRelayHandler 的处理逻辑比较多， 细节会放到下一章去讲， 简单来说就是负责建立socks 5代理， 转发客户端和目标服务器通信的数据， 并在请求处理完之后销毁自身。

### 总结
至此， 整个流程就算走通了，流程图可以大致整理如下：

![描述](/img/old-post/a551782063f8bf89d3e89bbec4c6fbb19740.JPEG)

1. server.py 的main函数启动服务， 创建TCPServer(TCPRelay)（1）, 创建TCPServer的过程中会 根据配置生成服务端server_socket, 绑定监听端口， 并放入到loop监听读事件（2）。最后， 阻塞在loop的大循环中（3）。
2. eventloop.py的run函数中， 会不停扫描有活动（event）的socket（4）， 一旦有活动， 就取出对应的回调函数（TCPRelay.handle_event）进行处理（5）
3. tcprelay.py文件中的TCPRelay.handle_event根据当前激活的socket来进行不同处理， 如果socket是当前服务器用来监听连接的socket， 代表着有新的连接过来， 调用accept方法建立新的连接， 创建对应的用来处理新连接数据传输的socket——conn，实例化一个TCPRelayHandler, 绑定conn， 并将conn放入loop中， 监听后续的读写事件（6）。如果当前激活的socket是之前创建的conn中的一个， 则直接交给对应绑定的TCPRelayHandler.handle_event 处理（7）
4. tcprelay.py 中的TCPRelayHandler.handle_event会根据当前的状态来对新激活的socket做不同的处理， 包括建立socks5连接， dns解析， 连接目标服务器，透明代理客户端和目标服务器等。等客户端关闭了连接， 则销毁自身。


对于第4步的具体内容， 我们单独放到一章中去讲解。

