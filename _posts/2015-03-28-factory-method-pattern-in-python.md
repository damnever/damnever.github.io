---
layout:     post
title:      工厂方法模式在 Python 中的应用
subtitle:   坑爹的 Win10 + A卡让我玩不了 LOL，%>_<%，于是在 Ubuntu 下诞生了这个 Blog
date:       2015-3-28 23:30:31
author:     Damnever
categories:
  - 设计模式
  - Python
tags:
  - Python
  - 工厂方法模式
  - Tornado

---

每过一阵子总有那么几天，想要手残一下。这不又格式化了整个硬盘，重装了双系统。结果 Win10+A卡 玩不了撸啊撸了…… 只好老老实实回到 Ubuntu，在网上乱入发现了这张图，给我的逃课行为提供了有力的支撑～

![2015-03-28]({{ site.baseurl }}/img/post/2015-03-28.jpg)

然后，这个基于`git-pages`的 Blog 也随之诞生了。

---
&emsp;&emsp;&emsp;&emsp;扯谈的分割区

---

不知天高地厚的看了一下`Tornado`的`ioloop`模块的源码，结果是毫无头绪，其它的就暂且不说。`tornado.ioloop.IOLoop.instance().start()`是用来启动`Tornado`的，但是在源码里面`IOLoop().start()`压根就没实现，子类`PollIOLoop().start()`倒是实现了。看看`instance()`到底是啥：
{% highlight python linenos %}
>>> import tornado.ioloop
>>> repr(tornado.ioloop.IOLoop.instance())
'<tornado.platform.epoll.EPollIOLoop object at 0x7f4ac177ae50>'
{% endhighlight %}

哪里又冒出了一个`EPollIOLoop`?

{% highlight python %}
继承关系:
                    +-------------------+
                    | util.Configurable |
                    +-------------------+
                            ^
                            |
                    +---------------+
                    | ioloop.IOLoop |
                    +---------------+
                            ^
                            |
                    +-------------------+     +------------------------------+
                    | ioloop.PollIOLoop | <-- | platform.select.SelectIOLoop |
                    +-------------------+     +------------------------------+
                         ^            ^
                         |            |
+----------------------------+    +------------------------------+
| platform.epoll.EpollIOLoop |    | platform.kqueue.KQueueIOLoop |
+----------------------------+    +------------------------------+
{% endhighlight %}

`util.Configurable`的文档是这样描述的：Base class for configurable interfaces.这是个抽象类，它的构造函数(<span id="r1">这里是说`initialize()`，通过`__new__()`来调用[[1]](#1)</span>)给其实现子类之一(应该就是这厮了`ioloop.IOLoop`)扮演工厂函数的角色。还说了其实现子类可以用`configure()`在运行时全局的改变实例，不过这里貌似没用到。通过用`initialize()`作为工厂方法，这个接口就像正常的类一样，`isinstance`之类的方法都可以正常使用。这种模式在这些时候是最有用的：当选择的实现可能是一个全局的决定时（如果`epoll`可用就会代替`select`，在*unix上都可用），或者`previously-monolithic class`(what?)被分为特定的子类时。

看了类文档，眼前一亮，难道这就是我们上节设计模式课所说的工厂方法模式，不是也是了，反正上课也没听太懂……

---

又了上网了解一下工厂方法模式，拙谈这几个类的角色：

> - `util.Configurable`：使用了黑魔法-元类，通过调用工厂方法`initialize()`来创建实例。抽象工厂。
> - `ioloop.IOLoop`：当你调用`tornado.ioloop.IOLoop.instance().start()`时，会根据你所使用的平台通过`configrable_defult()`来返回`SlectIOLoop`/`EpollIOLoop`/`KQueueIOLoop`中最佳的一个类给`util.Configurable`来创建实例。具体工厂。
> - `ioloop.PollIOLoop`：为`IOLoop`来创建接口一致的`select-like`方法。抽象产品。
> - `platform.select.SelectIOLoop/platform.epoll.EpollIOLoop/platform.kqueue.KQueueIOLoop`：这几个只有被选择的份，已经不能再具体了。具体产品。

---

还是来看一个代码片段，看看这个魔法是如何进行的：
{% highlight python linenos %}
from __future__ import print_function

EPOLL, SELECT = 2, 1

class Configurable(object):

    def __new__(cls, **kwargs):
        impl = cls.configurable_default()
        instance = super(Configurable, cls).__new__(impl)
        instance.initialize(**kwargs)
        return instance

    @classmethod
    def configurable_default(cls):
        raise NotImplementedError()

    def initialize(self): # 都可以替换成 __init__，因为__init__ 并不是构造函数
        """ 工厂方法 """


class IOLoop(Configurable):

    @staticmethod
    def instance():
        """ 简单的单例模式 """
        if not hasattr(IOLoop, '_instance'):
            IOLoop._instance = IOLoop()
        print("IOLoop -> instance()")
        return IOLoop._instance

    @classmethod
    def configurable_default(cls):
        print("IOLoop -> configurable_default()")
        if EPOLL: # 假设 EPOLL是最佳方式
            return EPollIOLoop
        return SelectIOLoop

    def initialize(self):
        print("IOLoop -> initialize()")
        pass


class PollIOLoop(IOLoop):

    def initialize(self, impl, **kwargs):
        print("PollIOLoop -> initialize()")
        super(PollIOLoop, self).initialize()


class EPollIOLoop(PollIOLoop):

    def initialize(self, **kwargs):
        print("EPollIOLoop -> initialize()")
        super(EPollIOLoop, self).initialize(impl=EPOLL, **kwargs)


class SelectIOLoop(PollIOLoop):

    def initialize(self, **kwargs):
        print("SelectIOLoop -> initialize()")
        super(SelectIOLoop, self).initialize(impl=SELECT, **kwargs)

if __name__ == '__main__':
    print(repr(IOLoop.instance()))
	print(repr(IOLoop.instance()))
{% endhighlight %}
输出
{% highlight python linenos %}
IOLoop -> configurable_default()
EPollIOLoop -> initialize()
PollIOLoop -> initialize()
IOLoop -> initialize()
IOLoop -> instance()
<__main__.EPollIOLoop object at 0x7f91f7da1590>  # 两个对象id一致
IOLoop -> instance()
<__main__.EPollIOLoop object at 0x7f91f7da1590>
{% endhighlight %}

在这里`Configurable`是一个元类，他被隐式的继承到了子类`IOLoop`，所以在`IOLoop.instance()`里调用`IOLoop()`时，会调用`__new__()`来实例化一个对象，调用`__new__`的时候会调用`IOLoop.configurable_default()`来获取一个最佳的选择（`EPollIOLoop`）,然后实例化这个选择，返回这个实例，所以`IOLoop()`得到了实例是一个`EPollIOLoop`对象。

**可见工厂方法模式的核心就是如何去实例化并得到一个具体的对象，而这个对象又恰恰是我们想要的。`Tornado`这里就用的很好，可以在不同的平台上获得一个一致的接口，而不需要每个平台都去调用一个不同的接口来完成相似的工作。**

---

<span id="1" class="caption text-muted">[[1]](#r1) `__init__()`并不是真正意义上的构造方法，`__init__()`所做的工作是在累的对象创建好之后进行变量的初始化。`__new__()`才会真正的创建实例，是类的构造方法。</span>

***

<span class="fresh">转载请注明出处，原文链接：<a href="{{ site.url }}{{ page.url }}">{{ site.url }}{{ page.url }}</a>。</span>
