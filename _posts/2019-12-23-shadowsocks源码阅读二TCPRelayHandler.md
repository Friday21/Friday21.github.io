---
layout:     post
title:      shadowsocks源码阅读二TCPRelayHandler
date:       2019-12-23
author:     "FridayLi"
catalog: true
tags:
  - 源码
  - Python
---

上一章我们分析到， 对于新建立的socket连接， 会实例化一个新的TCPRelayHandler， 对于已有的socket连接读写活动， 则全部转发到对应socket的TCPRelayHandler 的handle_event处理， 那么TCPRelayHandler是如何处理socks5连接过程以及如果透明代理客户端和目标服务器的呢？
## 不同的Stage及其对应的处理函数
首先， 我们先分析下源码中对当前socket连接定义的几个状态——stage
先看下源码中的一段注释：
```Python
# as sslocal:
# stage 0 auth METHOD received from local, reply with selection message ——STAGE_INIT
# stage 1 addr received from local, query DNS for remote ——STAGE_ADDR
# stage 2 UDP assoc ——STAGE_UDP_ASSOC
# stage 3 DNS resolved, connect to remote ——STAGE_DNS 
# stage 4 still connecting, more data from local received ——STAGE_CONNECTING
# stage 5 remote connected, piping local and remote ——STAGE_STREAM

# as ssserver:
# stage 0 just jump to stage 1
# stage 1 addr received from local, query DNS for remote
# stage 3 DNS resolved, connect to remote
# stage 4 still connecting, more data from local received
# stage 5 remote connected, piping local and remote
```
sslocal 为shadowsocks (ss)的客户端， ssserver是ss的服务端， 在源码中用is_local的True 或 False来区分。
对于socks5建立连接的过程， 可以简单说是这几步：
1. Client建立与Server之间的连接， client告诉server自己的socks版本， 以及支持的认证方法。
2. Server返回可以使用的方法。这里ss返回的是0， 也就是不需要认证。（ss对data有自己的加密方法）
3. 客户端告知目标地址
4. 服务端回复是否连接成功
5. 开始传输流量（透明代理）

对比stage可以看出， stage 0 对应socks5连接的第一步和第二步， stage 1、2对应socks5连接第三步， stage 3对应第四步， stage 5对应第五步， stage 4则是一个中间态， 这时候已经告诉客户端建立连接成功， 但实际上和目标服务器之间的连接还在建立中， 这时候客户端发来的数据会先缓存起来， 放到`self._data_to_write_to_remote`中， 等和目标服务器建立连接成功后再发送过去。

对于每一种状态下数据读写的处理， 都有对应的函数去处理：
```shell
STAGE_INIT —— _handle_stage_init
STAGE_ADDR —— _handle_stage_addr
STAGE_UDP_ASSOC ——对应UdpRelay中的方法，这里不做说明
STAGE_DNS ——  不做处理， 继续等待dns解析
STAGE_CONNECTING —— _handle_stage_connecting
STAGE_STREAM —— _handle_stage_stream
STAGE_DESTROYED —— destroy
```

接下来， 我们一个个看这几个函数的具体处理过程
### _handle_stage_init
```Python
    def _handle_stage_init(self, data):
        self._check_auth_method(data)  # 检查客户端支持的认证方法
        self._write_to_sock(b'\x05\00', self._local_sock)  # socks 版本5， 认证方式为不需要认证
        self._stage = STAGE_ADDR
```
处理STAGE_INIT状态, socks5 连接第二步, 检查socks5版本和认证方法并返回给socks5客户端选中的socks5认证方法（0：不需要认证）。 状态变更为STAGE_ADDR。
注意， 这个方法只有shadowsocks客户端会用， 用来和本地浏览器交流， ss 服务端直接跳过了这一步（毕竟ss客户端和服务端可以直接默认认证方式）

