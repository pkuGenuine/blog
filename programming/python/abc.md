# ABC

Module `abc` provides the infrastructure for defining abstract base classes (ABCs) in Python.

A class that has a metaclass derived from `ABCMeta` cannot be instantiated unless all of its abstract methods and properties are overridden.

~~~python
from abc import ABC, ABCMeta, abstractmethod

class MyABC(metaclass=ABCMeta): # or MyABC(ABC)
    @abstractmethod
    def abs_func():
        ...
~~~

~~~
>>> MyABC()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: Can't instantiate abstract class MyABC with abstract method abs_func
~~~

ABC 为 API 的编写提供了规范。如果 `isinstance(x, MyABC)`，那么 `x` 一定实现了 `abs_func`。通常情况下，函数的 signature 也是一致的。据此可以在不知道 `x` 具体类型的情况下对 `x` 的方法进行 reasonable 的调用。

ABC 的出现可以认为是对 duck typing 的一种优化，其本质还是 duck typing 的思想：如果一个类实现了所有规定的方法，那它就是想要的类。


## Duck Typing
"If it walks like a duck and it quacks like a duck, then it must be a duck". 

Duck-typing programming style avoids tests using `type()` or `isinstance()`. Instead, it typically employs `hasattr()` tests or EAFP programming (The presence of many try and except statements.).

Duck-typing 显然 work，但 `isinstance` 比 `hasattr` 要直观一些。并且借助 type hints，静态分析就可以得到传入的对象是否满足要求，甚至可以不写 `isinstance`。

## Deeper Rationale

Reference this answer: [Why use ABC](https://stackoverflow.com/a/19328146/17498112).

Abstract base classes' real power lies in the way they allow you to customise the behaviour of isinstance and issubclass. 

Python's source code is exemplary. [Here](https://hg.python.org/cpython/file/8e0dc4d3b49c/Lib/_collections_abc.py#l143) is how collections.Container is defined in the standard library (at time of writing):

~~~python
class Container(metaclass=ABCMeta):
    __slots__ = ()

    @abstractmethod
    def __contains__(self, x):
        return False

    @classmethod
    def __subclasshook__(cls, C):
        if cls is Container:
            if any("__contains__" in B.__dict__ for B in C.__mro__):
                return True
        return NotImplemented
~~~

This definition of `__subclasshook__` says that any class with a `__contains__` attribute is considered to be a subclass of Container, even if it doesn't subclass it directly:

~~~python
class ContainAllTheThings(object):
    def __contains__(self, item):
        return True
~~~

~~~
>>> issubclass(ContainAllTheThings, collections.Container)
True
>>> isinstance(ContainAllTheThings(), collections.Container)
True
~~~

In other words, if you implement the right interface, you're a subclass! ABCs provide a formal way to define interfaces in Python, while staying true to the spirit of duck-typing. Besides, this works in a way that honours the [Open-Closed Principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle).

### Addendum
There is a pitfall in overriding the behaviour of `isinstance` and `issubclass`.

~~~python
class MyABC(metaclass=abc.ABCMeta):
    def not_abstract_method(self):
        pass
    @classmethod
    def __subclasshook__(cls, C):
        return True

class C(object):
    pass

# typical client code
c = C()
if isinstance(c, MyABC):  # will be true
    c.not_abstract_method()  # raises AttributeError
~~~

Avoid defining ABCs with both a `__subclasshook__` and non-abstract methods. Moreover, you should make your definition of `__subclasshook__` consistent with the set of abstract methods your ABC defines.