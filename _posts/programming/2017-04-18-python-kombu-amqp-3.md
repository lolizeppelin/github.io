---
layout: post
title:  "python kombu与amqp(3)"
date:   2017-04-14 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


来看看上一节kombu的connection中的transport

```python
# amqp.transport.TCPTransport
class TCPTransport(_AbstractTransport):

    def _setup_transport(self):
        # 找到落地socket.send了
        self._write = self.sock.sendall
        self._read_buffer = EMPTY_BUFFER
        # 落地socket.recv
        self._quick_recv = self.sock.recv

    def _read(self, n, initial=False, _errnos=(errno.EAGAIN, errno.EINTR)):
        # initial表示从头开始读
        recv = self._quick_recv
        # 处理分包年粘包的变量
        rbuf = self._read_buffer
        try:
            # 读取指定长度
            while len(rbuf) < n:
                try:
                    s = recv(n - len(rbuf))
                except socket.error as exc:
                    # 可以接受的socket错误
                    if exc.errno in _errnos:
                        # 抛出timeout错误
                        # 外部只有读包头才会设置initial=True
                        # 包头7字节中任意一个自己字节读取过程有错都把之前的部分抛掉
                        if initial and self.raise_on_initial_eintr:
                            raise socket.timeout()
                        # 否则继续读socket数据
                        continue
                    raise
                # 读到空返回,socket关闭
                if not s:
                    raise IOError('Socket closed')
                rbuf += s
        except:
            self._read_buffer = rbuf
            raise
        # self._read_buffer切片,切掉返回的帧数
        # 这里处理玩分包粘包返回完整协议帧
        result, self._read_buffer = rbuf[:n], rbuf[n:]
        return result


class _AbstractTransport(object):
    connected = False
    def __init__(self, host, connect_timeout=None,
                 read_timeout=None, write_timeout=None,
                 socket_settings=None, raise_on_initial_eintr=True, **kwargs):
        self.connected = True
        self.sock = None
        self.raise_on_initial_eintr = raise_on_initial_eintr
        self._read_buffer = EMPTY_BUFFER
        self.host, self.port = to_host_port(host)
        self.connect_timeout = connect_timeout
        self.read_timeout = read_timeout
        self.write_timeout = write_timeout
        self.socket_settings = socket_settings

    def connect(self):
        self._connect(self.host, self.port, self.connect_timeout)
        self._init_socket(
            self.socket_settings, self.read_timeout, self.write_timeout,
        )

    def read_frame(self, unpack=unpack):
        read = self._read
        # 初始化保存整帧变量, 不用''是兼容2、3
        read_frame_buffer = EMPTY_BUFFER
        try:
            # 读取帧头7个字节,从头开始读
            # 后面的True配合raise_on_initial_eintr
            # 是为了保证第一个包出错能截掉错误的数据
            frame_header = read(7, True)
            # 真帧数据包括原始包头
            read_frame_buffer += frame_header
            # 从帧头解析出帧长度
            frame_type, channel, size = unpack('>BHI', frame_header)
            # 最大帧长度超过int的上限
            # 差成两部分.....
            # 也就是说...要是长度超过2倍int上限就有问题
            if size > SIGNED_INT_MAX:
                part1 = read(SIGNED_INT_MAX)
                part2 = read(size - SIGNED_INT_MAX)
                payload = ''.join([part1, part2])
            else:
                # 读出包体
                payload = read(size)
            read_frame_buffer += payload
            # 在多读一个字节
            ch = ord(read(1))
        except socket.timeout:
            self._read_buffer = read_frame_buffer + self._read_buffer
            raise
        except (OSError, IOError, SSLError, socket.error) as exc:
            # Don't disconnect for ssl read time outs
            # http://bugs.python.org/issue10272
            if isinstance(exc, SSLError) and 'timed out' in str(exc):
                raise socket.timeout()
            if get_errno(exc) not in _UNAVAIL:
                self.connected = False
            raise
        # 包尾必须是206
        if ch == 206:  # '\xce'
            # 最后一个payload是普通字节
            # 所以把msg再封装的部分在channel的dispatch_method中
            # channel的id没有在客户端connect的时候设置
            # channel的id值是rabbit自己设置的？
            return frame_type, channel, payload
        else:
            raise UnexpectedFrame(
                'Received {0:#04x} while expecting 0xce'.format(ch))
```