### _handle_stage_addr
```Python
    def _handle_stage_addr(self, data):
        if self._is_local:
            cmd = common.ord(data[1])
            if cmd == CMD_UDP_ASSOCIATE:
                ...
            elif cmd == CMD_CONNECT:  # socks5第三步， 建立连接的命令
                # just trim VER CMD RSV
                data = data[3:]
            else:
                logging.error('unknown command %d', cmd)
                self.destroy()
                return
        header_result = parse_header(data)  # 获取请求的URL地址（根据socks5 协议）
        addrtype, remote_addr, remote_port, header_length = header_result  # 解析客户端传过来的请求连接的地址
        self._remote_address = (common.to_str(remote_addr), remote_port)
        # pause reading
        self._update_stream(STREAM_UP, WAIT_STATUS_WRITING)
        self._stage = STAGE_DNS  # 更新stage状态
        if self._is_local:  # sslocal
            # jump over socks5 response
            if not self._is_tunnel:
                # forward address to remote
                # TODO  returning BND.ADDR with all zeros will tell the SOCKS5 client to connect to the SOCKS5
                #  server as the relay(中继) server.
                self._write_to_sock((b'\x05\x00\x00\x01'  # 连接成功
                                     b'\x00\x00\x00\x00\x10\x10'),  # server绑定的地址和端口是 0.0.0.0:4112 (端口不重要)
                                    self._local_sock)

            data_to_send = self._cryptor.encrypt(data)
            self._data_to_write_to_remote.append(data_to_send)
            # notice here may go into _handle_dns_resolved directly（如果配置里ss服务端的地址是IP的话）
            self._dns_resolver.resolve(self._chosen_server[0], self._handle_dns_resolved)  #  解析的是ss服务端的地址
        else:  ## ssserver
            if len(data) > header_length:
                self._data_to_write_to_remote.append(data[header_length:])
            # notice here may go into _handle_dns_resolved directly
            self._dns_resolver.resolve(remote_addr, self._handle_dns_resolved)  # 解析的是目标服务器的地址
```
可以看到， socks5建立连接的第四步只发生在sslocal（ss客户端）和本地浏览器之间， ss服务端则省掉了这些过程， 直接处理ss客户端发来的数据。对于目标地址解析，ss客户端只需要从配置文件中选出一个ss服务端地址来解析即可，而ss服务端需要调用dns解析真实的目标服务器地址。 dns解析成功后会回调`self._handle_dns_resolved` 函数

### _handle_dns_resolved
```Python
 def _handle_dns_resolved(self, result, error):
        ip = result[1]
        self._stage = STAGE_CONNECTING  # stage 更新为CONNECTING
        remote_addr = ip
        if self._is_local:
            remote_port = self._chosen_server[1]
        else:
            remote_port = self._remote_address[1]

        # do connect
        remote_sock = self._create_remote_socket(remote_addr, remote_port)
        remote_sock.connect((remote_addr, remote_port))
        self._loop.add(remote_sock,
                       eventloop.POLL_ERR | eventloop.POLL_OUT,
                       self._server)  # 调用connect之后（非阻塞），监听remote的可写事件
        self._update_stream(STREAM_UP, WAIT_STATUS_READWRITING)
        self._update_stream(STREAM_DOWN, WAIT_STATUS_READING)
```
逻辑还是比较清晰得， 解析出来要连接的目标服务器ip后， 创建remote_sock， connect， 并监听remote_sock的可读事件。这里要注点意的是调用connect是非阻塞的， 因为remote_sock设置了block=False. 另外， 对于ss客户端来说， remote_sock要连接的是ss服务端， 对于ss服务端remote_sock要连接的是真实的目标服务器。

### _handle_stage_connecting
前面已经提过， 在stage为STAGE_CONNECTING状态下， 如果有客户端数据过来， 则存储在  `self._data_to_write_to_remote`中， 等remote_sock connect成功后再将数据发送出去。

### _handle_stage_stream
stage在STAGE_STREAM状态下， ss客户端和ss服务端都只是简单的将下游的数据原封不断的转发给上游， 将上游的数据转发给下游。需要注意的是， ss客户端和ss服务端之间传输数据时， 会对data加密。

## handle_event 和stage的联系
弄懂了每个stage下对应的handler， 那么， handle_event是怎么具体处理整个流程的呢？其实比较简单， 我们前面知道event_loop监控的socket有读写活动发生时会最终调到TCPRelayHandler的handle_event， 所以handle_event处理的第一步是先判断有活动的socket是remote_sock还是local_sock, remote_sock就是上边_handle_dns_resolved所创建的， 而locol_sock是TCPRelay在和客户端建立连接成功后所创建的socket。根据不同的socket以及对应的活动是读事件还是写事件会分别调用`_on_remote_write`、 `_on_remote_read`、 `_on_local_read`、 `_on_local_write`函数， `_on_local_write` 和 `_on_remote_write`都比较简单， 只要把内存中的数据写入对应的socket就行了，`_on_remote_read` 用来读取上游的数据并写入下游（local_sock)， `_on_local_read`逻辑会多一点， stage不同handler的分发其实发起点是这里，因为在其他三个活动状态下， 基本上都已经是STAGE_STREAM状态了，不用再做不同stage的处理， 只用透明传输即可。而local_sock可读则在各种stage状态下都有可能。

