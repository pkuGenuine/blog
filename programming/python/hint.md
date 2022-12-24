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

## Protocols

In[PEP544](https://peps.python.org/pep-0544/), static and runtime semantics of protocol classes was introduced to provide a support for structural subtyping (static duck typing).

### Basic Usage

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
# Because dog implements `walk` and `talk` method.
# And their sigature is consitent with `Animal` protocol. 
make_animal_speak(dog)
~~~

### Difference with ABC
It really similar to ABC, isn't it? But they have subtle differences. Reference this [article](https://jellis18.github.io/post/2022-01-11-abc-vs-protocol/).