下面我们来看看channel的dispatch_method

```python

# amqp.channel.Channel
class Channel(AbstractChannel):
    # 和amqp.connection.Connection继承自AbstractChannel
    _METHODS = {
    spec.method(spec.Channel.Close, 'BsBB'),
    spec.method(spec.Channel.CloseOk),
    spec.method(spec.Channel.Flow, 'b'),
    spec.method(spec.Channel.FlowOk, 'b'),
    ....
    }
    # 上面是个蛋碎的python2.7+语法糖
    # 结果是
    # set([method_t(method_sig=(20, 41), args=None, content=False),
    # method_t(method_sig=(20, 40), args='BsBB', content=False)])
    # 是一个set不是dict............简直卧槽
    _METHODS = {m.method_sig: m for m in _METHODS}
    # 上面的set转换成字典
    # {(20, 41): method_t(method_sig=(20, 41), args=None, content=False),
    # (20, 40): method_t(method_sig=(20, 40), args='BsBB', content=False)}
    # 上述写法倒是提供了一些参数处理的思路
    def __init__(self, connection,
                 channel_id=None, auto_decode=True, on_open=None):
        # 一般最终是在amqp.connection.Connection生成Channel实例
        # connection当然也就是amqp.connection.Connection
        # 一个connection对应多个Channel
        # ------------------------------
        # 初始化的时候指定了channel_id
        # 让connection将channel_id从可用id队列中移除
        # connection生成channel一般都不指定id
        if channel_id:
            connection._claim_channel_id(channel_id)
        # 让connection自动分配一个可用channel_id
        else:
            channel_id = connection._get_free_channel_id()
        AMQP_LOGGER.debug('using channel_id: %s', channel_id)
        # 父类中将调用_setup_listeners注册回调函数
        super(Channel, self).__init__(connection, channel_id)

    def _setup_listeners(self):
        # 这里可以看出
        self._callbacks.update({
            spec.Channel.Close: self._on_close,
            spec.Channel.CloseOk: self._on_close_ok,
            spec.Channel.Flow: self._on_flow,
            spec.Channel.OpenOk: self._on_open_ok,
            spec.Basic.Cancel: self._on_basic_cancel,
            spec.Basic.CancelOk: self._on_basic_cancel_ok,
            spec.Basic.Deliver: self._on_basic_deliver,
            spec.Basic.Return: self._on_basic_return,
            spec.Basic.Ack: self._on_basic_ack,
        })


    def open(self):
        # 生成channel实例后一般会立刻调用channel.open()
        if self.is_open:
            return
        # 发送
        return self.send_method(
            spec.Channel.Open, 's', ('',), wait=spec.Channel.OpenOk,
        )

    def dispatch_method(self, method_sig, payload, content):
        # connection最后是调用channel的dispatch_method来分发数据
        if content and \
                self.auto_decode and \
                hasattr(content, 'content_encoding'):
            try:
                content.body = content.body.decode(content.content_encoding)
            except Exception:
                pass
        try:
            # 匹配到对应的amqp_method
            # 返回值是类似method_t(method_sig=(20, 40), args='BsBB', content=False)
            # 的对象
            amqp_method = self._METHODS[method_sig]
        except KeyError:
            raise AMQPNotImplementedError(
                'Unknown AMQP method {0!r}'.format(method_sig))
        # 判断下method_sig是不是Channel专用的
        # channel专用的的method_sig看_setup_listeners中注册的回调
        # 一般是channel初始化用的和关闭的时候用的
        try:
            listeners = [self._callbacks[method_sig]]
        except KeyError:
            listeners = None
        # 大部分操作都会走这里
        try:
            # 从_pending中弹出回调方法
            # 关键点在于_pending里有什么
            one_shot = self._pending.pop(method_sig)
        except KeyError:
            # _pending找不到,也不是channel默认方法
            if not listeners:
                return
        else:
            if listeners is None:
                listeners = [one_shot]
            else:
                listeners.append(one_shot)
        args = []
        # amqp_method.args是反序列化用的格式化字符串
        if amqp_method.args:
            args, _ = loads(amqp_method.args, payload, 4)
        # amqp_method的content为True
        # 插入函数入口传入的content
        if amqp_method.content:
            args.append(content)
        # 顺序调用回调
        for listener in listeners:
            listener(*args)
    # -----------------------------------------------------#
    # 绑定消费者的过程依次调用下面的函数
    # 1 exchange_declare 声明交换机
    # 2 queue_declare    声明队列
    # 3 queue_bind      队列绑定到交换机(传入routing_key)
    # 4 basic_consume   消费者订阅/绑定队列
    # 外部无论如何封装都是到channel中调用上述方法
    # 也就是说channel必须被创建
    # -----------------------------------------------------#
    def exchange_declare(self, exchange, type, passive=False, durable=False,
                         auto_delete=True, nowait=False, arguments=None,
                         argsig='BssbbbbbF'):
         if auto_delete:
             warn(VDeprecationWarning(EXCHANGE_AUTODELETE_DEPRECATED))

         self.send_method(
             spec.Exchange.Declare, argsig,
             (0, exchange, type, passive, durable, auto_delete,
              False, nowait, arguments),
             wait=None if nowait else spec.Exchange.DeclareOk,
         )

    def queue_declare(self, queue='', passive=False, durable=False,
                      exclusive=False, auto_delete=True, nowait=False,
                      arguments=None, argsig='BsbbbbbF'):

        self.send_method(
            spec.Queue.Declare, argsig,
            (0, queue, passive, durable, exclusive, auto_delete,
             nowait, arguments),
        )
        if not nowait:
            return queue_declare_ok_t(*self.wait(
                spec.Queue.DeclareOk, returns_tuple=True,
            ))

    def queue_bind(self, queue, exchange='', routing_key='',
                   nowait=False, arguments=None, argsig='BsssbF'):
       return self.send_method(
           spec.Queue.Bind, argsig,
           (0, queue, exchange, routing_key, nowait, arguments),
           wait=None if nowait else spec.Queue.BindOk,
       )

    def basic_consume(self, queue='', consumer_tag='', no_local=False,
                     no_ack=False, exclusive=False, nowait=False,
                     callback=None, arguments=None, on_cancel=None,
                     argsig='BssbbbbF'):
       # 声明消费者
       p = self.send_method(
           spec.Basic.Consume, argsig,
           (0, queue, consumer_tag, no_local, no_ack, exclusive,
            nowait, arguments),
           wait=None if nowait else spec.Basic.ConsumeOk,
       )

       if not nowait and not consumer_tag:
           consumer_tag = p

       self.callbacks[consumer_tag] = callback

       if on_cancel:
           self.cancel_callbacks[consumer_tag] = on_cancel
       if no_ack:
           self.no_ack_consumers.add(consumer_tag)
       return p
    # -----------------------------------------------------#
    # 我们来看看send_method
    # 这部分是父类方法
    # 所以amqp.connection.Connection的send_method也是这个
    def send_method(self, sig,
                    format=None, args=None, content=None,
                    wait=None, callback=None, returns_tuple=False):
        # 这玩意是vine模块的
        p = promise()
        # amqp.channel.Channel的connection是amqp.connection.Connection
        # amqp.connection.Connection的connection自身(self)
        conn = self.connection
        if conn is None:
            raise RecoverableConnectionError('connection already closed')
        # 如果有format传入,用format解析args
        # 就使用第二个参数解析第三个参数
        args = dumps(format, args) if format else bytes_if_py2('')
        try:
            # 通过connection回包
            conn.frame_writer(1, self.channel_id, sig, args, content)
        except StopIteration:
            raise RecoverableConnectionError('connection already closed')
        # TODO temp: callback should be after write_method ... ;)
        if callback:
            p.then(callback)
        p()
        if wait:
            # 需要等待rabbit的回复确认
            # 调用等待函数
            return self.wait(wait, returns_tuple=returns_tuple)
        return p

    def wait(self, method, callback=None, timeout=None, returns_tuple=False):
        p = ensure_promise(callback)
        pending = self._pending
        prev_p = []
        if not isinstance(method, list):
            method = [method]

        for m in method:
            # 从self._pending获取到对应的回调
            prev_p.append(pending.get(m))
            pending[m] = p

        try:
            while not p.ready:
                self.connection.drain_events(timeout=timeout)

            if p.value:
                args, kwargs = p.value
                return args if returns_tuple else (args and args[0])
        finally:
            for i, m in enumerate(method):
                if prev_p[i] is not None:
                    pending[m] = prev_p[i]
                else:
                    pending.pop(m, None)