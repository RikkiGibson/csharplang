# Defaultable structs

## Prior art

- [Championed: Non-defaultable value types #146](https://github.com/dotnet/csharplang/issues/146)
- [Should non defaultable value types be per instance rather than per type? #3322](https://github.com/dotnet/csharplang/issues/3322)
- [Non-defaultable struct types and parameterless struct constructors (LDM notes)](https://github.com/dotnet/csharplang/blob/master/meetings/2020/LDM-2020-07-01.md#Non-defaultable-struct-types-and-parameterless-struct-constructors)

## Annotating struct type declarations
In C#, we like explicit requirements around the usage of types and methods. For example, we prefer that a method declare explicitly whether its return value could be null, as opposed to the compiler inserting an annotation implicitly by analyzing the implementation of the method.

Non-defaultable structs intuitively fall into this category: if the default value of the struct isn't valid for performing some number of operations with it, then we want to require special annotations of such struct types when the value could be the default value. Previous proposals have suggested use of an attribute like the following, to opt-in to the increased restrictiveness of the checks:

```cs
var arr = default(ImmutableArray<int>);
_ = arr.Length; // warning: possible default receiver?

[NonDefault] // or a similar attribute or modifier.
public readonly struct ImmutableArray<T>
{
    private T[] _array;
    public int Length => _array.Length;
}
```

However, it isn't clear that this explicit annotation to opt-in to "non-defaultability" is necessary or desirable. Instead, the language could focus on "filling the gaps" of the nullable reference types in combination with structs, by introducing a new set of annotations and analysis specifically around structs that *contain* non-nullable references. .NET library developers understand (or are bound to find out) that all fields on a struct, no matter their accessibility, are still effectively part of the public contract of a struct, and that changing them can result in binary compatibility breaks, for example.

This is not unlike the way the language decides whether structs meet the `unmanaged` constraint or whether pointers may be taken to them: do they contain reference types in any of their fields? Now, let's explore the implications of a language feature where behavior differs based on whether structs contain *non-nullable reference types*.

## Annotating type usages instead of type declarations
We define a new type annotation, `~`, which can be used as a postfix on a struct type. For example: `ImmutableArray<int>~`. Such a struct type is called a "defaultable struct type".
- A value of such a type will have the initial flow state of its fields assumed to be the result of assigning the `default` literal to the field: maybe-null for reference types, and maybe-default for unconstrained generic types.
- We define an implicit conversion from `T` to `T~` that always succeeds, much like `T` to `T?` with nullable reference types.
- We define an implicit conversion from `T~` to `T` which succeeds only if:
    - the concrete type of the `T` is known (i.e. it must not be a type parameter)
    - all non-nullable reference type fields contained at any level of nesting have a non-null state
    - all the unconstrained or `class?`-constrained generic type fields contained at any level of nesting do not have a maybe-default state
    - Otherwise, a warning is produced.
- `default(T)` and `new T()` on structs are defined to return the type `T~`.
- `T~` is permitted as a type argument for unconstrained `T`.

- Types may contain members which accept a `this~` receiver by putting the `default` modifier on the member, similar to `readonly` members:
```cs
public struct ImmutableArray<T>
{
    public readonly void M0() { } // Type of `this` parameter is `in ImmutableArray<T>`
    public default void M1() { } // Type of `this` parameter is `ref ImmutableArray<T>~`
    public readonly default void M2() { } // Type of `this` parameter is `in ImmutableArray<T>~`

    // Usage likely looks like:
    [NotDefaultWhen(false)]
    public readonly default bool IsDefault => _array is null;

    public readonly int Length => _array.Length; // Type of `this` parameter is `in ImmutableArray<T>`
}
```
- Potentially, structs which contain non-nullable reference types, but whose members all "function" in terms of meeting their non-nullability constraints and not throwing NRE due to the state of fields, could be decorated with the `default` modifier to make all their members defaultable.
- The `default` modifier and `~` annotations are only permitted in a `#nullable enable` context.
- Auto-implemented properties are implicitly defaultable--accessing such properties will not produce a nullability warning, but the initial state of the property will be maybe-null or maybe-default if the receiver type is defaultable.
- Members can add `[NotDefaultThis]` to indicate that the method will throw when the receiver is default, and that we shouldn't warn on the call.
- Members can add `[NotDefaultThisWhen(bool)` attributes to indicate that a particular return value means the receiver has a particular defaultability.
```cs
public struct ValueHolder
{
    public string Prop1 { get; } // type of `this` parameter is implicitly `in ValueHolder~`
    public string? Prop2 { get; } // type of `this` parameter is implicitly `in ValueHolder~`
}
```

## Open questions:
1. Is a defaultable value type an acceptable argument for a type parameter constrained to `new()`?
   - Because `T : struct` implicitly meets `T : new()` I think this must be permitted. **Need to think through any pain this could cause.**
2. Is a defaultable value type an acceptable argument for a type parameter constrained to interface, where `T` implements the interface?
   - No. This design assumes that a value type whose non-nullable reference type fields may contain null is in an "unready" state, and needs to be checked or initialized before use.
3. Consider the type `T~` where `T` is a type parameter constrained to `struct`. 
4. How can we determine an initial flow state for properties of defaultable value types? Aren't they supposed to be opaque? Will there be some way to distinguish auto-implemented properties from manual properties?
  - This is easy to do in source but it's not doable by design for metadata properties. Therefore it's not clear how we can come up with an initial state here.
  - Potentially, we could assume that the initial state of `default`able properties is the state resulting from assigning `default`, requiring either writing to the property, testing its value, etc.

## Composition

Let's explore how it will work to analyze structs which contain non-nullable reference types at various levels of nesting.
```cs
public struct Inner
{
    public string Prop { get; }
}

public struct Outer
{
    public Inner Inner { get; }

    public void M()
    {
        Inner.Prop.ToString();
    }

    public default void MDefault()
    {
        Inner.Prop.ToString(); // warning: dereference of a possible null reference.
    }

    public static Outer Create1()
    {
        // warning: Outer~ can't be converted to Outer
        return new Outer();
    }

    public static Outer Create2()
    {
        var outer = new Outer();
        outer.Inner.Prop = "a";
        return outer;
    }

    public static void M1()
    {
        var outer = new Outer();
        outer.M(); // warning: 
    }
}
```