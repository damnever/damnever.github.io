---
layout:      post
title:       如何设计一个权限系统
subtitle:    在 stackoverflow 上看到一个答案，觉得不错就搬运过来了
date:        2015-09-19 21:00:11
author:      Damnever
keywords:    Permission Design, Database Design, 权限系统, 数据库设计
description: 权限系统数据库设计, How to design a permission system.
categories:
  - Database
tags:
  - Perssion System

---

如果让我来设计的话，建一张表放用户和相关的权限字段，有就为`true`，没有就`false`，或者每种权限建一张表。。。看这几个句点就知道太戳了，查询和扩展等都比较麻烦。

---

还是看看别人怎么玩的吧：[best practice for designing user roles and permission system](http://stackoverflow.com/questions/333620/best-practice-for-designing-user-roles-and-permission-system/25643919#25643919)

以下是糙译:

---

我认为位操作是实现用户权限的最好方式，这里我展示下我们如何用`MySQL`来实现。

下面是一个示例表和一些示例数据：

**表1**：权限表用来存储权限名称以及相应的位，如1,2,4,8...（2的幂）

```
CREATE TABLE IF NOT EXISTS `permission` (
  `bit` int(11) NOT NULL,
  `name` varchar(50) NOT NULL,
  PRIMARY KEY (`bit`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

往表里插入些示例数据。

```
INSERT INTO `permission` (`bit`, `name`) VALUES
(1, 'User-Add'),
(2, 'User-Edit'),
(4, 'User-Delete'),
(8, 'User-View'),
(16, 'Blog-Add'),
(32, 'Blog-Edit'),
(64, 'Blog-Delete'),
(128, 'Blog-View');
```

**表2**：用户表用来存储用户 ID、用户名、和角色。角色通过计算相关权限的和得到。

例如：
如果用户`Ketan`有`User-Add`(bit=1)和`Blog-Delete`(bit=64)的权限，那么他的角色是`65`(1+64)；
如果用户`Mehata`有`Blog-View`(bit=128)和`User-Delete`(bit=4)的权限，那么他的角色是`132`(128+4)。

```
CREATE TABLE IF NOT EXISTS `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL,
  `role` int(11) NOT NULL,
  `created_date` datetime NOT NULL
  PRIMARY KEY (`id`)
) ENGINE=InnoDB  DEFAULT CHARSET=latin1;
```

示例数据：

```
INSERT INTO `user` (`id`, `name`, `role`, `created_date`)
   VALUES (NULL, 'Ketan', '65', '2013-01-09 00:00:00'),
   (NULL, 'Mehata', '132', '2013-01-09 00:00:00');
```

如果想要在用户登录后加载用户的权限，那么我们能通过下面的方式来查询：

```
SELECT permission.bit,permission.name  
   FROM user LEFT JOIN permission ON user.role & permission.bit
 WHERE user.id = 1
```

这里`user.role` `&` `permission.bit`是位操作，输出的结果是：

```
User-Add - 1
Blog-Delete - 64
```

如果我们想检查某个用户是否有`User-Edit`的权限：

```
SELECT * FROM `user` 
     WHERE role & (select bit from permission where name='User-Edit')
```

结果没有行输出。

你还可以看：[http://goo.gl/ATnj6j](http://sforsuresh.in/implemention-of-user-permission-with-php-mysql-bitwise-operators/)

---

全文完

***