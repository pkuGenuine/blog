# Type Hints
With type hints, static checker can help us to discover some bugs without running the code. There are two ways to annotate the param, return value or property: norminal or structrual.

## Nominal & Structual
Nominal: compatibility and equivalence of data types is determined by explicit declarations and/or the name of the types. 

In the following example, `f` expects its first param to be an instance of `Foo`. `Bar` is a subtype of `Foo`, thus an instance of `Bar` satisfy this constrain.

~~~python
class Foo:
    pass

class Bar(Foo):
    pass

def f(a: Foo):
    pass 

# This will make the static checker happy.
f(Bar())
# This won't.
f('deadbeaf')
~~~

Structual: compatibility and equivalence are determined by the type's actual structure or definition. Also called static duck typing.

~~~python
from typing import Iterator

class I:
    def __next__(self) -> str:
        return 'deadbeaf'

def f(a: Iterator[str]):
    next(a)

# This will make the static checker happy.
f(I())
~~~

Though class `I` has no declaration with `Iterator[str]`, it is indeed an iterator of `str`. 

Static duck typing is implemented with `Protocol` class (see [PEP544](https://peps.python.org/pep-0544/) for more details). The `typing` module defines various protocol classes that correspond to common Python protocols, such as `Iterator[T]`. 

## Protocols
To define your own protocol, simply inherit from `Protocol` and write the function signatures. Any class that has consistent function signatures are regarded as your protocol by static checker.


~~~python
#animal.py
from typing import Protocol

class Animal(Protocol):
    def walk(self) -> None:
        ...

    def speak(self) -> None:
        ...

def make_animal_speak(animal: Animal) -> None:
    animal.speak()
~~~
~~~python
# Client code
from animal import make_animal_speak
class Dog:
    def walk(self) -> None:
        print("This is a dog walking")

    def speak(self) -> None:
        print("Woof!")

dog = Dog()
# Static checker will be happy,
# Because `Dog` implements `walk` and `talk` method.
# And their sigature is consistent with `Animal` protocol.
# Thus, dog satisfies the Animal constrain.
make_animal_speak(dog)
~~~


### Difference with ABC
It is similar to ABC, isn't it? Actually `Protocol`'s metaclass is inherited from `ABCMeta` with minor amendment.

With hooks like `__subclass__`, ABCs provide structural behavior at runtime. Protocols support such behavior statically. At runtime, protocols has few differences with ABCs.

Protocols are rarely appear outside the type hints. It is good for defining interfaces. Protocols are not "implemented" but tell downstream code what the structure of the input object is expected to be.

Normal ABCs are usually used to construct a framework and inherited by client classes. They are a good mechanism for code reuse, as ABCs (i.e. parent class) do most of the work and have the children implement the specifics.

## More Issues

### Generics & Type Bound
There is a way to "define generic functions" (in the static checker's perspective) in python.

~~~python
from typing import TypeVar, Protocol


class SupportsLessThan(Protocol):
    def __lt__(self, __other: Any) -> bool:
        ...

S = TypeVar('S', bound=SupportsLessThan)

def my_max(x: S, y: S) -> S:
    if x < y:
        return y
    return x
~~~


### Runtime Protocol
Protocol classes decorated with `runtime_checkable()` act as simple-minded runtime protocols that check only the presence of given attributes, ignoring their type signatures.

~~~
>>> from typing import Iterable
>>> isinstance(['t'], Iterable)
True
>>> isinstance(['t'], Iterable[str])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/local/lib/python3.11/typing.py", line 1288, in __instancecheck__
    return self.__subclasscheck__(type(obj))
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/typing.py", line 1291, in __subclasscheck__
    raise TypeError("Subscripted generics cannot be used with"
TypeError: Subscripted generics cannot be used with class and instance checks
~~~

!Question: 不是很懂泛型和 Runtime 的纠缠关系。直观上来看，不管有没有泛型，`Protocol` 都可以在 runtime 做检查，但检查不管 signature 也不管泛型。即便 runtime checkable 了在静态时也会检查 signature 以及 泛型？