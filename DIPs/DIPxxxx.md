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
* `opCmp` reveals an outdated design and implementation. Its presence was historically required by built-in associative arrays, which used binary trees needing ordering. The current implementation of associative arrays uses hashtables that lift the requirement. In addition, not all objects can be meaningfully ordered, so the best approach is to make comparison opt-in. Ordering comparisons in other class-based languages are done by means of interfaces, e.g. [`Comparable<T>` (Java)](https://docs.oracle.com/javase/7/docs/api/java/lang/Comparable.html) or [`IComparable<T>` (C#)](https://msdn.microsoft.com/en-us/library/4d7sx9hd.aspx).
* The static method `factory` is a global dependency sink because it allows creating an instance of any class in the application from a string containing its name. Currently there is no way for a class to opt out. This feature creates [code bloat](https://forum.dlang.org/post/mr6bl7$26f5$1@digitalmars.com) in the generated executable. At best, class factory registration should be opt-in because only a small number of classes (none in the standard library) require the feature.

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

TODO: more to come

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
