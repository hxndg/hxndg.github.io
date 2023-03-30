---
layout:     post   				    # 使用的布局（不需要改）
title:      2023-03-28-devsecops实践
subtitle:   I'm programmer #副标题
date:       2023-03-28 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 学习
---

# devsecops实践



## 1 SAST实践





- 













#### 1.1.1 CodeChecker调研

CodeChecker本身是可以做多种审计的，针对C/C++而言，可以拆分为四步

+ 生成compile_commands.json
+ 根据compile_commands.json，调用analysis流程
+ 调用parse部分分析结果



如何减少误报：

+ 可以使用codechecker的代码内部[in-code-suppression](https://codechecker.readthedocs.io/en/v6.9.0/user_guide/#suppression-code)来标记代码中这部分是误报，





## 2 代码审计

### 2.1 SQL部分

针对SQL的代码审计一般发生在有对数据库提交的部分，这里针对最常用的GORM做审计：GORM对结构体、map结构的value在框架底层都进行了预编译，所以使用此类方式进行CRUD操作时是十分安全的，初次之外存在如下问题

+ GORM对map的value进行了预编译，但却没对key进行预编译，因此如果是key是可以用户指定的，那么就可能存在注入点。我们的代码没有这个问题
+ GORM对表结构的查询使用了预编译，最常见的例子是用？表示数据，比方说`db.Where("name = ?", name)`，但是如果没使用默认的预编译比方说如果是`db.Where("name = '" + name + "'")`这种，那么就会有注入风险。
+ raw sql执行，`db.Raw(sql)`，只要用户能控制sql的内容就存在注入。发现了这个问题
+ exec sql执行，和上面的raw sql执行一样，
+ db.order，采用预编译执行SQL语句传入的参数不能作为SQL语句的一部分，那么OrderBy所代表的的列名、或者是后面跟随的ASC/DESC也无法进行预编译处理。



### 2.2 XSS审计


## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)