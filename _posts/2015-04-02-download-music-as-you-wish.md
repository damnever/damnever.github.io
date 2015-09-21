---
layout:      post
title:       再也没有人能阻止我下载音乐了
subtitle:    神器 Live HTTP Headers，一个浏览器插件，VENI，VIDI，VICI！
date:        2015-4-2 11:00:31
author:      Damnever
keywords:    下载音乐,Live HTTP Headers
description: 再也没有人能阻止我下载音乐了,download music as you wish
categories:
  - Python
tags:
  - 工具

---

客官，不好意思，你被忽悠了，`Live HTTP Headers`不是你想象中的那种神器，只是一个辅助开发工具而已。

<span id="1" class="caption text-muted">`Live HTTP Headers`是一款可以帮助用户查看当前使用浏览器打开的所有网页的状态，在浏览器中安装了`Live HTTP Headers`插件以后，使用浏览器打开的每个页面加载的每个资源到会被它监测到。</span>

“这跟下载音乐有半毛钱关系啊？”

---

不急，且看我粗暴分解……

> - **安装`Live HTTP Headers`**。

> - 首先找到一个不提供下载的音乐网站（如 http://www.luoo.net/，在 HTML 源码里也找不到音乐链接的那种），找到一个音乐列表，播放……

> - 然后打开这个插件，如果里面已经有了一系列链接，`clear`掉，选择另一首播放，这时`Live HTTP Headers`里就监测到了播放这首音乐所要加载的资源，示例网站就仅仅加载一条`http://mp3.luoo.net/low/luoo/radio621/08.mp3`，呵呵，太`low`了。So easy！

这个网站简直太没有挑战了。

来试试 xiami.com，按照上述步骤，会发现列表里有很多链接，点击`settings`仅选中`XMLHttpRequests`和`Other`，重复上述步骤，发现加载的链接变少了，扫一遍列表，发现一个以`http://m5.file.xiami.com`开头的链接，点开看看`.mp3`的字样实在太明显了，点击链接复制，干你想干的…… 我用百度网盘测试了一下，下载成功。把链接放在迅雷里面应该也是同样可以的，没这个环境没试过。

---

为了提高生活品质，有必要通过编程简化这些繁琐的步骤。

先来看看虾米的链接`http://m5.file.xiami.com/414/61414/1783878337/1772318892_15162340_l.mp3?auth_key=a237fd069a9132400b8f052c3cdbe4ee-1427932800-0-null`，看起来有点复杂的样子，`auth_key`是必须的，貌似一个`md5`值加些其它的，看下`HTML`源码，我勒个去…… 就发现了`61414`和`1783878337`，简直毫无规律可言，的确很复杂…… 好吧暂时放下，先易后难。

看看落网的每首歌曲的`HTML`源码，精简版如下：
{% highlight html %}
<li class="track-item rounded" id="track12084">
    <a class="trackname btn-play">01. I Get Doe (Feat. The Cataracs)</a>
    <span class="artist btn-play">Glasses Malone</span>
    <a data-sid="12084"></a>
</li>
{% endhighlight %}
还附带歌词的，顺便也分析一下得到`http://www.luoo.net/single/lyric/12084`，最后几个数字是不是有是曾相识的感觉？当然还有封面什么的直接就在`HTML`源码里了。

算了这里不需要歌词了，我只要得到歌名和歌手，然后写入`<name-artist>.mp3`即可完成工作，很简单只要正则匹配一下就行了。

要尽量减少输入工作，看一下原网页链接：`http://www.luoo.net/music/621`和音乐源文件链接：`http://mp3.luoo.net/low/luoo/radio621/01.mp3`，再看一下网页，`621`是期刊序号，`01`是歌曲序号，要输入的也只有这两个。

如下简单的运行脚本即可，在当前文件夹下会得到`I Get Doe (Feat. The Cataracs)-Glasses Malone.mp3`的音乐文件：
{% highlight bash %}
*damnever->>> python luoo_download.py 621 1
{% endhighlight %}

简单的代码仅供参考，不排除网站做出资源调整的可能:
{% highlight python %}
from __future__ import print_function

import re
import os
import sys
import urllib2

class Download(object):

    def __init__(self, url):
        self._url = url

    def __enter__(self):
        try:
            self._f = urllib2.urlopen(self._url)
        except urllib2.URLError as e:
            print("{0}\n {1}".format(self._url, e.reason))
        except urllib2.HTTPError as e:
            print("{0}\n {1}".format(e.code, e.reason))
        else:
            return self._f

    def __exit__(self, exc_type, exc_val, exc_tb):
        if hasattr(self, '_f'):
            self._f.close()


class Luoo(object):

    MUSIC_URL = 'http://mp3.luoo.net/low/luoo/radio{0}/{1:0>2}.mp3'
    TITLE_URL = 'http://www.luoo.net/music/{0}'

    TITLE_RE = r'.+?class="trackname btn-play">{0:0>2}\. (.+?)<.+?class="artist btn-play">(.+?)<'

    def __init__(self, journal, order_num):
        self._music_url = self.get_music_url(journal, order_num)
        self._file = self.get_title(journal, order_num) + '.mp3'

    def write_to(self, path='./'):
        fpath = os.path.join(os.path.abspath(path), self._file)
        with Download(self._music_url) as fi, open(fpath, 'wb') as fo:
            fo.write(fi.read())

    def get_title(self, journal, order_num, baseurl=TITLE_URL):
        with Download(baseurl.format(journal)) as f:
            html = f.read()
            pattern = re.compile(self.TITLE_RE.format(order_num), re.S)
            title = pattern.findall(html)
        return '-'.join(title[0]) if title else 'unknown'

    def get_music_url(self, journal, order_num, baseurl=MUSIC_URL):
        return baseurl.format(journal, order_num)

def main():
    if len(sys.argv) == 3:
        journal, order_num = sys.argv[1:]
        Luoo(journal, order_num).write_to('./')
    else:
        print("The program must take 2 arguments!\n \
                Now just a example for: python luoo_download.py 621 3")
        Luoo(621, 3).write_to('./')

if __name__ == '__main__':
    main()
{% endhighlight %}

---
