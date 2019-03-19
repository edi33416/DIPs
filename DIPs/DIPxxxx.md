# `ProtoObject`

| Field           | Value                                                      |
|-----------------|------------------------------------------------------------|
| DIP:            | TBA                                                        |
| Author:         | Andrei Alexandrescu (andrei@erdani.com)                    |
| Review Count:   | 0 [Most Recent]                                            |
| Implementation: | n/a                                                        |
| Status:         | Draft                                                      |

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
