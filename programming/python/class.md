# Class

## Type & Class
In Python 2, classes are in old style. Class and type are not quite the same thing. 

~~~text
>>> class Foo:
...     pass
...
>>> x = Foo()
>>> x.__class__
<class __main__.Foo at 0x000000000535CC48>
>>> type(x)
<type 'instance'>
~~~

In Python 3, all classes are new-style classes. 

~~~
>>> class Foo:
...     pass
... 
>>> x = Foo()
>>> type(x)
<class '__main__.Foo'>
>>> x.__class__
<class '__main__.Foo'>
~~~

Thus, in Python 3 it is reasonable to refer to an object’s type and its class interchangeably.


## Class as an Object

In Python, everything is an object. Classes are objects as well. In general, the type of any new-style class is `type`. The type of the built-in classes you are familiar with is also `type`:

~~~
>>> for t in [int, float, dict, list, tuple]:
...     assert isinstance(type(t), type)
~~~
The type of `type` is also `type`.
~~~
>>> type(type)
<class 'type'>
~~~

Digress: We know that `type` is built-in function in Python. However, it seems that the `type` inside the parentheses is not a function, which makes me confusing. Does name `type` corresponse to a function, or type?

实际上是我自己有点混淆概念了，没搞清楚 Python 和 CPython 的界限。简单来说，`type` 是 Python 的 global name，指向了一个 CPython 的 object。后续我们就用 `ty_obj` 来指代这个 object。`ty_obj` 既可以被当成 callable 来调用，又是 class hierarcky 的根。

`type(type)` 就是将 `ty_obj` 作为 callable 来调用，参数也是 `ty_obj`。在 Cpython 发现这个 callable 是 `ty_obj` 之后，于是便用一个特殊的 C 函数来处理了。从这个角度来说，`type` 确实是个 built-in function。更好一点但很啰嗦的说法可能是：

`type` 指向一个 `obj_ty`，`obj_ty` 是一个 class 或 type，是所有 class 包括其自身的 class/type。将 `obj_ty` 作为 callable 进行调用的时候，会 hook 到一个 built-in 函数。

此外注意，这里提到的 class hierarcky 并不是继承的 hierarcky。而是 class/type & instance 的 hierarcky。

~~~
>>> Foo = type('Foo', (), {})
>>> type(Foo)
<class 'type'>
>>> Foo.__bases__
(<class 'object'>,)
>>> type.__bases__
(<class 'object'>,)
~~~

默认情况下继承 hierarcky 的根是 object。即所有的 type/class 也都是 object。


### Class Construction
We can also call type() with three arguments, in form `type(<name>, <bases>, <dct>)`, which will dynamically creates a new class.

- `<name>` specifies the class name. This becomes the `__name__` attribute of the class.
- `<bases>` specifies a tuple of the base classes from which the class inherits. This becomes the `__bases__` attribute of the class.
- `<dct>` specifies a namespace dictionary containing definitions for the class body. This becomes the `__dict__` attribute of the class.

~~~
>>> Foo = type('Foo', (), {'attr_val': lambda x : 100})
>>> x = Foo()
>>> x.attr_val()
100
>>> type(x.attr_val)
<class 'method'>
~~~

实际上，`class xxx:` 的写法，也是通过执行构造出了 class 对象。



## Metaclass
Any class in Python 3, is an instance of the `type`.

Consider again this well-worn example:

~~~
>>> class Foo:
...     pass
...
>>> f = Foo()
~~~

The expression `Foo()` creates a new instance of class Foo. When the interpreter encounters `Foo()`, the following occurs:

The `__call__()` method of Foo’s parent class is called. Since Foo is a standard new-style class, its parent class is the type metaclass, so type’s `__call__()` method is invoked.

That `__call__()` method in turn invokes `Foo`'s `__new__()` and
`__init__()` method.

For example:
...

So, can we custom the construction of a class? Yes, if we inherit from `type`.

~~~
>>> class Meta(type):
...     def __new__(cls, name, bases, dct):
...         x = super().__new__(cls, name, bases, dct)
...         x.attr = 100
...         return x
...
>>> class Foo(metaclass=Meta):
...     pass
...
>>> Foo.attr
100
~~~

Such class constructor classes are called metaclass.

### Useful in ORM
一般情况下，各种需求都能用继承来解决，很少会出现需要 metaclass 的地方。但是在 ORM 中会比较有用，比如 django。

~~~python
class Field(object):

    def __init__(self, name, column_type):
        self.name = name
        self.column_type = column_type

    def __str__(self):
        return '<%s:%s>' % (self.__class__.__name__, self.name)

class ModelMetaclass(type):

    def __new__(cls, name, bases, attrs):
        if name=='Model':
            return type.__new__(cls, name, bases, attrs)
        mappings = dict()
        for k, v in attrs.items():
            if isinstance(v, Field):
                mappings[k] = v
        for k in mappings.keys():
            attrs.pop(k)
        attrs['__mappings__'] = mappings
        attrs['__table__'] = name
        return type.__new__(cls, name, bases, attrs)

class Model(dict, metaclass=ModelMetaclass):

    def __init__(self, **kw):
        super(Model, self).__init__(**kw)

    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError(r"'Model' object has no attribute '%s'" % key)

    def __setattr__(self, key, value):
        self[key] = value

    def save(self):
        fields = []
        params = []
        args = []
        for k, v in self.__mappings__.items():
            fields.append(v.name)
            params.append('?')
            args.append(getattr(self, k, None))
        sql = 'insert into %s (%s) values (%s)' % (self.__table__, ','.join(fields), ','.join(params))
        print('SQL: %s' % sql)
        print('ARGS: %s' % str(args))
~~~


## Summary

Type 的维度可能多于三个吗？或者说，狭义的 class 的 class 能不是 type 吗？感觉不会有。
然后一个类也不可能同时继承 `type` 和一个普通 class 吧？感觉不能吧。

那么: python 内一切都是 object，而其 type system 有两个维度。
1. 在 type 维度有三个级别，最顶层的 metaclass，中间是狭义的 class，最底层是狭义的 instance。
2. 继承关系的维度不好分级，但在最顶层是 object。在 metaclass 和狭义 class 内部可以继承。


