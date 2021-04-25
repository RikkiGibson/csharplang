# Defaultable value types

Related LDM notes:
- [2020-07-01 Discussion of 'defaultable value types' proposal from sharwell](https://github.com/dotnet/csharplang/blob/3c8559f186d4c5df5d1299b0eaa4e139ae130ab6/meetings/2020/LDM-2020-07-01.md#non-defaultable-struct-types-and-parameterless-struct-constructors)
- [2019-03-04 The 'default' loophole](https://github.com/dotnet/csharplang/blob/3c8559f186d4c5df5d1299b0eaa4e139ae130ab6/meetings/2019/LDM-2019-03-04.md#the-default-loophole)
- [2017-08-09 Struct fields](https://github.com/dotnet/csharplang/blob/3c8559f186d4c5df5d1299b0eaa4e139ae130ab6/meetings/2017/LDM-2017-08-09.md#struct-fields)

## Summary
Structs containing non-nullable reference types are a known "hole" in nullability analysis.

```cs
#nullable enable

var widget = default(Widget);
widget.Prop.ToString(); // NRE, no warning

struct Widget
{
    public string Prop { get; set; }
}
```

It's useful to be able to indicate when a variable of a struct type might contain the default value. The "hero" scenario for this is `ImmutableArray<T>`:

```cs
#nullable enable
using System.Collections.Immutable;

M(default);

void M(ImmutableArray<int> arr)
{
    // produces NRE at runtime, but there's currently no way
    // to convey that the 'default' value is not allowed here.
    _ = arr.Length;
}
```

This proposal describes how handling of structs containing non-nullable reference type fields could be changed to improve safety and allow users to express their intent in more places. Rather than using a special attribute to denote "non-defaultability", this proposal relies primarily on checking the types of fields within struct types to determine the struct's "non-defaultability".

## Struct fields outside of nullable
There is precedent in the language for the fields within a struct type affecting the usage of a struct type.

### Unmanaged
When a struct is used in an unmanaged context, such as when we take a pointer to it, the compiler must check all of the struct's fields at all levels of nesting to determine if the struct contains reference type fields. If it does, we disallow taking pointers to it.

Also, if not all the nested field types are known, such as when a field type is from an assembly that is not referenced, we disallow taking a pointer to it and issue a use-site diagnostic.

```cs
unsafe void M(S1 s1, S2 s2)
{
    var p1 = &s1; // ok
    var p2 = &s2; // error
}

struct S1
{
    private int x;
}

struct S2
{
    private object y;
}
```

### Definite assignment of struct variables
A variable of a struct type is definitely assigned if all the individual fields of the struct are definitely assigned. A few implications of this:
- When some of the fields on a struct are not accessible, it's only possible to definitely assign a struct by assigning the entire thing.
- If a struct adds a new field, consumers that were definitely assigning struct variables by assigning all their fields have to update to assign the new field.

```cs
S s;
s.x = 1;
s.y = new object();
s.ToString(); // ok

struct S
{
    internal int x;
    internal object y;
}
```

## Struct fields in nullable
In C# "next", we introduce the concept of a "non-defaultable" struct type. We introduce a new type annotation `~`, which when applied to a non-defaultable struct type `S`, denotes the "defaultable" struct type `S~`.

Note that previous proposals suggested the annotation `??`, which was found to be unworkable due to the ambiguity of `S ?? s`. The annotation `~` evokes the idea that it's "not quite" an instance of that struct type.

A struct type is considered non-defaultable if it contains any fields which would produce a nullability warning if `default` were assigned to it. For example:
- Non-nullable reference type fields.
- Non-defaultable value type fields.
- `T where T : class?` fields.
- Unconstrained `T` fields.
- Note that this *does not* include fields whose types have "oblivious" nullability.

The expression `default(S)` has type `S~` when `S` is a non-defaultable struct type, and its flow state is maybe-default.  
The initial flow state of all fields on a defaultable struct type is the same as the flow state given by assigning a `default` literal to the field.  
A warning is produced when assigning a value of type `S~` to a variable of type `S` if any of the field states within the value are not compatible with their declared types.

```cs
M1(default); // warning: possible `default` argument.

S~ s = default(S);
s.obj = new object();
M1(s); // ok

M2(default); // ok

void M1(S s)
{
    s.obj.ToString(); // ok
}

void M2(S~ s)
{
    s.obj.ToString(); // warning: possible null reference receiver `s.obj`.
}

struct S
{
    public object obj;
}
```

A warning is produced when accessing a non-field member on a "defaultable" receiver if:
- the member does not have the `[AllowDefault]` attribute, **and**
- any fields within the receiver have flow state incompatible with their declared type. For example, if a non-nullable reference type field has maybe-null flow state.

`var` declarations are implicitly "defaultable" where applicable, for the same reason that `var` declarations of reference types are implicitly nullable.
```cs
var s = default(S); // ok
s.M1(); // ok
s.M2(); // warning: possible default receiver.

struct S
{
    object obj;

    [AllowDefault]
    void M1()
    {
        obj.ToString(); // warning: possible null reference receiver.
    }

    void M2()
    {
        obj.ToString(); // ok
    }
}
```

Tentatively: warnings for assignments to fields are given in the same conditions for a defaultable struct type as in its non-defaultable counterpart.
```cs
var s = default(S);
s.obj = null; // warning: possible null reference assignment.
struct S
{
    public object obj;
}
```
The assumption motivating this behavior is that a "defaultable" value type is used for an optional or incomplete thing. The user's "goal" is assumed to be to either check the value before safely using it, or to initialize its fields and then use it, much like when initialization is performed inside a constructor.

### Properties

Struct properties can vary widely in their behavior when accessed on a `default` receiver.
1. They may return a valid value of the property's type. Here we expect the property to have an `[AllowDefault]` annotation.
2. They may throw an exception when the `get` and/or `set` accessor accessor is called.
3. They may return the `default` value of the property's type.

It feels like the language default should be conservative and assume that any property accessor without `[AllowDefault]` may throw on a struct receiver whose non-nullable fields do not have the required flow state. Hopefully this provides an adequate level of expressiveness without excessively complicating the design.

One potentially "odd" result of this is that if analysis can't see that all the necessary fields are non-null, then accessing a property immediately after setting it can still produce a warning. If it's found to be important to make this work without warnings, it might make sense to introduce an annotation that lets you say "you can access this property only if these fields are non-null", or perhaps "this property can be accessed on a `default` receiver, and its flow state is 'linked' to the flow state of this field".

```cs
var s = default(S);
s.Prop1 = "a"; // ok
s.Prop1.ToString(); // warning: `field2` may not have been initialized

struct S
{
    private string field1;
    public string Prop1
    {
        get => field1;
        [AllowDefault, MemberNotNull(nameof(field1))]
        set { field1 = value; }
    }

    private string field2;
    public string Prop2
    {
        get => field2;
        [AllowDefault, MemberNotNull(nameof(field2))]
        set { field2 = value; }
    }
}
```

An auto property on a struct is equivalent to the following manual property:

```cs
struct S1
{
    private string _prop;
    public string Prop
    {
        get => _prop;
        [AllowDefault, MemberNotNull(nameof(_prop))]
        set { _prop = value; }
    }
}
```

If flow analysis observes that all non-nullable fields are non-null in a defaultable struct variable `S~`, its flow state changes to not-null, and conversion to `S` is permitted without warning.

```cs
var s1 = default(S1);
s1.f1 = "a";
s1.s2.f2 = "b";
s1.ToString(); // ok

struct S1
{
    public string f1;
    public S2 s2;
}

struct S2
{
    public string f2;
}
```

There is a potential overlap with "required properties" here. It's not currently clear what happens when a struct has required properties. For example, is the `default` value allowed to be used, and if so, is it required to assign some set of "required" properties before accessing other members? This proposal is written with the goal to not interfere with such decisions--a struct may require its non-nullable fields to be initialized before certain members are accessed on it, but that is independent of the decision to mark properties "required", for example.

### Postconditions

Struct members can use `[NotDefault]` and `[NotDefaultWhen]` attributes to signal to callers that applicable fields have been initialized.

```cs
void M(MyImmutableArray<T>~ arr)
{
    if (!arr.IsDefault)
    {
        Console.Write(arr.Length); // ok
    }
    Console.Write(arr.Length); // warning
}

struct MyImmutableArray<T>
{
    private T[] _array;

    // `public int Length`, `public T this[int i]`, etc.

    [AllowDefault, NotDefaultWhen(false)]
    public bool IsDefault => _array is null;

    [AllowDefault, NotDefault]
    public void AssertInitialized()
    {
        Debug.Assert(_array is not null);
    }
}
```

This is equivalent to referencing all the applicable fields in the type in a `[MemberNotNull]`, with similar call-site and declaration-site behaviors.

```cs
struct MyImmutableArray<T>
{
    private T[] _array;

    // `public int Length`, `public T this[int i]`, etc.

    [AllowDefault, MemberNotNullWhen(false, nameof(_array))]
    public bool IsDefault => _array is null;

    [AllowDefault, MemberNotNull(nameof(_array))]
    public void AssertInitialized()
    {
        Debug.Assert(_array is not null);
    }
}
```

When `[NotDefault]` or `[NotDefaultWhen]` is applied to the member itself, then the constraint is applied to the `this` parameter, but it can also be applied to parameters.

```cs
public static MyImmutableArrayExtensions
{
    public bool IsDefaultOrEmpty<T>([NotDefaultWhen(false)] this MyImmutableArray<T> arr)
    {
        return arr.IsDefault || arr.Length == 0;
    }
}
```

Open question: Should we introduce `[NotDefault]` and related attributes, would it be reasonable to "repurpose" `[NotNull]`, etc. attributes to mean "not default" when used on structs?

### Constraints

A defaultable struct type `S~` can be used as a type argument for most type parameters without warning.
```cs
T M<T>(T t) where T : struct => t;
```

When a non-defaultable struct type `S` is used as a type argument for an unconstrained type parameter `T`, any usages of `T?` become `S~` after substitution.
```cs
var container = new NullableContainer<ImmutableArray<int>>();
_ = container.Value.Length; // warning: possible default receiver 'container.Value'.

class NullableContainer<T>
{
    public T? Value { get; set; }
}
```

Open question: there is a potential safety hole here in declarations that use type parameters. However, given the *Interfaces* section below, it seems limited to the methods inherited from `System.ValueType`. We could potentially:
- Warn if such overrides do not specify `[AllowDefault]`.
- Implicitly treat overrides of `object.ToString()`/`object.GetHashCode()`/`object.Equals(object)` as `[AllowDefault]`.
- Not warn on declarations, but instead warn when such defaultable types are used as type arguments.

```cs
M(default(S));

void M<T>(T? obj)
{
    if (obj != null)
    {
        obj.ToString(); // boom
    }
}

struct S
{
    object obj;
    public override string? ToString() => obj.ToString();
}
```

### Interfaces

Given a struct type `S` that implements interface `I`, conversion from `S~` to `I` or to a type parameter constrained to `I` is only permitted if all implementation members for interface members accept a `default` receiver.

```cs
M1<S~>(); // ok
M2<S~>(); // warning

void M1<T> where T : IEquatable<T> { }
void M2<T> where T : IDisposable { }

struct S : IEquatable<S~>, IDisposable
{
    public string field;

    [AllowDefault]
    public bool Equals(S~ other)
    {
        return field == other.field;
    }

    public void Dispose() { field.ToString(); }
}
```

### Open questions:

Should it be allowed to put an attribute on a struct type which causes analysis to treat it as "non-defaultable" regardless of the fields within the struct? This would require reassigning the entire struct to make it convertible to a non-defaultable, similar to definitely assigning a struct containing private fields. This leans in the direction of allowing a general "invalid value" pattern, which is itself a bit of a risk, because it grows the scope of the already fairly complex nullable reference types feature.

Should it be permitted to have the type argument to a `System.Nullable` be a defaultable type, e.g. `S~?`? It doesn't seem right to let a nullable value type be defaultable, e.g. `S?~`.
