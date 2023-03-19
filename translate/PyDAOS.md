# PyDAOS

一个名为 PyDAOS 的 python 模块为 python 用户提供了 DAOS API。它旨在通过本机 python 数据结构公开 DAOS 对象，从而为它们提供 pythonic 接口。本节重点介绍带有自己的容器类型和布局的主要 PyDAOS 界面。它不涵盖可通过 PyDAOS.raw 子模块获得的本机 DAOS API 的 python 绑定。Design

PyDAOS 是一个主要用 C 编写的 python 模块。它将 DAOS 键值存储对象公开为 python 字典。其他数据结构（例如与 numpy 兼容的数组）正在考虑中。 PyDAOS 分配的 Python 对象有：

- 持久化并由字符串名称标识。命名空间由所有对象共享，并由存储名称和对象之间关联的根键值存储实现。
- 在创建时对运行在同一节点或不同节点上的任何进程立即可见。
- 不消耗任何大量内存。对象的内存占用非常低，因为实际内容是远程存储的。这允许操作比节点上可用内存量大得多的巨大数据集。

## Python Container

要在一个pool里创建一个标有tank的python容器:

```
$ daos cont create tank --label neo --type PYTHON
  Container UUID : 3ee904b3-8868-46ed-96c7-ef608093732c
  Container Label: neo
  Container Type : PYTHON

Successfully created container 3ee904b3-8868-46ed-96c7-ef608093732c
```

然后，人们可以通过向DCont构造函数传递池和容器标签来连接到容器。

```
>>> import pydaos
>>> dcont = pydaos.DCont("tank", "neo")
>>> print(dcont)
tank/neo
```

!!!注意PyDAOS有自己的容器布局，因此会拒绝访问非 "PYTHON "类型的容器。

## DAOS Dictionaries

PyDAOS模块导出的第一种数据结构是DAOS字典（DDict），旨在模仿python的dict接口。在设计过程中曾考虑过利用mutablemapping和UserDict，但由于性能原因最终被排除。DDict类建立在DAOS键值存储之上，支持常规python dictionary类的所有方法。一个限制是，只能存储字符串和字节。

一个新的DDict对象可以通过调用父python容器上的dict()方法来分配:

```
>>> dd = dcont.dict("stadium", {"Barcelona" : "Camp Nou", "London" : "Wembley"})
>>> print(dd)
stadium
>>> print(len(dd))
2
```

这将创建一个名为 "stadium "的新的持久化对象，并用两个键值对来初始化它。

一旦创建了字典，它就是持久的，不能被重写:

```
>>> dd1 = dcont.dict("stadium")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/jlombard/src/new/daos/install/lib64/python3.6/site-packages/pydaos/pydaos_core.py", line 116, in dict
    raise PyDError("failed to create DAOS dict", ret)
pydaos.PyDError: failed to create DAOS dict: DER_EXIST
```

!!!注意 对于每个方法，如果操作不能完成，就会引发一个 PyDError 异常，并有一个适当的 DAOS 错误代码（字符串格式）。

要检索一个现有的字典，请使用 get() 方法：

```
>>> dd1 = dcont.get("stadium")
```

新记录可以通过put操作一次插入一条。现有的记录可以通过get()操作来获取。与Python字典类似，也支持直接赋值。

```
>>> dd["Milano"] = "San Siro"
>>> dd["Rio"] = "Maracanã"
>>> print(dd["Milano"])
b'San Siro'
>>> print(len(dd))
4
```

键值对也可以通过bput()/bget()方法批量插入/查找，以一个python dict作为输入。批量操作是平行进行的（最多可以有16个操作），以使操作率最大化。

```
>>> dd.bput({"Madrid" : "Santiago-Bernabéu", "Manchester" : "Old Trafford"})
>>> print(len(dd))
6
>>> print(dd.bget({"Madrid" : None, "Manchester" : None}))
{'Madrid': b'Santiago-Bernabéu', 'Manchester': b'Old Trafford'}
```

键值对通过put/bput操作被删除，将值设置为None或空字符串。一旦被删除，键就不会在迭代过程中被报告。它还支持通过del()和pop()方法进行的del操作。

```
>>> del dd["Madrid"]
>>> dd.pop("Rio")
>>> print(len(dd))
4
```

关键空间可以通过python迭代器进行处理。

```
>>> for key in dd: print(key, dd[key])
...
Manchester b'Old Trafford'
Barcelona b'Camp Nou'
Milano b'San Siro'
London b'Wembley'
```

DAOS 字典的内容可以通过 dump() 方法导出为普通的 python 字典。

```
>>> d = dd.dump()
>>> print(d)
{'Manchester': b'Old Trafford', 'Barcelona': b'Camp Nou', 'Milano': b'San Siro', 'London': b'Wembley'}
```

!!!警告 当使用dump()方法处理大型DAOS字典时，需要小心。

产生的 python 字典将被报告为等同于原始 DAOS 字典。

```
>>> d == dd
True
```

并将被报告为不同，因为两个对象都有分歧。

```
>>> dd["Rio"] = "Maracanã"
>>> d == dd
False
```

我们也可以直接测试一个键是否在字典中。

```
>>> "Rio" in dd
True
>>> "Paris" in dd
False
```

## Arrays

代表DAOS阵列的类，利用numpy的调度机制。更多信息见https://numpy.org/doc/stable/user/basics.dispatch.html。正在进行的工作