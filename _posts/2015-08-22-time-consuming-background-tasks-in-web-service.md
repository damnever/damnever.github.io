---
layout:      post
title:       关于 Web 服务中的耗时后台任务的意淫
subtitle:    捣腾的聊天室里的某个坑是时候填了 -- 土制任务队列
date:        2015-08-22 17:00:11
author:      Damnever
keywords:    Task Queue, 耗时后台任务
description: 关于 Web 服务中的耗时后台任务的构想, time-consuming-background-tasks-in-web-service
categories:
  - Web
tags:
  - Task Queue

---

在捣腾这个[聊天室](https://github.com/Damnever/Chat-Room)的时候，注册完发送验证邮件、找回密码发送邮件的时候，整个页面都会阻塞的很明显，当时的需求只要跑起来就行了，也没太在意...不难怪我一天到晚屎尿多，就是意淫少了。

---

首先能想到的最暴力最直接的解决方案就是，起一个后台进程（线程），把任务扔到里面去也不管其死活任由其发展，下面实验一下：

```Python
# -*- coding: utf-8 -*-

import time
import multiprocessing

import tornado.web
import tornado.ioloop


class TestHandler(tornado.web.RequestHandler):

    def get(self):
        def _func():
            time.sleep(10)
            print("Hello, world")
        p = multiprocessing.Process(target=_func, args=())
        p.daemon = True
        p.start()
        print("No blocking...")
        self.write("Cool!")


app = tornado.web.Application([
    (r"/", TestHandler),
])


if __name__ == '__main__':
    app.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```

请求这个页面的时候能立即得到响应，这里的后台任务造化还不错也能正常执行：

![http :8888/]({{ site.baseurl }}/img/post/2015-08-22-01.png)
![python server.py]({{ site.baseurl }}/img/post/2015-08-22-02.png)

诶呦，还不错喔~ 页面阻塞的问题如此轻易的就解决了.........

---

脑子里也蹦出过用`RPC`做这种事情的想法，只见过的却没有实操过的我都想操下... 但是一想觉得还是挺麻烦的，首先要在`RPC`服务端把相关任务写好，想要执行的话发个请求过去，还要接受响应这该如何是好？难道服务端立马回复说我收到请求了，再开个进程来做这个事情，这样也可以把不同的事分开来做，甚至搞成分布式的（这东西我不太懂...）？算了，回头看能不能找些东西脑补下...

---

有种叫任务队列的东西就是专门用来解决这种问题的：

> Task queues manage background work that must be executed outside the usual HTTP request-response cycle.

也有很多成熟的解决方案，最强大的应该是[Celery](http://www.celeryproject.org/)，这货比较笨重，因为就如何配置它网上都有专门的文章来介绍...

轻量级的像[rq](http://python-rq.org/)，看了文档，大概原理应该就是将需要执行的`Python`代码[pickle](https://docs.python.org/2/library/pickle.html)之后等储存在`Redis`里，另一边通过`fork`出来的进程执行任务，使用也比较简单。

[rq](http://python-rq.org/)是个不错的选择，但是还不够轻量...杀鸡焉用牛刀，自己造轮子...

源代码及其实现原理见[taskq](https://github.com/Damnever/Chat-Room/tree/master/taskq)。

下面是一个例子([example.py](https://github.com/Damnever/Chat-Room/blob/master/taskq/example.py))：

```Python
# -*- coding: utf-8 -*-

from __future__ import print_function

import time

import tqueue
import connection


def exc(*args, **kwargs):
    raise TypeError('{0}, {1}'.format(args, kwargs))

def foobar(*args, **kwargs):
    return args, kwargs

class A(object):
    def foobar(self, *args, **kwargs):
        return args, kwargs

class B(object):
    def __call__(self, *args, **kwargs):
        return args, kwargs


if __name__ == '__main__':
    connection.Connection.setup(db=3)

    q = tqueue.Queue()
    id1 = q.enqueue(exc, args=('foo',), kwargs={'e': 'bar'})
    id2 = q.enqueue(foobar, ('foobar',), {'foo': 'bar'})
    id3 = q.enqueue(A().foobar, ('foobar',), {'foo': 'bar'})
    id4 = q.enqueue(B(), ('foobar',), {'foo': 'bar'})
    id5 = q.enqueue(pow, (4, 2),)

    ids = [id1, id2, id3, id4, id5]
    results = dict()

    while True:
        for id_ in ids:
            result = tqueue.get_result_by_id(id_)
            if result is not None:
                results[id_] = result
            else:
                if tqueue.is_waiting(id_):
                    print('#{0} is waiting...'.format(id_))
                elif tqueue.is_failed(id_):
                    print('#{0} need retry...'.format(id_))

        if len(results) == len(ids):
            break
        else:
            time.sleep(0.2)

    print('-'*10)
    for k, v in results.items():
        print('#{0} : {1}'.format(k, v))
```

需要运行`executor.py`来执行任务。

![python executor.py]({{ site.baseurl }}/img/post/2015-08-22-03.png)

还有一些需要完善的地方，来日再战！

***