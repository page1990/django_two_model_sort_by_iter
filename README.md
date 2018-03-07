django 使用生成器给不同的model组合排序
=======================

遇到的问题描述
-----------------------

在django中，有两个model，他们都有一个create_time的字段，用来表示创建时间。
然而，在前端页面展示的时候，需要把这两个表联合起来一起展示，并且需要根据他们的创建时间，从最近的一直到最开始的时间来展示。

这个查询还会使用到分页展示，比如每次展示10条数据，用来减少数据库的查询压力。

两个model的表结构
-----------------

表结构(model)如下定义:
这里为了简化，两个表的字段除了**field_c** 和**field_s**，其他都是相同的
```
class ClientHotUpdate(models.Model):
    create_time = models.DateTimeField()
    project = models.ForeignKey(GameProject)
    title = models.CharField(max_length=100, unique=True, help_text='标题')
    field_c = models.CharField(max_length=100)

class ServerHotUpdate(models.Model):
    create_time = models.DateTimeField()
    project = models.ForeignKey(GameProject)
    title = models.CharField(max_length=100, unique=True, help_text='标题')
    field_s = models.CharField(max_length=100)
```

想要得到的排序结果类似于如下的输出:
分别是 **类 + 时间 + 标题**
```
<class 'myworkflows.models.ServerHotUpdate'> 2018-03-07 10:23:35.419845 title_test
<class 'myworkflows.models.ClientHotUpdate'> 2018-03-06 16:46:42.340582 title_test
<class 'myworkflows.models.ClientHotUpdate'> 2018-03-06 15:45:38.933650 xxx
<class 'myworkflows.models.ClientHotUpdate'> 2018-03-05 17:49:22.751559 title_test
<class 'myworkflows.models.ClientHotUpdate'> 2018-03-05 17:15:14.732055 title_test
<class 'myworkflows.models.ServerHotUpdate'> 2018-03-05 16:22:01.060712 title_test
<class 'myworkflows.models.ClientHotUpdate'> 2018-03-05 16:18:42.115524 title_test
<class 'myworkflows.models.ClientHotUpdate'> 2018-03-05 16:18:42.115524 title_test
<class 'myworkflows.models.ServerHotUpdate'> 2018-03-02 22:33:40.125024 
title_test
```

其中，ServerHotUpdate 和 ClientHotUpdate根据他们的创建时间，会**交替输出**


使用django ORM三种不同的查询方式
-----------------

1 第一版查询方法
第一版本的查询方法很原始简单，具体的思路是查找出全部的 ServerHotUpdate 和ClientHotUpdate，然后把他们放在一个list中，然后在list根据create_time的属性来进行排序

```
# 一个用来存放以后两个model所有查询的结果
all_query = []

# 查询ServerHotUpdate的所有结果
hot_server_query=ServerHotUpdate.objects.select_related('project').order_by('-create_time')

# 查询ClientHotUpdate的所有结果
hot_client_query=ClientHotUpdate.objects.select_related('project').order_by('-create_time')

# 将这两个结果都添加到list中
all_query.extend(hot_server_query)
all_query.extend(hot_client_query)

# 最后根据create_time的属性来排序
all_query = sorted(all_query, key=lambda obj: obj.create_time, reverse=True)[0: 10]
```

**缺点分析:** 当使用 `all_query.extend(hot_server_query)`这里的时候，其实已经将django 的queryset转化成了一个list，这里会触发sql的查询，并且会查询所有的数据，还会生成一个很大的list,然后取这个list的前10条数据

但是其实有时候只是想获取前10条数据而已，因此后面的数据其实是不需要的，这样子会造成一些不必要的浪费(多了一个大的list和mysql的多余查询)

2 第二版查询方法
第二版的查询方法在语法上比第一版的精简了很多，但是其实本质上没有性能方面的提升

```
client_iter = ClientHotUpdate.objects.select_related('project')

server_iter = ServerHotUpdate.objects.select_related('project')

# 这里使用了itertools的chain合并了两个可迭代的对象
all_data = sorted(
    chain(client_iter, server_iter), key=lambda obj: obj.create_time, reverse=True)
raw_data = all_data[0: 10]
```

**缺点分析:** 相对于第一版，这里并没有使用一个第三方的list来保存ClientHotUpdate 和ServerHotUpdate这两个queryset查询的list结果。但是sorted还是会返回一个list的结果，所以这里迭代的时候，其实是先查询了ClientHotUpdate和ServerHotUpdate的所有结果，然后在排序，并且最后把排序的list来切片。如果两个model的数据量很大，最后的list结果也会非常大，造成内存的消耗

3 第三版的查询方法
前面两种方法的缺点很明显，无论怎么样，最终都会生成一个list，如果两个model的数据量很大，这个list也会很大，必然会造成内存的消耗。

解决这个问题的方式其实很明显，就是使用python的生成器，根据自己的需要，来返回多少数据，这样，对服务器的性能消耗也会小很多

```
from itertools import islice
import heapq

# 创建一个反向排序的迭代器
reversed_hot_server_iter = reversed(ServerHotUpdate.objects.select_related('project'))

reversed_hot_client_iter = reversed(ClientHotUpdate.objects.select_related('project'))

# 使用heapq的merge方法，根据create_time来创建一个迭代器
heapq_merge = heapq.merge(reversed_hot_server_iter, reversed_hot_client_iter, key=lambda obj: obj.create_time, reverse=True)

# 需要注意的是，这里的heapq_merge是一个迭代器
# 如果想要取迭代器的切片，使用islice
raw_data = islice(heapq_merge, 0, 10)

# 到这里raw_data其实也还是一个迭代器，如果需要得到具体的参数
# 把他转化为一个list即可
list(raw_data)
```

**优点分析：** 一直到最后的list(raw_data)，之前的操作其实都是只是创建了迭代器而已，并没有真正的触发sql执行，也没有创建或者返回一个临时的list，同时，由于对迭代器做了切片的操作，最终的list其实只和我们最终要切片的大小有关，节省了内存，减少了一些不必要的操作和查询
