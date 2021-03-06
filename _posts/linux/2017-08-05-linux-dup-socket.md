---
layout: post
title:  "Linux dup socket的原因"
date:   2017-08-05 15:05:00 +0800
categories: "系统基础"
tag: ["linux"]
---

* content
{:toc}


openstack的多进程wsgi服务用了如下方式


```python
def start(self):
    """Start serving a WSGI application.

    :returns: None
    """

    self.dup_socket = self.socket.dup()

    wsgi_kwargs = {
        'func': eventlet.wsgi.server,
        'sock': self.dup_socket,
        'site': self.app,
        'protocol': self._protocol,
        'custom_pool': self._pool,
        # 'log': self._logger,
        'log': LOG,
        'log_format': self.conf.wsgi_log_format,
        'debug': False,
        'keepalive': self.conf.wsgi_keep_alive,
        'socket_timeout': self.client_socket_timeout
        }

    if self._max_url_len:
        wsgi_kwargs['url_length_limit'] = self._max_url_len

    self._server = eventlet.spawn_n(**wsgi_kwargs)
```

self.socket是init的时候就bind listen好的,start里的代码是子进程里执行的

比较好奇的是为什子进程要用dup的socket而不直接用self.socket

英文的说明如下

    # The server socket object will be closed after server exits,
    # but the underlying file descriptor will remain open, and will
    # give bad file descriptor error. So duplicating the socket object,
    # to keep file descriptor usable.

    意思是主进程退出会导致socket关闭？
    按照道理只要还有进程的fd指向文件表...文件表就不会被关闭
    其他进程的fd和文件表照样打开的呀  


关闭sock的地方在eventlet.wsgi.server中

```python
finally:
    pool.waitall()
    serv.log.info("(%s) wsgi exited, is_accepting=%s" % (
        serv.pid, is_accepting))
    try:
        # NOTE: It's not clear whether we want this to leave the
        # socket open or close it.  Use cases like Spawning want
        # the underlying fd to remain open, but if we're going
        # that far we might as well not bother closing sock at
        # all.
        sock.close()
    except socket.error as e:
        if support.get_errno(e) not in BROKEN_SOCK:
            traceback.print_exc()
```

我估计问题就出在这里

    如果没有复制的socket的fd,当前进程内又显示调用了socket.close
    则会会触发监听关闭,导致其他进程虽然fd和文件表都在,但是监听已经被关闭了
