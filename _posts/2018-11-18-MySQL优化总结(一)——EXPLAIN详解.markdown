---
layout: post
title: MySQL优化总结(一)——EXPLAIN详解
date:   2018-11-18 15:54
comments: true
description: 详细介绍EXPALIN语句的十大参数具体含义
tags:
- mysql
---

# 1. MySQL优化概述

&emsp;&emsp; `MySQL`优化是门很深的学问，涉及到的方面非常广，一直没有时间来总结，趁今天有空把掌握的、理解的一部分内容重新梳理一下吧，主要是`EXPALIN`语句和`索引优化`两大部分.

&emsp;&emsp;影响数据库性能主要划分为以下几个方面:
- 服务器硬件: 比如CPU， 内存， 硬盘IO， 网卡等
- 设计架构: 比如主从架构， 表大小及结构， 表引擎选择等
- 参数调整: 比如最大连接数`MAX_CONNECTION`， 表缓存大小`TABLE_CACHE`等
- 慢查询优化(重点): 比如由于SQL语句书写不当导致索引失效， 长时间锁表等

&emsp;&emsp;后端开发通常需要关注的重点是`慢查询优化`，一方面是如何找到哪条语句是慢查询，另一方面则是如何优化慢查询.而其中最重要，对性能影响最大的一点就是: `索引`.

# 2. EXPLAIN
&emsp;&emsp; `EXPLAIN`语句是用于分析当前`SQL`语句的常用工具， 下面逐一详细介绍其各个属性的含义:

## 2.1 EXPLAIN属性
&emsp;&emsp; `EXPLAIN的`属性有以下10个:

### (1) id
&emsp;&emsp;表示`SQL`执行的顺序， 如果是平级查询(没有子查询)， 则`id`一样， 顺序从上往下执行; 如果包含子查询， 则子查询的id会变大， 执行顺序为`id`大的先执行.
>id
{:.filename}
{% highlight text linenos%}
    --id 
        --同级SELECT中多个表的表头id相同，按顺序执行
        --SELECT嵌套子查询，表头id会递增，并且从大到小执行
{% endhighlight %}

### (2) select_type 
&emsp;&emsp;表示`SQL`的操作类型.
>select_type
{:.filename}
{% highlight text linenos%}
    --select_type 
        --SIMPLE   简单SELECT，不含子查询或UNION
        --PRIMARY  包含子查询，最外层的称为PRIMARY
        --SUBQUERY 子查询
        --DERIVED  临时表
        --UNION    若第二个SELECT出现在UNION之后，则第二个称为UNION
        --UNION RESULT  合并UNION结果集
{% endhighlight %}

### (3) table
&emsp;&emsp;表示`SQL`操作的表名.
>table
{:.filename}
{% highlight text linenos%}
    --type 
        --表名 一般直接显示表名， 如果显示derived(n)， 表示id为n的子查询的临时表
{% endhighlight %}

### (4) type
&emsp;&emsp;关键属性， 表示`SQL`使用到的索引类型， 根据性能从高到低如下排列: 
>type
{:.filename}
{% highlight text linenos%}
    --type
        --从快到慢排序：
        --system   系统表，只有一行记录
        --const    常量查询，一般是主键或者唯一键，比如WHERE id=1写死，会成为const查询
        --eq_ref   唯一索引，一般是主键或者唯一键，比如WHERE id=(val)查询，只有一条记录匹配
        --ref      非唯一索引，比如WHERE id=(val)查询，=查询，多条索引匹配
        --fulltext(不常见， 遇到再研究)
        --ref_or_null(不常见， 遇到再研究)
        --index_merge(不常见， 遇到再研究)
        --unique_subquery(不常见， 遇到再研究)
        --index_subquery(不常见， 遇到再研究)
        --range    范围索引，比如WHERE id > (val)查询，而非=
        --index    全索引扫描，比如SELECT id FROM emloyees;
        --ALL      全表扫描，比如SELECT * FROM emloyees;
        --一般来说，优化要达到range，最好是ref
{% endhighlight %}

### (5) possible_keys
&emsp;&emsp;表示`MySQL`推测使用到了哪些索引.
>possible_keys
{:.filename}
{% highlight text linenos%}
    --possible_keys
    --当存在多个索引时，mysql推测使用到哪些索引
{% endhighlight %}
    --rows
        --大概有多少行被查询
    --extra
        --十分重要的额外信息
        --using filesort  (一般会造成排序性能严重下降)无法正常使用索引来排序，而使用mysql自认为合适的方式排序
        --using temporary (更严重)使用了临时表保存中间结果
        --using index     (效率高的体现)使用了索引，没有直接访问数据行，
                        --如果同时出现using where，表示索引被用来执行查找，比如SELECT * FROM employees WHERE id = xxx;
                        --如果没有出现using where，表示索引被直接用来读取数据，而不是用来查找，比如SELECT id FROM employees;
        --using where
        --using join buffer 使用了连接缓存
        --impossible where
        --select tables optimized away
        --distinct

### (6) keys
&emsp;&emsp;表示`SQL`实际用到的索引.
>keys
{:.filename}
{% highlight text linenos%}
    --keys
        --当存在多个索引时，实际使用到的索引
        --如果SELECT a1，a2正好对应索引a1，a2，则会出现覆盖索引；也就是说我想要的数据就在索引上，就不用查数据库了；
{% endhighlight %}

### (7) key_len
&emsp;&emsp;(可能的)索引长度.
>key_len
{:.filename}
{% highlight text linenos%}
    --key_len
        --表示最大可能索引长度，并不是实际使用的长度
        --一般根据该字段判断用到了几个索引
{% endhighlight %}

### (8) ref
&emsp;&emsp;表示在`type`为`ref`或`eq_ref`的情况下， 具体用到了哪些字段
>ref
{:.filename}
{% highlight text linenos%}
    --ref
        --这个只在type为ref或eq_ref的情况下才出现
        --表示这个非唯一索引具体引用了哪些字段
{% endhighlight %}

### (9) rows
&emsp;&emsp;表示有多少行被查询
>rows
{:.filename}
{% highlight text linenos%}
    --rows
{% endhighlight %}

### (10) extra
&emsp;&emsp;十分重要的额外信息
>extra
{:.filename}
{% highlight text linenos%}
    --extra
        --using filesort  (一般会造成排序性能严重下降)无法正常使用索引来排序，而使用mysql自认为合适的方式排序
        --using temporary (更严重)使用了临时表保存中间结果
        --using index     (效率高的体现)使用了索引，没有直接访问数据行，
                        --如果同时出现using where，表示索引被用来执行查找，比如SELECT * FROM employees WHERE id = xxx;
                        --如果没有出现using where，表示索引被直接用来读取数据，而不是用来查找，比如SELECT id FROM employees;
        --using where
        --using join buffer 使用了连接缓存
        --impossible where
        --select tables optimized away
        --distinct
{% endhighlight %}

<hr>

&emsp;&emsp;本节列举了`EXPLAIN`语句的属性含义，下一节将将介绍索引的具体优化以及`EXPLAIN`的具体用法。