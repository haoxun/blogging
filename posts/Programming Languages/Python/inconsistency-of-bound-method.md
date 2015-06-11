Title: __get__ 于 py2 与 py3 的不一致问题
Date: 26/4/2015

# 前言

目前本文是一篇关于 inconsistency of `function`'s `__get__` in py2 and py3 的部分问题分析，目标读者的可能要求：

* 熟悉 data structure of Python.
* 熟悉 Python 2/3 差异性。

# 问题分析

以下代码的行为在 py2 与 py3 中的结果是不相同的：

```
from __future__ import print_function

class Test(object):
    pass

def func():
    pass

Test.func = func
print(Test.func)
```

运行结果：

```
$ python descriptor.py
<unbound method Test.func>
$ python3 descriptor.py
<function func at 0x10497fc80>
$ python --version
Python 2.7.9
$ python3 --version
Python 3.4.3
```

为了搞清楚这种 inconsistency 的原因，我做了一些检索工作。

根据 Python 2.7 reference 的 [3.4.2.1.][1] 小节，attribute access ，以 `a.name` 为例，会调用 `type(a).__getattribute__`，其返回值为 `a.name` expression 的值：

> The following methods only apply to new-style classes.
>
> object.\_\_getattribute\_\_(self, name)
>		
> **Called unconditionally to implement attribute accesses for instances of the class.**
> ...

stackoverflow 上的 [answer1][2] 与 [answer2][3] 支撑了这种说法。于此同时，[answer1][2] 中提及了 `__getattribute__` 的默认行为：

> Now, what the default \_\_getattribute\_\_ does is that it checks if the attribute is a so-called descriptor or not, i.e. if it implements a special \_\_get\_\_ method. If it implements that method, then what is returned is the result of calling that \_\_get\_\_ method.

同时， py2 与 py3 的文档对于 attribute lookup 的说明并没有区别：

> The default behaviour for attribute access is to get, set, or delete the attribute from an object’s dictionary. For instance, a.x has a lookup chain starting with a.\_\_dict\_\_['x'], then type(a).\_\_dict\_\_['x'], and continuing through the base classes of type(a) excluding metaclasses.

还有就是，[How-To Guide for Descriptors][4] 中对于 `__getattribute__` 的行为作了明确的描述：

> The details of invocation depend on whether obj is an object or a class. Either way, descriptors only work for new style objects and classes. A class is new style if it is a subclass of object.
>
> For objects, the machinery is in object.\_\_getattribute\_\_ which transforms b.x into **type(b).\_\_dict\_\_['x'].\_\_get\_\_(b, type(b))**. The implementation works through a precedence chain that gives data descriptors priority over instance variables, instance variables priority over non-data descriptors, and assigns lowest priority to \_\_getattr\_\_ if provided. The full C implementation can be found in PyObject_GenericGetAttr() in Objects/object.c.
>
> For classes, the machinery is in type.\_\_getattribute\_\_ which transforms B.x into **B.\_\_dict\_\_['x'].\_\_get\_\_(None, B)**.

所以我觉得 inconsistency 的根源应该是在 `function` 的 `__get__` 上面，以下为测试代码：

```
print(func.__get__(None, Test))
```

运行结果与上例相同：

```
$ python descriptor.py
<unbound method Test.func>
$ python3 descriptor.py
<function func at 0x10497fc80>
$ python --version
Python 2.7.9
$ python3 --version
Python 3.4.3
```

这种不一致的根源在于 Python 3 取消了 unbound method 的概念：

```python
class Example:
    @staticmethod
    def member_a():
        pass

    def member_b():
        pass
```

`Example.member_a` 与 `Example.member_b` 是等价的，未经 `staticmethod` decorate 的 `Example.member_b` 不再是一个 unbound method，而是一个 `function`.

[1]: https://docs.python.org/2/reference/datamodel.html#more-attribute-access-for-new-style-classes
[2]: http://stackoverflow.com/a/8961717/4501774
[3]: http://stackoverflow.com/a/4295757/4501774
[4]: http://users.rcn.com/python/download/Descriptor.htm