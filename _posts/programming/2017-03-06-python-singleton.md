---
layout: post
title:  "python 单例模式"
date:   2017-03-06 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


neutron的代码里比较喜欢如下单例模式实现

```python
class Plugin(db.AutoAllocatedTopologyMixin):

    _instance = None

    supported_extension_aliases = ["auto-allocated-topology"]

    @classmethod
    def get_instance(cls):
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance

    def get_plugin_description(self):
        return "Auto Allocated Topology - aka get me a network."

    def get_plugin_type(self):
        return "auto-allocated-topology"

#获取单例
plugin = Plugin.get_instance()
```

nova里的单例launcher写法

```python
def launch(conf, service, workers=1):
    ...
    if workers is None or workers == 1:
        launcher = ServiceLauncher(conf)
        launcher.launch_service(service)
    else:
        launcher = ProcessLauncher(conf)
        launcher.launch_service(service, workers=workers)
    return launcher

_launcher = None

def serve(server, workers=None):
    global _launcher
    if _launcher:
        raise RuntimeError(_('serve() can only be called once'))
    _launcher = service.launch(CONF, server, workers=workers)

```


再来一个new的写法

```python
class Singleton(object):

    # 定义静态变量实例
    __instance = None

    def __init__(self):
        pass

    def __new__(cls, *args, **kwargs):
        if not cls.__instance:
            cls.__instance = super(Singleton, cls).__new__(cls, *args, **kwargs)
        return cls.__instance

instance1 = Singleton()
```

线程安全写法
```python
Lock = threading.Lock()

class Singleton(object):
    __instance = None

    def __init__(self):
        pass

    @classmethod
    def get_instance(cls):
        if cls._instance is None:
            try:
                Lock.acquire()
                # 通过线程锁做双重检查
                if not cls.__instance:
                    cls._instance = cls()
            finally:
                Lock.release()                
        return cls._instance
```

python流行协程,所以现在都不关心线程安全了


其实最简单的单例模式方式

文件singleton.py
```python
class Singleton(object):
    __instance = None
    def __init__(self):
        pass
    @classmethod
    def get_instance(cls):
        if cls._instance is None:
            cls._instance = cls()              
        return cls._instance

singleton = Singleton.get_instance()
```

文件main.py

```python
import singleton
intance = singleton.singleton
```

上述写法就是第一次import就初始化好了实例,不用考虑线程安全

因为python中的module就是最好的单例模式

#### 写在最末尾,了解单例模式可以更好的理解pthon中一切皆对象,module里的class、function其实都是对象,而且都是单例的


最近发现openstack里一个厉害的单例写法,通过元类达成

```python
# 继承type实现单例
class Singleton(type):
    _instances = {}
    _semaphores = lockutils.Semaphores()

    def __call__(cls, *args, **kwargs):
        # lock可以用thread的lock做
        with lockutils.lock('singleton_lock', semaphores=cls._semaphores):
            if cls not in cls._instances:
                cls._instances[cls] = super(Singleton, cls).__call__(
                    *args, **kwargs)
        return cls._instances[cls]

# 添加metaclass的闭包
def add_metaclass(metaclass):
    """Class decorator for creating a class with a metaclass."""
    def wrapper(cls):
        orig_vars = cls.__dict__.copy()
        orig_vars.pop('__dict__', None)
        orig_vars.pop('__weakref__', None)
        for slots_var in orig_vars.get('__slots__', ()):
            orig_vars.pop(slots_var)
        return metaclass(cls.__name__, cls.__bases__, orig_vars)
    return wrapper

@add_metaclass(Singleton)
class YourClass(object):
    """docstring for ."""
    def __init__(self, arg):
        super(, self).__init__()
        self.arg = arg
```
