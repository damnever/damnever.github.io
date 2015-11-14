--
layout:      post
title:       关于自动提示和补全
subtitle:    Trie(前缀树)，Redis(基于内存的数据库)
date:        2015-11-14 16:00:11
author:      Damnever
keywords:    autocomplete, trie, redis
description: implement autocomplement with trie and redis, 用 Trie 或 Redis 实现自动提示和补全
categories:
  - 自动提示
tags:
  - Trie
  - Redis

---

蛋疼，就酱（[autocomplete](https://github.com/Damnever/Note/tree/master/autocomplete)）↓:

![](https://raw.githubusercontent.com/Damnever/Note/master/autocomplete/test.gif)

参考↓：

- [Trie（维基百科）](https://zh.wikipedia.org/wiki/Trie)
- [Redis 命令参考](http://redisdoc.com/)
- [Three Autocomplete implementations compared（Java 实现）](http://sujitpal.blogspot.com/2007/02/three-autocomplete-implementations.html)
- [Powerful autocomplete with Redis in under 200 lines of Python](http://charlesleifer.com/blog/powerful-autocomplete-with-redis-in-under-200-lines-of-python/)

***