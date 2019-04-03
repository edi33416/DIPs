# `ProtoObject`

| Field           | Value                                                                          |
|-----------------|--------------------------------------------------------------------------------|
| DIP:            | TBA                                                                            |
| Author:         | Andrei Alexandrescu (andrei@erdani.com), Eduard Staniloiu (edi33416@gmail.com) |
| Review Count:   | 0 [Most Recent]                                                                |
| Implementation: | n/a                                                                            |
| Status:         | Draft                                                                          |

## Abstract

The root of all classes, and the default superclass if none is
specified in a class definition, is `Object`. `Object` defines four
methods: `toString`, `toHash`, `opCmp`, and `opEquals`.  Their
signatures predate the introduction of the `@nogc`, `nothrow`, `pure`,
and `@safe` attributes, and also of the `const`, `immutable`, and
`shared` type qualifiers.  As a consequence, these methods make it
difficult to use `Object` with qualifiers or in code with properties
such as `@nogc`, `pure`, or `@safe`. This proposal introduces a new
class `ProtoObject` that is the superclass of `Object` and introduces
no method. Existing code will work unchanged; new code may inherit
`ProtoObject` instead of inheriting (implicitly or explicitly)
`Object`, and benefit of a clean slate with regard to defining and
using proper primitives. Modern approaches for formatting as a string,
hashing, and comparison are introduced for `ProtoObject`.

## Rationale

The current definition of `Object` is:

```D
class Object
{
    private __ImplementationDefined __mutex;
    string toString();
    nothrow @trusted size_t toHash();
    int opCmp(Object o);
    bool opEquals(Object o);
    static Object factory(string classname);
}
```

This definition is missing a number of desirable attributes and qualifiers. It is possible to define an improved class that inherits `Object` and overrides these primitives as follows:

```D
class ImprovedObject
{
const override pure @safe:
    string toString();
    nothrow size_t toHash();
    int opCmp(const Object);
    bool opEquals(const Object);
}
```

`ImprovedObject` may be defined in user code (or even in the core runtime library) and inherited from in all user-defined classes in a project for a better experience. However, `ImprovedObject` still has a number of issues:

