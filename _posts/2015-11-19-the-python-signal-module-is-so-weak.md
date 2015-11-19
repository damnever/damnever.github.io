---
layout:      post
title:       Python 2 对信号的支持简直弱爆了
subtitle:    no zuo no die，还好了解那么一丁点 unix 环境编程...
date:        2015-11-19 15:00:11
author:      Damnever
keywords:    ping, icmp, Python, Python C extension, signal
description: The Python signal module is so weak, Python 对信号的支持简直弱爆了
categories:
  - Signal
tags:
  - ping
  - signal

---

（更：原来 Python 3 已经支持 `sigprocmask/sigpending` 这些了，弃 Python 2 保平安！）

最近在倒腾一个 [ping](https://github.com/Damnever/dping) 程序，实现 ping 就不多说了，主要就是构造和解析 ICMP 报文，网上有很多介绍 ICMP 协议的文章，讲得很清楚了，我就不卖瓜了（自己都不敢买，更别说夸了。。。）。

这个 `ping` 程序（[dping](https://github.com/Damnever/dping)）是 `Ubuntu` 上不带参数的高仿货，就是因为要高仿，所以，遇到了一个问题。

Linux 上的 ping 程序应该是不会终止的，我造的这个如果不给它发送 `SIGINT` 信号（也就是`Ctrl+C`），会发送 255 个报文之后自动停止，然后计算丢失率。问题就出在这里，考虑这样一种情形：当程序发出一个 ICMP 请求回显报文后，将发出的报文计数加一，收到来自目的地址的回显应答报文后，将收到的报文计数加一。如果，如果，如果，程序发出了一个请求回显报文并把发出的报文计数加一，这个时候你按下了 `Ctrl+C` 给程序发送 `SIGINT` 信号，程序就终止了，假设，假设，假设这个回显应答报文是肯定能收到的，但是因为程序已经挂了，收到的报文计数就比理论上少了一，计算丢失率的时候就会比理论值要大些，既然 ping 得通丢几个包也无所谓嘛，但是我不能忍！！！

可能你会问，为毛不把最后一个包给丢弃了？好吧，这就是坑，我说不能丢！！！

好吧！现在的问题是要让程序发收报文的时候不受 `SIGINT` 的影响，这个过程完了之后再响应这个信号。也就是让这个信号先阻塞不能递送，变成当前未决的，要做的动作完成之后再恢复它。好歹我也了解那么一丁点 unix 环境编程，于是我就找寻 Python 对这个的支持，结果大失所望。。。看来是时候摆弄 Python 的 C 扩展了。

C 语言学渣，不喜勿喷！下面是 C 扩展代码（[_sigpending.c](https://github.com/Damnever/dping/blob/master/dping/_sigpending.c)，代码是 Pyhton 2 和 3 兼容的）：

```C
#include <Python.h>
#include <signal.h>

#if PY_MAJOR_VERSION >= 3
#define PyInt_AsLong(x) (PyLong_AsLong((x)))
#endif

static sigset_t newmask, oldmask, pendmask;


static PyObject*
save_mask(PyObject *self, PyObject *arg)
{
    int idx, size;
    PyObject *list, *item;

    if (!PyArg_ParseTuple(arg, "O", &list)) {
        return NULL;
    }

    sigemptyset(&newmask);
    size = PySequence_Length(list);
    for (idx = 0; idx < size; ++idx) {
        item = PySequence_GetItem(list, idx);
        sigaddset(&newmask, PyInt_AsLong(item));
    }

    if (sigprocmask(SIG_BLOCK, &newmask, &oldmask) != 0) {
        Py_Exit(1);
    }

    Py_RETURN_NONE;
}

static PyObject*
pending_and_restore(PyObject *self, PyObject *arg)
{
    if (sigpending(&pendmask) != 0) {
        Py_Exit(1);
    }
    if (sigprocmask(SIG_SETMASK, &oldmask, NULL) != 0) {
        Py_Exit(1);
    }

    Py_RETURN_NONE;
}


static PyMethodDef methods[] = {
    {"save_mask", save_mask, METH_VARARGS, ""},
    {"pending_and_restore", pending_and_restore, METH_VARARGS, ""},
    {NULL, NULL, 0, NULL},
};

#if PY_MAJOR_VERSION >= 3
static struct PyModuleDef sigpendingmodule = {
    PyModuleDef_HEAD_INIT,
    "_sigpending",
    NULL,
    -1,
    methods
};

PyMODINIT_FUNC
PyInit__sigpending(void)
{
    return PyModule_Create(&sigpendingmodule);
}
#else
PyMODINIT_FUNC
init_sigpending()
{
    Py_InitModule("_sigpending", methods);
}
#endif
```

编译成动态链接库：
```
gcc _sigpending.c -fPIC -shared -I /usr/include/python2.7/ -o _sigpending.so
```

还是用 Context Manager 封装一下吧（[sigpending.py](https://github.com/Damnever/dping/blob/master/dping/sigpending.py)）：

```Python
import contextlib

from _sigpending import save_mask, pending_and_restore


@contextlib.contextmanager
def sigpending(*signos):
    save_mask(signos)
    yield
    pending_and_restore()
```

测试一下：

```Python
import time
import signal

from sigpending import sigpending


def main():
    try:
        with sigpending(signal.SIGINT):
            print('Doing sth.')
            for i in range(1, 6):
                time.sleep(1)
                print('{0} sec passed!'.format(i))
            print('Done.')
    except KeyboardInterrupt:
        print('\nInterruped')

if __name__ == '__main__':
    main()
```

输出如下（`^C`即`Ctrl+C`）：

```
Doing sth.
1 sec passed!
^C2 sec passed!
3 sec passed!
4 sec passed!
5 sec passed!
Done.

Interruped
```

好了，自己挖的坑好歹也算是填了。

ping 程序的源码这边走：**[→](https://github.com/Damnever/dping)**

***
