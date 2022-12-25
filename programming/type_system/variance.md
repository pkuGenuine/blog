# Variance

Variance refers to how subtyping between more complex types relates to subtyping between their components. For example, how should a list of `Cat`s relate to a list of `Animal`s?

If a function accepts a `Animal` as a param, it should be fine we give it a `Cat`. And if it accepts a `List[Animal]` as a param, a `List[Cat]` also seems good.

However, if it accepts a `Callable[[Animal], str]`, `Callable[[Cat], str]` is not a safe choice. A function that can convert `Cat` to `str` is probably not able convert `Animal` to `str`. On the contrary, if it accepts a `Callable[[Cat], str]`, a `Callable[[Animal], str]` is a reasonable candidate.

We say the type constructor `List` is covariant, because if `A` ≤ `B` (`A` can be safely used as `B`, `A` is a subtype of `B`), then `List[A]` ≤ `List[B]`.

We say the type constructor `Callable[[T], str]` is contravariant because it reversed the subtyping relation of the simple types.

A type constructor can also be bivariant (both covariant and contravariant apply, i.e. if `A` ≤ `B`, then `I<A>` ≡ `I<B>`), or invariant (not covariant, contravariant nor bivariant). 

A programming language designer will consider variance when devising typing rules for language features such as arrays, inheritance, and generic datatypes.

By making type constructors covariant or contravariant instead of invariant, more programs will be accepted as well-typed. 

On the other hand, programmers often find contravariance unintuitive, and accurately tracking variance to avoid runtime type errors can lead to complex typing rules.
