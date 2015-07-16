---
layout:      post
title:       C & Python
subtitle:    Python调用C动态链接库、使用C编写Python扩展
date:        2015-07-16 11:30:11
author:      Damnever
keywords:    Python, C
description: Python调用C语言动态链接库, 使用C语言编写Python扩展
categories:
  - C&Python
tags:
  - C&Python

---

CONTENT:

* [调用动态链接库里的函数](#call-dll)
* [编写Python的C扩展](#C-extension)

---

<h3 id="call-dll">调用动态链接库里的函数</h3>

[ctypes](https://docs.python.org/2/library/ctypes.html)提供了一些方法使Python与C的数据类型相兼容，并且允许Python调用动态链接库或者共享库中的函数。

下面是一段计数排序的C语言代码([counting_sort.c](https://github.com/Damnever/toys/blob/master/algorithm/counting_sort.c))：

{% highlight C %}
#include <stdio.h>
#include <stdlib.h>

void counting_sort(int *nums, int len)
{
    int i, tmp, max = max_num(nums, len);
    int *cnums = (int *)malloc(max * sizeof(int));
    int *bp = nums;

    for(i=0; i < max; i++){
        cnums[i] = 0;
    }

    while((len--) > 0){
        cnums[*bp - 1] += 1;
        bp++;
    }

    bp = nums;
    for(i=0; i < max; i++){
        tmp = cnums[i];
        while((tmp--) > 0){
            *bp++ = i + 1;
        }
    }

    free(cnums);
    cnums = NULL;
}

int max_num(int *nums, int len)
{
    int max = 0;
    while((len--) > 0){
        if(*nums > max){
            max = *nums;
        }
        nums++;
    }
    return max;
}

void print_arr(int *nums, int len)
{
    while((len--) > 0){
        printf("%d ", *nums++);
    }
    printf("\n");
}
{% endhighlight %}

将它编译成动态链接库：

{% highlight bash %}
gcc counting_sort.c -fPIC -shared -o counting_sort.so
{% endhighlight %}

[ctypes](https://docs.python.org/2/library/ctypes.html)里面详细的说明了如何创建需要的`C`数据类型，在调用函数并传参的时候需要将`Python`数据类型转换成相应的`C`数据类型。

下面是调用`C`函数的`Python`代码：

{% highlight python %}
from __future__ import print_function

from ctypes import *


lib_path = './counting_sort.so'

counting_sort = cdll.LoadLibrary(lib_path)

Integers = c_int * 10
int_arr = Integers(2, 3, 5, 1, 4, 9, 8, 7, 6, 5)

length = c_int(len(int_arr))
counting_sort.print_arr(int_arr, length)
counting_sort.counting_sort(int_arr, length)
counting_sort.print_arr(int_arr, length)

for i in int_arr:
    print(i, end=' ')


##=>
# 2 3 5 1 4 9 8 7 6 5
# 1 2 3 4 5 5 6 7 8 9
# 1 2 3 4 5 5 6 7 8 9
{% endhighlight %}

这种方式用来封装（适配）已存在的动态链接库，并给`Python`调用还是非常好的，一般不会有人通过这个自己编写C来扩展`Python`，如果你非要这样当我没说……

---

<h3 id="C-extension">编写Python的C扩展</h3>

照[猫](http://www.tutorialspoint.com/python/python_further_extensions.htm)画虎，下面是一段计算`Python`列表里面所有数字和的`C`代码：

{% highlight C %}
#include <python2.7/Python.h>

static PyObject *sum_num_list(PyObject *self, PyObject *arg)
{
    int idx, size;
    PyObject *list = NULL, *item = NULL;
    PyObject *sum = PyLong_FromLong(0);

    if(!PyArg_ParseTuple(arg, "O", &list)){
        return NULL;
    }
    if(!PyList_Check(list)){
        Py_RETURN_NONE;
    }
    size = PyList_Size(list);

    for(idx = 0; idx < size; idx++) {
        item = PyList_GetItem(list, idx);
        if(!PyNumber_Check(item)){
            continue;
        }
        sum = PyNumber_Add(sum, item);
    }
    return sum;
}


static PyMethodDef sn_method[] = {
    {"sum_num_list", (PyCFunction)sum_num_list, METH_VARARGS, "Sum a list of numbers."},
    {NULL, NULL, 0, NULL}
};

PyMODINIT_FUNC initsumnum()
{
    Py_InitModule3("sumnum", sn_method, "sumnum is C extension module.");
}
{% endhighlight %}

可以通过标准库自带的`distutils`将`C`扩展编译成动态链接库并打包到相应的`dist-package`目录下：

{% highlight python %}
from distutils.core import setup, Extension

setup(
    name='sumnum',
    author='Damnever',
    version='0.1',
    ext_modules=[Extension('sumnum', ['sumnum.c'])])
{% endhighlight %}

这里直接编译成动态链接库，并在当前目录下启动解释器：

{% highlight python %}
[10:00:58] » gcc sumnum.c -fPIC -shared -o sumnum.so
 /vagrant/Python/Test/C
[10:00:58] » python
Python 2.7.10 (default, Jul 12 2015, 16:54:16)
[GCC 4.8.4] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import sumnum
>>> sumnum.sum_num_list([1, 2, 3])
6L
>>> sumnum.sum_num_list([1, 2, 's', 3])
6L
>>> sumnum.sum_num_list([1, 2, 's', 3.2])
6.2
>>> sumnum.__doc__
'sumnum is C extension module.'
>>> sumnum.sum_num_list.__doc__
'Sum a list of numbers.'
{% endhighlight %}

编写`C`扩展，里面有很多东西需要注意。如引用计数就是很重要的，关系到内存使用和垃圾回收，见[Objects, Types and Reference Counts](https://docs.python.org/2/c-api/intro.html#objects-types-and-reference-counts)。总之，这些东西还有待研究……

具体的接口参考[Python/C API Reference Manual](https://docs.python.org/2/c-api/index.html)

***