* The hidden member `__mutex`, needed for `synchronized` sections of code, is still present whether it is used or not. The standard library uses `synchronized` for 6 class types out of the over 70 classes it introduces. At best, the mutex would be opt-in.
* The `toString` method cannot be implemented meaningfully if `@nogc` is required for it. This is because of its signature—constructing a `string` and returning it will often create garbage by necessity. A better implementation would accept an output range in the form of a `delegate(scope const(char)[])` that accepts, in successive calls, the rendering of the object as a string.
* The `opCmp` and `opEquals` objects need to take `const Object` parameters, not `const ImprovedObject`. This is because overriding with covariant parameters would be unsound and is therefore not allowed. Using the weaker type `const Object` in the signature defers checks to runtime that should be done during compilation.
* Overriding `opEquals` must also require the user to override `toHash` accordingly: two objects that are equal, must have the same hash value.
* `opCmp` reveals an outdated design and implementation. Its presence was historically required by built-in associative arrays, which used binary trees needing ordering. The current implementation of associative arrays uses hashtables that lift the requirement. In addition, not all objects can be meaningfully ordered, so the best approach is to make comparison opt-in. Ordering comparisons in other class-based languages are done by means of interfaces, e.g. [`Comparable<T>` (Java)](https://docs.oracle.com/javase/7/docs/api/java/lang/Comparable.html) or [`IComparable<T>` (C#)](https://msdn.microsoft.com/en-us/library/4d7sx9hd.aspx).
* The static method `factory` is a global dependency sink because it allows creating an instance of any class in the application from a string containing its name. Currently there is no way for a class to opt out. This feature creates [code bloat](https://forum.dlang.org/post/mr6bl7$26f5$1@digitalmars.com) in the generated executable. At best, class factory registration should be opt-in because only a small number of classes (none in the standard library) require the feature.
* The current approach doesn't give the user the chance to opt in or out of certain functions (behaviours). There can be cases where the imposed methods don't make sense for the class type: ex. not all abstractions are of comparable types.
* Because of the hidden `__mutex` member, and the fact that the D programming language supports function attributes, the design of Object is susceptible to the Fragile Base Class [Problem](https://www.javaworld.com/article/2073649/why-extends-is-evil.html): this states that a small and seemingly unrelated change in the Base class can lead to bugs and breakages in the Derived classes.

## Related work in other languages

### Java

Java defines the `java.lang.Object` class which is the superclass of all Java classes. `java.lang.Object` defines the following methods that will be inherited by every Java class:

| Method name                                                 | Description |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| boolean equals(Object o);                                   | Gives generic way to compare objects|
| Class getClass();                                           | The Class class gives us more information about the object|
| int hashCode();                                             | Returns a hash value that is used to search objects in a collection|
| void notify();                                              | Used in synchronizing threads|
| void notifyAll();                                           | Used in synchronizing threads|
| String toString();                                          | Can be used to convert the object to String|
| void wait();                                                | Used in synchronizing threads|
| protected Object clone() throws CloneNotSupportedException; | Return a new object that are exactly the same as the current object|
| protected void finalize() throws Throwable;                 | This method is called just before an object is garbage collected |

Each object also has an `Object monitor` that is used in Java `synchronized` sections. Basically it is a semaphore, indicating if a critical section code is being executed by a thread or not. Before a critical section can be executed, the thread must obtain an Object monitor. Only one thread at a time can own that object's monitor.

We can see that there are quite a few simillarites between D and Java:
* both define `opEquals` and `getHash`
* `getClass` is simillar to `typeid(obj)`
* `clone` gives us access to a `Prototype` like creation pattern; in D we have the `Factory Method`
* both have an `object monitor`

### C#

C# defines the `System.Object` class which is the superclass of all C# classes. `System.Object` defines the following methods that will be inherited by every C# class:

| Method name   | Description |
| ------------  | ----------- |
| GetHashCode() | Retrieve a number unique to that object. |
| GetType()     | Retrieves information about the object like method names, the objects name etc. |
| ToString()    | Convert the object to a textual representation - usually for outputting to the screen or file. |

In C#, the [Monitor](https://docs.microsoft.com/en-us/dotnet/api/system.threading.monitor?view=netframework-4.7.2) object is implemented as a separate class that inherits from `Object` in the `System.Threading` namespace.

C# has a smaller number of *imposed* methods, but they are still imposed, and `toString` will continue to be GC dependent.

### Rust

Rust has a different approach: data aggregates (structs, enums and tuples) are all unrelated types. You can define methods on them, and make the data itself private, all the usual tactics of encapsulation, but there is no subtyping and no inheritance of data.

The relationships between various data types in Rust are established using traits. Before delving into `traits`, let's have a look at how one can define a method in Rust.

Structs in Rust can have methods attached to them. The methods for structs in Rust are surrounded by an impl block, like in the following example:

```
struct Address
{
    street: String,
    city : String
}

impl Address 
{
    fn printInfo(&self)
    {
        println!("{} , {}", self.street, self.city);
    }
}
```

Rust `Traits` are like interfaces in OOP languages, and they must be implemented, using `impl`, for the desired type.
An example is worth a thousand words:
```
trait Quack {
    fn quack(&self);
}

struct Duck ();

impl Quack for Duck {
    fn quack(&self) {
        println!("quack!");
    }
}

struct RandomBird {
    is_a_parrot: bool
}

impl Quack for RandomBird {
    fn quack(&self) {
        if ! self.is_a_parrot {
            println!("quack!");
        } else {
            println!("squawk!");
        }
    }
}

let duck1 = Duck();
let duck2 = RandomBird{is_a_parrot: false};
let parrot = RandomBird{is_a_parrot: true};

let ducks: Vec<&Quack> = vec![&duck1,&duck2,&parrot];

for d in &ducks {
    d.quack();
}
// quack!
// quack!
// squawk!
```
Rust traits allow traditional polymorphic OOP through interface inheritance.

The Rust standard library defines a set of default traits that define the interface for some common operations, such as ordering ([Ord](https://doc.rust-lang.org/std/cmp/trait.Ord.html), PartialOrd), equality ([Eq](https://doc.rust-lang.org/std/cmp/trait.Eq.html), PartialEq), hashing ([Hash](https://doc.rust-lang.org/std/hash/trait.Hash.html)) etc.

The compiler is capable of providing basic implementations for some traits via the `#[derive]` attribute. These traits can still be manually implemented if a more complex behavior is required.

The following is a list of derivable traits:
 * Comparison traits: Eq, PartialEq, Ord, PartialOrd
 * Clone, to create T from &T via a copy.
 * Copy, to give a type 'copy semantics' instead of 'move semantics'
 * Hash, to compute a hash from &T.
 * Default, to create an empty instance of a data type.
 * Debug, to format a value using the {:?} formatter.

**Not all traits can be #[derive] ‘d**, as there isn’t a sensible default implementation for all traits.

An example of a trait that can’t be derived is Display, which handles formatting for end users. You should always consider the appropriate way to display a type to an end user. What parts of the type should an end user be allowed to see? What parts would they find relevant? What format of the data would be most relevant to them? The Rust compiler doesn’t have this insight, so it can’t provide appropriate default behavior for you.

Below is a simple example of using derive
```
// `Centimeters`, a tuple struct that can be compared
#[derive(PartialEq, PartialOrd)]
struct Centimeters(f64);
```

Note that the `derive` strategy requires all fields implement the passed-in trait (ex. Eq), which isn't always desired.

Summary:
 * The role played by class is shared between data and traits
 * Structs and enums are just layouts, although you can define methods and do data hiding 
 * Traits don't have any data, but can be implemented for any type (not just structs)
 * Traits can inherit from other traits
 * Traits can have provided methods, allowing interface code re-use
 * Traits give you both virtual methods (polymorphism) and generic constraints (monomorphism)
 

## `ProtoObject`

We propose a simple, economic, and backward-compatible means for code to avoid `Object`: define simpler types as supertypes of `Object`, as follows:

```D
class ProtoObject
{
    // no monitor, no primitives
}
class ProtoObjectWithMonitor : ProtoObject
{
    private __ImplementationDefined __mutex;
}
class Object : ProtoObjectWithMonitor
{
    ... definition unchanged ...
}
```

This reconfiguration makes `ProtoObject`, not `Object`, the ultimate root of all classes. Classes that don't specify a base still inherit `Object` by default, thus preserving backward compatibility. But new code has the option (and will be well advised) to use `ProtoObject` as an explicit base class, therefore eschewing `Object`'s liabilities.

This proposal is based on the following key insight. Currently, `Object` has two roles: (a) the root of all classes, and (b) the default supertype of class definitions that don't specify one. But there is no requirement that these two roles are fulfilled by the same type. This proposal keeps `Object` the default supertype in class definitions, which preserves existing code behavior. The additional supertypes of `Object` only influence newly-introduced code that are aware of them and use them.

The recommended way of going forward with types that inherit from `ProtoObject` is write and implement interfaces that expose the desired behaviour that the type supports. It can be argued that one is not really interested what type an object is, but rather what actions it can perform: what types can it act like? In this regard, an object of type T can be treated as a Collection, a Range or a Key in map, provided that it implements the right interfaces.

The GoF Design Patterns book talks at length about prefering implementing interfaces to inheriting from concrete classes, and why we should **favor object composition over class inheritance**. Extending from concrete classes is usually seen as a form of code reuse, but this is easly overused by programmers and can lead to the fragile base class problem. The same code reuse can be achieved through composition and delegation schemes: we are already doing this with `structs` and `DbI`.

Let's have a look at an example that inherits `ProtoObject` and implements `opCmp`
```
interface IComparable(T)
{
    int opCmp(T)(T rhs);
}

class Book : ProtoObject, IComparable!Book
{
    string name;
    string author;
    
    @override int opCmp(Book T)(T rhs)
    {
        if (this == rhs) return 0;
        auto r = name < rhs.name;
        if (r) return r;
        return author < rhs.author;
    }
}
```

### Ordering

In order to be able to compare two objects, we define the `Ordered(T)` interface hierarchy.
```
interface Ordered(T)
if (is(T == ProtoObject))
{
    const @nogc nothrow pure @safe scope
    int cmp(scope const Ordered!ProtoObject rhs);
}

interface Ordered(T) : Ordered!ProtoObject
if (!is(T == ProtoObject))
{
    static assert(is(T : ProtoObject));

    const @nogc nothrow pure @safe scope
    int cmp(scope const T rhs);
}
```

At the root of the hierarchy sits `Ordered!ProtoObject`. This is required so we can determine if we can compare two instances of `ProtoObject`. Let's see the example

```
int __cmp(ProtoObject p1, ProtoObject p2)
{
  Ordered!ProtoObject o1 = cast(Ordered!ProtoObject) p1;
  if (o1 is null) return -1; // we can't compare
  return o1.cmp(cast(Ordered!ProtoObject) p2);
}
```
As you can see in the example, if we can't dynamic cast to `Ordered!ProtoObject`, then we can't compare the instances.
Otherwise, we can safely call the `cmp` function and let dynamic dispatch do the rest.

Any other class, `class T`, that desires to be comparable, needs to extend ProtoObject and implement `Ordered!T`; this means that `T` must implement the two overloads of `cmp`: `int cmp(Ordered!ProtoObject);` and `int cmp(T);`.
```
class Widget : ProtoObject, Ordered!Widget
{
  
  const @nogc nothrow pure @safe scope
  int cmp(scope const Ordered!ProtoObject rhs)
  {
    return cmp(cast(Widget) rhs);
  }
  
  const @nogc nothrow pure @safe scope
  int cmp(scope const Widget rhs)
  {
    /* actual cmp impl */
  }
  
  /* impl */
}

```
As you can see, `cmp(Ordered!ProtoObject)` dynamically casts `rhs` to `T` and calls the `cmp(T)` implementation.

`Ordered` implies that implementing types form a total order.
An order is a total order if it is (for all a, b and c):
 * total and antisymmetric: exactly one of a < b, a == b or a > b is true; and
 * transitive, a < b and b < c implies a < c. The same must hold for both == and >.
 
 As you can expect, most of the classes that desire to implement `Ordered` will have to write some boilerplate code.
 Inspired by Rust's `derive`, we implemented `ImplOrdered(T, bool hasCustomCompare = false, M...)` template mixin.
 This provides the basic implementation for `Ordered!T` and more.
 
 At it's most basic form, `mixin`g in `ImplOrdered!T` will go through all the members of the implementing type and compare them with `rhs`
 ```
 @safe unittest
{
    class Widget : ProtoObject, Ordered!Widget
    {
        mixin ImplOrdered!Widget;
        int x;
        int y;

        this(int x, int y)
        {
            this.x = x;
            this.y = y;
        }
    }

    auto w1 = new Widget(10, 20);                                                                                                                                                                                  
    auto w2 = new Widget(10, 21);
    assert(w1.cmp(w2) != 0);
    
    auto w3 = new Widget(10, 20);                                                                                                                                                                                  
    assert(w1.cmp(w3) == 0);
}
 ```
 
 In a more advanced usecase, `ImplOrdered` allows you to specify the only members that you wish to compare.
  ```
 @safe unittest
{
    class Widget : ProtoObject, Ordered!Widget
    {
        mixin ImplOrdered!(Widget, false, "x");
        int x;
        int y;

        this(int x, int y)
        {
            this.x = x;
            this.y = y;
        }
    }

    auto w1 = new Widget(10, 20);                                                                                                                                                                                  
    auto w2 = new Widget(10, 21);
    assert(w1.cmp(w2) == 0);
}
 ```
 
 The final form allows the user to also provide the comparator to use for each specified member
   ```
 @safe unittest
{
    class Widget : ProtoObject, Ordered!Widget
    {
        mixin ImplOrdered!(Widget, true, "x", (int a, int b) => a - b);
        int x;
        int y;

        this(int x, int y)
        {
            this.x = x;
            this.y = y;
        }
    }

    auto w1 = new Widget(10, 20);                                                                                                                                                                                  
    auto w2 = new Widget(10, 21);
    assert(w1.cmp(w2) == 0);
}
 ```
 
Implementations of `Equal` and `Ordered` must agree with each other. That is, `a.cmp(b) == 0` if and only if `a == b`. It's easy to accidentally make them disagree by deriving some of the traits and manually implementing others.


## Implementation Notes

The mechanics for `ProtoObject` already exist in the form of classes with C++ layout:

```D
extern(C++) class ProtoObject
{
}
```

This preexisting facility simplifies implementation effort considerably. Also, the successful coexistence of `extern(C++)` classes with `Object` gives confidence to the approach. The remaining work to do is to confine the mutex to `ProtoObjectWithMonitor`, and to wire together `ProtoObject` and `ProtoObjectWithMonitor` as the supertypes of `Object`.

## Breaking changes / deprecation process

Currently, generic code may use the tests `is(T : Object)` and `is(T == class)` interchangeably, assuming both yield the same result for all unqualified types `T`. This is actually not correct even today because `extern(C++)` classes do not inherit `Object`. With the proposed additions, such an assumption will remain just as incorrect, but possibly the prevalence of mistaken cases will increase.

TODO: more to come

## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

### Reviews