这几个函数中需要额外注意的两点是：
1. `_on_remote_write`中第一行会把stage更新为STAGE_STREAM， 这是因为remote_sock可写， 一定是已经成功建立了连接， 已经是对应stage的STAGE_STREAM了。
2. `_on_remote_read` 和 `_on_local_read` 中都有处理socket关闭的逻辑：
```Python
data = self._local_sock.recv(buf_size)
if not data:  # 对端关闭链接
    # When recv returns a value of 0 that means the connection has been closed.
    # These calls return the number of bytes received, or -1 if an error occurred.
    # The return value will be 0 when the peer has performed an orderly shutdown.
    logger.error('没有data(客户端断掉连接)， 执行销毁')
    self.destroy()
    return
```
当recv返回的data为空时， 代表对端关闭了连接， 这时候要执行本socket的销毁操作。

## UP_STREAM 和 DOWN_STREAM
代码中可以在很多地方看到调用`_update_stream`的操作， 那么， 这是用来做什么的呢？
我们先来看看上下游的定义：
```Python
# for each handler, we have 2 stream directions:
#    upstream:    from client to server direction
#                 read local and write to remote
#    downstream:  from server to client direction
#                 read remote and write to local
```
用一张图表示的话如下：
![描述](/img/old-post/3da7955429486cdc839d630907fcdaae9964.PNG)

对于ss客户端来说（sslocal）,  up_stream是浏览器向ss客户端传输数据或者ss客户端向ss服务端传输数据。down_stream是ss服务器向ss客户端传输数据或者ss客户端向浏览器传输数据。
对于ss服务端来说（ssserver）， up_stream是ss客户端向ss服务端传输数据或者ss服务端向目标服务器传输数据。down_stream是目标服务器向ss服务器传输数据或者ss服务器向ss客户端传输数据。
我们看下`_update_stream`的具体代码：
```Python
def _update_stream(self, stream, status):
    dirty = False
    if stream == STREAM_DOWN:
        logger.info('[流程观察TCPRelayHandler-_update_stream]更新STREAM_DOWN为{}, TCPRelayHandler:{}'.format(
            stream_desc.get(status), id(self)))
        if self._downstream_status != status:
            self._downstream_status = status
            dirty = True
    elif stream == STREAM_UP:
        logger.info('[流程观察TCPRelayHandler-_update_stream]更新STREAM_UP为{}, TCPRelayHandler:{}'.format(
            stream_desc.get(status), id(self)))
        if self._upstream_status != status:
            self._upstream_status = status
            dirty = True
    if not dirty:
        return

    if self._local_sock:
        event = eventloop.POLL_ERR
        if self._downstream_status & WAIT_STATUS_WRITING:
            event |= eventloop.POLL_OUT
        if self._upstream_status & WAIT_STATUS_READING:
            event |= eventloop.POLL_IN
        self._loop.modify(self._local_sock, event)
    if self._remote_sock:
        event = eventloop.POLL_ERR
        if self._downstream_status & WAIT_STATUS_READING:
            event |= eventloop.POLL_IN
        if self._upstream_status & WAIT_STATUS_WRITING:
            event |= eventloop.POLL_OUT
        self._loop.modify(self._remote_sock, event)
```
主要作用是根据更改之后的状态更新event loop监听的socket事件类型。根据上图定义的up_stream和down_stream， 可以很容易的读懂本函数的逻辑：
1. 如果更新前后的`stream_status`不变， 则什么都不做（废话。。。）
2. 如果`self._downstream` 更新为等待写， 则监听`self.locol_sock`的可写事件（POLL_OUT）；如果`self._downstream` 更新为等待读， 则监听self.remote_sock的可读事件（POLL_IN）。下游数据的流动是从`remote_sock`流向`locol_sock`
3. 如果``self._upstream``更新为等待读， 则监听``self.locol_sock``的可读事件（POLL_IN）。如果``self._upstream``更新为等待写， 则监听``self.remote_sock``的可写事件（POLL_OUT）。上游数据的流动是从``local_sock``流向``remote_sock``

那么， 为什么要频繁改动event_loop监听socket的事件类型呢？一直监听每个socket的读写事件不行吗？其实不行，如果remote_sock一直不可写（不可以发送到当前服务器的上游）， 这时候如果还一直监听local_sock的可读事件， 就会源源不断的把下游来得数据缓存到内存（`_data_to_write_to_remote`）里， 直到内存被占满， 服务崩溃。另外， 当你刚把下游的请求转发到上游服务器后， 你应该期待的是从上游服务器读取响应数据并转发到下游， 而不是继续接受下游的数据转发到上游服务器（万一上游服务器故障了呢？即使不故障， 在响应返回之前也不太应该继续请求更多的数据）。 如果我们不监听某个socket的可读事件， 那么对应socket的另一端的写缓冲区就会阻塞， 那么这时候另一端自然就不会发送更多的数据了（TCP协议决定的）， 这样， 就真正的实现了透明代理，代理服务器尽量不缓存太多的临时数据。