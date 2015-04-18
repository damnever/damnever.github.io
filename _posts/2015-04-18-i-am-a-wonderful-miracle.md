---
layout:      post
title:       能像我这样解题，恐怕也算是一朵不可多得的奇葩了
subtitle:    没办法，学的东西不成体系，只能搞些歪门邪道的路子
date:        2015-04-18 19:00:11
author:      Damnever
keywords:    解题，奇葩
description: 一朵不可多得的奇葩，I am a wonderful miracle
categories:
  - Python
tags:
  - 解题

---


最近又被捅了一刀，恩！题目是这样的，没有题目，时间复杂度不可估量。

<span class="caption text-muted">没有详情没有摘要，只有代码，怕被打 ……</span>


*   [奇一](#miracle-one)
*   [葩二](#miracle-two)


---

<h3 id="miracle-one">奇葩解法一</h3>

{% highlight python %}
from functools import partial

def get_groups(matrix, target):
    width = len(matrix[0])
    height = len(matrix)
    overs = []
    count = 0
    get_neighbor = partial(find_neighbor, width=width, height=height, matrix=matrix)
    for j in range(height):
        for i in range(width):
            if matrix[j][i] != target or (j, i) in overs:
                continue
            tree = Tree(Coord(i, j))
            get_neighbor(tree)
            seq = []
            tree2list(tree, seq)
            overs.extend(seq)
            count += 1
    return count

def find_neighbor(tree, width, height, matrix):
    y, x = tree.root()
    current = matrix[y][x]
    if x+1 < width and matrix[y][x+1] == current:
        tree.right = Tree(Coord(x+1, y))
        find_neighbor(tree.right, width, height, matrix)
    if y+1 < height and matrix[y+1][x] == current:
        tree.down = Tree(Coord(x, y+1))
        find_neighbor(tree.down, width, height, matrix)


def tree2list(tree, seq):
    if tree and tree.root:
        tree2list(tree.down, seq)
        seq.append(tree.root())
        tree2list(tree.right, seq)


class Tree(object):

    __slots__ = ['root', 'down', 'right']

    def __init__(self, root):
        self.root = root
        self.down = None
        self.right = None


class Coord(object):

    __slots__ = ['x', 'y']

    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __call__(self):
        return self.y, self.x


if __name__ == '__main__':
    matrix = [
            [1, 0, 0, 1, 1],
            [1, 1, 0, 0, 1],
            [0, 1, 1, 1, 0],
            [0, 1, 0, 0, 1],
            [1, 0, 1, 1, 0]
            ]
    print get_groups(matrix, 1)
    print get_groups(matrix, 0)
{% endhighlight %}

---

<h3 id="miracle-two">奇葩解法二</h3>

{% highlight python %}
def groups(matrix, target=1):
    width = len(matrix[0])
    height = len(matrix)
    groups = []
    for j in range(height):
        for i in range(width):
            if matrix[j][i] == target:
                groups.append(Point(j, i))
    overs = MyList()
    for each in groups:
        tmp = reduce(lambda x, y: x.combine(y), groups, each).sets
        if tmp not in overs:
            overs.append(tmp)
    return len(overs)


class Point(object):

    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.count = x + y

    def combine(self, other):
        _sets = []
        for each in self.sets:
            if other in self.sets: continue
            if (each.x == other.x or each.y == other.y) \
                    and abs(each.count - other.count) == 1:
                _sets.append(other)
        self.sets.extend(_sets)
        return self

    @property
    def sets(self):
        if not hasattr(self, '_sets'):
            self._sets = [self]
        return self._sets


class MyList(list):

    def __contains__(self, item):
        for i in self:
            for j in i:
                if j in item:
                    return True
        return False


if __name__ == '__main__':
    matrix = [
            [1, 0, 0, 1, 1],
            [1, 1, 0, 0, 1],
            [0, 1, 1, 1, 0],
            [0, 1, 0, 0, 1],
            [1, 0, 1, 1, 0]
            ]
    print groups(matrix)
    print groups(matrix, 0)
{% endhighlight %}

***