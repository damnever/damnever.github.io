---
layout:      post
title:       给 Github 的 Markdown 文件生成目录
subtitle:    这是个坑，久久未填 ……
date:        2015-3-31 22:10:31
author:      Damnever
keywords:    Github,Markdown,TOC,Table of content
description: 给 Github 的 Markdown 文件生成目录，TOC for Github's markdown file
categories:
  - Python
tags:
  - Python
  - Markdown

---

以前维护这份[学(dai)习(xue)笔(lie)记(biao)](https://github.com/Damnever/Note/blob/master/Python.md)的都是用手工来生成目录的，说来惭愧……

今天实在受不了了，天气变热，上课时老师让我们用给定的税率计算税钱 ……

往[Python.md](https://github.com/Damnever/Note/blob/master/Python.md)添加东西的时候，发现都接近 2K 行了，`Github`也不支持自动生成目录。到时候如果又添加一个类别，岂不是眼睛都要找瞎，想想还是把这个坑填了吧！

---

首先必须保证你的标题格式是这样的才能使用这个脚本，我想不出还有其它格式的……
{% highlight python %}
<h1 id="id1">H1 title</h1>
一级标题
<h2 id="id2">H2 title</h2>
二级标题
<h1 id="id11">H11 title</h1>
又是一级标题……
{% endhighlight %}
生成的目录格式如下，应该支持大多`Markdown`编辑器：
{% highlight python %}
*   [H1 title](#id1)
    *    [H2 title](#id2)
*   [H11 title](#id11)
{% endhighlight %}

帮助文档如下：
{% highlight python %}
*damnever->>> python toc_gen.py -h
usage: toc_gen.py [-h] [-S src] [-D des]

Generates TOC from markdown text.

optional arguments:
      -h, --help  show this help message and exit
      -S src      A path of source file.
      -D des      A file path to store TOC.
{% endhighlight %}

---
`-S`参数后接源`Markdown`文件路径，`-D`后面接要写入的文件路径，相对或绝对路径都行，不指定目的文件，直接打印在屏幕上。

好吧我又废话了，脚本源文件在这： [toc_gen.py](https://github.com/Damnever/toys/blob/master/toc_gen.py)。

---

主要用到了`HTMLParser`和`argparse`：
{% highlight python linenos=table %}
from __future__ import print_function

import os
import argparse
from HTMLParser import HTMLParser

def get_toc(html):

    toc_list = []

    class MyHTMLParser(HTMLParser):

        _prefix = ''
        _id = ''
        _title = ''

        def handle_starttag(self, tag, attrs):
            if tag[-1].isdigit():
                space = (int(tag[-1]) - 1) * 4
                self._prefix = space * ' ' + '*   '
            attrs = dict(attrs)
            if self._prefix and 'id' in attrs:
                self._id = '(#' + attrs['id'] + ')'

        def handle_data(self, data):
            if self._prefix:
                self._title = '[' + data.strip() + ']'
                toc_list.append(self._prefix + self._title + self._id)
            self._prefix = ''
            self._id = ''
            self._title = ''

    parser = MyHTMLParser()
    parser.feed(html)
    return '\n'.join(toc_list)

def read(fpath):
    with open(fpath, 'r') as f:
        data = f.read()
    return data

def write(fpath, toc):
    with open(fpath, 'w') as f:
        f.write(toc)

def parse_args():
    parser = argparse.ArgumentParser(
            description = "Generates TOC from markdown text.")
    parser.add_argument(
            '-S',
            type = file_check,
            default = None,
            help = "A path of source file.",
            metavar = 'src',
            dest = 'src')
    parser.add_argument(
            '-D',
            type = path_check,
            default = None,
            help = "A file path to store TOC.",
            metavar = 'des',
            dest = 'des')
    args = parser.parse_args()
    return args.src, args.des

def file_check(fpath):
    if os.path.isfile(fpath):
        return fpath
    raise argparse.ArgumentTypeError("Invalid source file path,"
            " {0} doesn't exists.".format(fpath))

def path_check(fpath):
    if fpath is None: return
    path = os.path.dirname(fpath)
    if os.path.exists(path):
        return fpath
    raise argparse.ArgumentTypeError("Invalid destination file path,"
            " {0} doesn't exists.".format(fpath))


def main():
    src, des = parse_args()
    toc = get_toc(read(src))
    if des:
        write(des, toc)
        print("TOC of '{0}' has been written to '{1}'".format(
                    os.path.abspath(src),
                    os.path.abspath(des)))
    else:
        print("TOC for '{0}':\n '{1}'".format(
                    os.path.abspath(src),
                    toc))

if __name__ == '__main__':
    main()
{% endhighlight %}

---
