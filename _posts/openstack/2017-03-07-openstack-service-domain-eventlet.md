---
layout: post
title:  "OpenStack Mitaka从零开始 openstack的守护进程启动过程"
date:   2017-03-07 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python", "linux"]
---

* content
{:toc}

openstack的服务都和eventlet有关,建议看本篇之前先看[eventlet使用,绿色线程工作原理](http://www.lolizeppelin.com/2017/03/10/python-eventlet/)

1. launcher  
> openstack里的launcher有两种(在oslo_service.service中),
> ServiceLauncher和ProcessLauncher,一般用法为

```python
def launch(conf, service, workers=1):
    if workers is None or workers == 1:
        launcher = ServiceLauncher(conf)
        launcher.launch_service(service)
    else:
        launcher = ProcessLauncher(conf)
        launcher.launch_service(service, workers=workers)
    return launcher
```

由于各个组件是不同人写的,写法有点不同,上面写法是neutorn里的比较标准

nova里的为了单例绕了一下就难看一点

```python
# 用这个来做单例
_launcher = None
# 外部代码里调用的是serve而不是launch比较难看,neutron里的比较好看
def serve(server, workers=None):
    global _launcher
    if _launcher:
        raise RuntimeError(_('serve() can only be called once'))
    # 就是前面的launch函数
    _launcher = service.launch(CONF, server, workers=workers)

def wait():
    _launcher.wait()
```


一般都使用多进程的ProcessLauncher,我们来看下核心代码

```python

class ServiceWrapper(object):
    def __init__(self, service, workers):
        # service对象
        self.service = service
        # 总共可fork的进程数量
        self.workers = workers
        # 存放子进程pid
        self.children = set()
        # 存放每次fork出子进程的用时
        # 用于防止fork进程过快
        self.forktimes = []


class ProcessLauncher(object):
    """Launch a service with a given number of workers."""

    def __init__(self, conf, wait_interval=0.01):
        ...
        # 子进程字典,pid为key
        self.children = {}
        # 运行标志,实例化的时候就认为已经启动
        self.running = True
        # 循环间隔
        self.wait_interval = wait_interval
        # 这launcher是子进程里用的,主进程中无用
        self.launcher = None
        # eventlet通信管道
        rfd, self.writepipe = os.pipe()
        self.readpipe = eventlet.greenio.GreenPipe(rfd, 'r')
        # 信号处理
        self.signal_handler = SignalHandler()
        self.handle_signal()

    def launch_service(self, service, workers=1):
        # 这个函数是一个初始化函数只运行一次
        # 这里检查service的类型是不是继承于ServiceBase
        _check_service_base(service)
        # 生成wrap, 看前面ServiceWrapper类,里面有部分属性没有方法
        wrap = ServiceWrapper(service, workers)
        LOG.info(_LI('Starting %d workers'), wrap.workers)
        # 生成工作的worker,也就是这里fork出多个进程
        # 这里可以看出ServiceWrapper里面保存了所有的子进程的pid之类的信息
        while self.running and len(wrap.children) < wrap.workers:
            self._start_child(wrap)

    def _start_child(self, wrap):
        ...
        # 省略前面防止fork过快的代码
        pid = os.fork()
        if pid == 0:
            # 子进程中给launcher赋值
            self.launcher = self._child_process(wrap.service)
            # 自动重启launcher的循环
            while True:
                # 子进程注册进程接受信号
                self._child_process_handle_signal()
                # 接收到信号返回,未收到信号的时候进程阻塞于此,内有死销魂还
                status, signo = self._child_wait_for_exit_or_signal(
                    self.launcher)
                # 收到正常退出信号,子进程结束
                if not _is_sighup_and_daemon(signo):
                    self.launcher.wait()
                    break
                # 重启sefl.launcher
                self.launcher.restart()
            os._exit(status)
        # 主进程
        LOG.debug('Started child %d', pid)
        wrap.children.add(pid)
        self.children[pid] = wrap
        return pid

    # 子进程才用到下面代码

    def _child_process(self, service):
        # 这里注册一次信号接收函数
        # 后面又注册一次不是很理解
        self._child_process_handle_signal()
        # 后面就是eventlet相关代码
        # 这里初始化一个hub
        eventlet.hubs.use_hub()
        # 子进程关闭写管道
        os.close(self.writepipe)
        # 孵化一个_pipe_watcher函数,用于监控管道
        eventlet.spawn_n(self._pipe_watcher)
        # 运行一个随机函数,具体作用未知
        random.seed()
        # 这里就是调用Launcher,注意不是ProcessLauncher
        launcher = Launcher(self.conf)
        launcher.launch_service(service)
        return launcher


    def _child_wait_for_exit_or_signal(self, launcher):
        status = 0
        signo = 0
        try:
            # 子进程的工作循环在这里
            # 这个launcher不是ProcessLauncher
            # 看下面Launcher的wait部分
            launcher.wait()
        except SignalExit as exc:
            signame = self.signal_handler.signals_to_name[exc.signo]
            LOG.info(_LI('Child caught %s, exiting'), signame)
            status = exc.code
            signo = exc.signo
        except SystemExit as exc:
            status = exc.code
        except BaseException:
            LOG.exception(_LE('Unhandled exception'))
            status = 2
        return status, signo
    # 上面是子进程用到的代码

    # 下面是主进程用到的代码
    def wait(self):
        # 最外层的(也就是systemd指向的脚本里调用到的wait)
        # launch_service后会调用这里
        # 这里是主进程的wait死循环部分,不是self.launcher的wait部分
        systemd.notify_once()   # 通知systemd
        if self.conf.log_options:
            LOG.debug('Full set of CONF:')
            self.conf.log_opt_values(LOG, logging.DEBUG)
        try:
            # 这个循环不是工作循环
            # 这个是用于进程崩溃后自动重启的
            while True:
                # 注册信号
                self.handle_signal()
                # 死循环在此,这个函数用于监控并重启子进程
                self._respawn_children()
                # 走到这里表示主进程收到退出信号
                if not self.sigcaught:
                    return
                # 省略部分是处理崩溃的
                ....
        except eventlet.greenlet.GreenletExit:
            LOG.info(_LI("Wait called after thread killed. Cleaning up."))
        # 后面是退出以及清理部分
        ....

    def _respawn_children(self):
        # 重启子进程的函数
        while self.running:
            # 里面主要是调用os.waitpid获取子进程的返回
            wrap = self._wait_child()
            # 因为子进程没有退出所有正常都是走到这里
            if not wrap:
                # 这个sleep比较特别
                # 这个sleep会切换到其他绿色线程
                # 最少self.wait_interval秒后才会返回当前位置
                # eventlet.greenthread.sleep其实是创建了一个定时器
                # 定时器被调度到的时候才切换回当前位置
                # neutron的的主进程里没有socket相关的调用
                # 也没有在Hub注册定时器定时器
                # 所以没有其他绿色线程可以切换
                # 这里的代码相当于切换到hub中的main loop
                # 到时间又切换回来
                eventlet.greenthread.sleep(self.wait_interval)
                continue
            # 有子进程退出,重新fork出一个worker子进程
            while self.running and len(wrap.children) < wrap.workers:
                self._start_child(wrap)

```

下面我们来看和子进程相关的Launcher和Services, Launcher也就是ProcessLauncher中的self.Launcher

```python
class Launcher(object):

    def __init__(self, conf):
        self.conf = conf
        conf.register_opts(_options.service_opts)
        # 创建一个services类
        # services类用于管理service
        self.services = Services()
        # 后门端口
        self.backdoor_port = (
            eventlet_backdoor.initialize_if_enabled(self.conf))

    def launch_service(self, service):
        # service和主进程里的service是同一个
        # 这里和ProcessLauncher中一样先检查service的类型
        _check_service_base(service)
        service.backdoor_port = self.backdoor_port
        # 添加到services中,这里其实就是绿色线程spawn开始的地方
        # keyston启动的时候两个service,一个public的一个admin的
        # systemd如果启动nova-api就有两个services, os-computer和mete-data
        # systemd也可以分别启动os-computer和metadata
        # 其他一般的services都只有一个service
        self.services.add(service)

    def stop(self):
        self.services.stop()

    def wait(self):
        # 看下下面service的wait
        # 阻塞再次
        self.services.wait()

    def restart(self):
        self.conf.reload_config_files()
        self.services.restart()


class Services(object):
    # service的start函数的绿色线程在Services中
    def __init__(self):
        # 一个存放service的列表
        self.services = []
        # 初始化一个threadgroup.ThreadGroup()
        # 这是一个线程池组
        self.tg = threadgroup.ThreadGroup()
        # event对象
        self.done = event.Event()

    def add(self, service):
        # 当我们调用add的时候
        # 就是调用了ThreadGroup的add_thread方法
        # self.run_service是函数, service, self.done是
        # self.run_service的参数
        self.services.append(service)
        # run_service最终会调用service的start方法
        # service就是外部传入的基于ServiceBase类的外部实例
        # 传入self.done的目的在于
        # 任意一个service都可以给self.done发信息通知真个进程退出
        self.tg.add_thread(self.run_service, service, self.done)

    def stop(self):
        """Wait for graceful shutdown of services and kill the threads."""
        # 收到信号后会调用stop函数
        # python的主线程才能收到信号
        for service in self.services:
            # 这里的调用必须能让的serverc.wait里死循环结束
            service.stop()
        # Each service has performed cleanup, now signal that the run_service
        # wrapper threads can now die:
        # 这里就是通知程序退出的地方
        # 需要调用stop才能让run_service线程结束
        # 也就是说调用stop才能让守护进程退出
        if not self.done.ready():
            self.done.send()

        # reap threads
        # 这里其实相当于重复调用tg.wait()
        # 当这里处理完tg里的线程后
        # wait里的tg.wait()也会退出
        self.tg.stop()

    def wait(self):
        """Wait for services to shut down."""
        # 这个wait先调用外部service的wait
        # service的wait一般是调用所有定时器的wait函数
        # service常用的定时器 1是定期汇报(相当于心跳)  2 是定时任务
        # 如果外部service的wait没有死循环会直接走到下面
        # 这里是为了等待service里的循环正常退出
        for service in self.services:
            service.wait()
        # 这里最终会调用ThreadGroup的wait
        # 看下面ThreadGroup的wait
        # Launcher调用的的wait的阻塞就是这里
        # 当然service的wati里有死循环就阻塞在前面
        self.tg.wait()

    def restart(self):
        """Reset services and start them in new threads."""
        self.stop()
        self.done = event.Event()
        for restart_service in self.services:
            restart_service.reset()
            # 这里最终会spawn
            # 实例的event会作为参数传入
            # 也就是说service.start()完成后
            # 调用event.wait()
            self.tg.add_thread(self.run_service, restart_service, self.done)

    @staticmethod
    def run_service(service, done):
        # 这里调用了service的start方法
        try:
            service.start()
        except Exception:
            LOG.exception(_LE('Error starting thread.'))
            raise SystemExit(1)
        else:
            # run_service就是tg里的绿色线程
            # 具体的阻塞位置就是这里
            done.wait()

# 我们再来看ThreadGroup是什么
class ThreadGroup(object):
    def __init__(self, thread_pool_size=10):
        # eventlet 绿色线程的池在这里初始化
        self.pool = greenpool.GreenPool(thread_pool_size)
        self.threads = []
        self.timers = []

    def add_thread(self, callback, *args, **kwargs):
        # 也就是说前面的add方法最终调用了
        # 绿色线程池的spawn方法
        # 也就是说,这里开始,绿色线程池开始让run_service
        # 也就是service的start开始被调用
        # 返回的gt是一个绿色线程
        gt = self.pool.spawn(callback, *args, **kwargs)
        # 这个Thread类是用于关联绿色线程和线程组的
        # Thread初始化的时候会调用绿色线程的link方法
        # link方法用于绿色线程结束后回调一个exit函数
        th = Thread(gt, self)
        self.threads.append(th)
        return th

    def wait(self):
        # 这里是ThreadGroup自己的定时器
        # 需要使用ThreadGroup的时候自行添加
        # 目前没看到哪里有用
        for x in self.timers:
            try:
                x.wait()
            except eventlet.greenlet.GreenletExit:  # nosec
                # greenlet exited successfully
                pass
            except Exception:
                LOG.exception(_LE('Error waiting on timer.'))
        # wait调用这里
        self._perform_action_on_threads(
            lambda x: x.wait(),
            lambda x: LOG.exception(_LE('Error waiting on thread.')))

    def _perform_action_on_threads(self, action_func, on_error_func):
        current = threading.current_thread()
        # threads中全部是Thread的实例
        for x in self.threads[:]:
            # 表明这个绿色线程是当前线程,跳过
            if x.ident == current.ident:
                continue
            try:
                # 这里就调用了Thread.wait()
                # Thread.wait()
                action_func(x)
            except eventlet.greenlet.GreenletExit:  # nosec
                # 抛出的异常是GreenletExit
                # 说明是绿色线程已经正常结束
                pass
            except Exception:
                # 上面lamdba里可以看出这里是调用LOG做了记录
                on_error_func(x)

# 我们来看Thread是什么
class Thread(object):
    def __init__(self, thread, group):
        self.thread = thread
        # 调用绿色线程的link函数
        # 绿色线程的link函数用于所执行函数退出的时候回调用
        # 相当于绿色线程结束的时候调用
        # _on_thread_done(thread, group, self)
        # _on_thread_done是把当前Thread从ThreadGroup中移除
        self.thread.link(_on_thread_done, group, self)
        # 每个绿色线程的id
        self._ident = id(thread)

    @property
    def ident(self):
        return self._ident

    def stop(self):
        self.thread.kill()

    def wait(self):
        # 这里最终调用到绿色线程的wait函数
        # 这里的绿色线程是封装过的GreenThread
        return self.thread.wait()


def _on_thread_done(_greenthread, group, thread):
    # 参数来源参考GreenThread._resolve_links
    # 第一个参数是调用这个函数的绿色线程
    # 第二个参数是当前ThreadGroup,link的时候传入的参数
    # 第三个参数是当前Thread, 也是link的时候传入的
    # 作用是把当前Thread从ThreadGroup里移除
    group.thread_done(thread)
```

接下来就是复杂的[openstack里eventlet的使用](http://www.lolizeppelin.com/2017/03/13/openstack-eventlet-use/)
