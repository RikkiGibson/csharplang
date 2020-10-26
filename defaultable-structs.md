# Defaultable types

## Prior art

- [Championed: Non-defaultable value types #146](https://github.com/dotnet/csharplang/issues/146)
- [Should non defaultable value types be per instance rather than per type? #3322](https://github.com/dotnet/csharplang/issues/3322)
- [Non-defaultable struct types and parameterless struct constructors (LDM notes)](https://github.com/dotnet/csharplang/blob/master/meetings/2020/LDM-2020-07-01.md#Non-defaultable-struct-types-and-parameterless-struct-constructors)

## The general problem of initialization

```cs
#nullable enable

// Today, when a type contains non-nullable reference fields, we warn if the constructor doesn't initialize those fields.
public class C1
{
    public string Prop { get; set; } // warning CS8618

    [MemberNotNull(nameof(Prop))]
    public void Init(string prop) => Prop = prop;
}

// The easiest fix (if the field type should not change) is to add a constructor
public class C2
{
    public string Prop { get; set; }

    public C2(string prop) => Prop = prop;
}

// However some people would rather not write/maintain constructors that require maintaining a parameter list and field list in parallel, etc.
var a = new C1 { Prop = "abc" };

var b = new C1();
b.Init();
b.Prop.ToString();

// What can be done for such type declarations that would enable the compiler to understand them?
public partial class C3
{
    public string Prop { get; set; }
}
```

## Disclaimer:
This proposal may prove to be impractical or overly complex. But it feels like there is something here.

Also, the phrase "by default" is confusing to use in this context, so consider saying "unless explicitly specified" or another similar phrase. :)

### Feature goals:
- Provide a desirable set of "out-of-the-box" behaviors which work for analyzing legacy POCOs and use cases where objects mostly only have a 'constructed, but not fully initialized' defaultable phase, and a 'fully initialized' non-defaultable phase.
- The feature is primarily for helping the user to ensure that object fields contain values that are compatible with the field types.
- The feature will have gaps which will not address certain people's use cases. For such use cases, it will be reasonably easy to suppress resulting warnings.

### Feature non-goals:
- Language support for "post-construction" checks after nominal initialization, i.e. "are these fields mutually compatible with each other" checks.
- Language support for "null object" pattern, a la Unity. **However, this may end up falling out of `MyType~` and defaultability postconditions.**
- Hard and fast checks, which for example would give a runtime error if new non-nullable properties are added to a type such that new warnings would be introduced if we recompile.
- Helping users ensure that a nullable reference type property is explicitly specified in an object initializer or in flow analysis, etc. Sorry, if it's nullable, we don't help you (at least not out of the box.)
- Intricate preconditions for method parameters, e.g.: "fields A, B, and nested field C.D" must be initialized before passing the instance to this method".

### Constructors can return non-defaultable or defaultable instances
```cs
#nullable enable
public partial class C3
{
    // This constructor is identical to the implicit constructor that is emitted when no explicit constructors are present.
    // We use the 'default' modifier to indicate that the constructor returns an instance that may not be fully initialized.
    //
    // Note: this is not set in stone. For example, we may require the user to specify a 'default' modifier on the class to make the implicit constructor 'default' in order to avoid changing already shipped field warning behaviors.
    public default C3() { }

    // A constructor which doesn't have the 'default' modifier returns a non-defaultable instance.
    public C3(string prop) => Prop = prop; // ok
    public C3(object obj) { } // warning CS8618: non-nullable property Prop is not initialized.
}
```

### Object creations may create defaultable or non-defaultable instances
```cs
var c3_1 = new C3~(); // ok. creates a defaultable instance without warnings.
var c3_2 = new C3(); // warning: non-nullable property 'Prop' was not initialized.
var c3_3 = new C3() { Prop = "abc" }; // ok.
```

### The `default` literal and `new()` on structs returns a defaultable value
```cs
public struct S1
{
    public string Prop { get; set; }
    public S1(string prop) => Prop = prop;

    public static void UseCtor()
    {
        var s1_1 = new S1(); // The struct default constructor is defined to return `S1~`.
        s1_1.Prop.ToString(); // warning: possible null reference receiver.

        var s1_2 = default; // Default expressions of struct types have a defaultable type.
        s1_2.Prop.ToString(); // warning: possible null reference receiver.

        var s1_3 = new S1("hello"); // Explicit constructors without a 'default' modifier return non-defaultable values.
        s1_3.Prop.ToString(); // ok
    }
}
```

### Unless otherwise specified, instance members are assumed to only accept a non-defaultable 'this'
```cs
#nullable enable
public partial class C3
{
    public void NonDefaultableMethod()
    {
        this.Prop.ToString(); // ok
    }

    public static void UseNonDefaultableMethod()
    {
        new C3().NonDefaultableMethod(); // warning: receiver may not be fully initialized.
    }
}
```

### Members can specify that their parameters or return values are defaultable.
```cs
#nullable enable
public partial class C3
{
    // Use of the 'default' modifier means the 'this' parameter is defaultable.
    public default void DefaultableMethod()
    {
        this.Prop.ToString(); // warning: possible null reference receiver.
    }

    public static void UseDefaultableMethod()
    {
        new C3().DefaultableMethod(); // ok
    }

    // warning: 'default' modifier has no effect, since the method has no 'this' parameter.
    public static default void DefaultableMethod2()
    {
    }

    public static C3~ GetInstance() => new C3(); // Method returns an instance that may not be fully initialized.

    public static C3 UseInstance(C3~ c3)
    {
        if (...) { c3.Prop.ToString(); } // warning: possible null reference receiver.
        return c3; // warning: "return value may not be fully initialized". Consider listing fields that may not be initialized.
    }
}
```
The 'default' modifier on a member has the effect of changing the type of the 'this' parameter to `C3~` (for example). This is similar to using the 'readonly' modifier on a struct member, which changes the ref kind of the 'this' parameter from 'ref' to 'in'.

### Nullable annotations must be enabled to allow use of the 'default' modifier or defaultable annotation
```cs
#nullable enable
public partial class C3
{
    public default void DefaultableMethod1() { } // warning: The 'default' modifier should only be used in a '#nullable' annotations context.
    public C3~ DefaultableMethod2() => this; // warning: The annotation for defaultable types should only be used in a '#nullable' annotations context.
}
```

### Properties and fields on classes can specify that they are always initialized, even when the type is defaultable
```cs
#nullable enable
public partial class C3
{
    // Here, the class makes a guarantee to its user that this property is
    // always initialized by every constructor, and it should be assumed to be not-null
    // even on values of type 'C3~'.
    // Value types are not allowed to use this attribute
    [ConstructorInitialized]
    public string AlwaysInitializedProp { get; set; } = "default";

    public default void UseAlwaysInitializedProp()
    {
        this.AlwaysInitializedProp.ToString(); // ok
        this.Prop.ToString(); // warning: possible null reference receiver
    }
```

### Readonly fields are not assumed to be initialized by the constructor
```cs
// ... TODO: fill in this example
// Explicitly implemented 'init' accessors can write to these fields,
// therefore we don't assume the constructor initializes them.
```

### Methods can specify defaultability postconditions on their parameters
```cs
#nullable enable
public partial class C3
{
    [NotDefault] // a la [NotNull]
    public default void AssertInitialized()
    {
        if (this.Prop is null)
        {
            throw new InvalidOperationException();
        }
    }

    [NotDefaultWhen(true)]
    public default bool IsInitialized()
    {
        return this.Prop is not null;
    }

    public static void CheckInitialized1(C3~ c)
    {
        if (c.IsInitialized())
        {
            c.NonDefaultableMethod(); // ok
        }

        c.AssertInitialized();
        c.NonDefaultableMethod(); // ok
    }

    public static void InitAndUse(C3~ c)
    {
        // in addition to using special "is it initialized" methods you can also use structural methods if the members to initialize are all public.
        if (c.Prop is null)
        {
            c.Prop = "default value";
        }
        c.NonDefaultableMethod(); // ok
    }
}
```

### Defaultable types may be used as arguments to unconstrained generic parameters.
```cs
#nullable enable
public partial class C3
{
    public static ImmutableArray<C3~> MakeImmutableArray(C3 c3) => ImmutableArray.Create(c3);
}
```

### Method type arguments are "defaultable-reinferred"
```cs
#nullable enable
public partial class C3
{
    public static T[] MakeArray<T>(T t) { return new[] { t }; }

    public static void UseMakeArray(C3~ c3)
    {
        var array = MakeArray(c3); // Resulting type is `C3~[]`
        array[0].Prop.ToString(); // warning: possible null reference receiver

        if (c3.Prop is null) return;

        var array2 = MakeArray(c3); // Resulting type is `C3[]`
        array2[0].Prop.ToString(); // ok
    }
}
```

### Recursive or mutually recursive type hierarchies
```cs
#nullable enable
public class ListNode1<T>
{
    public T Value { get; }
    public ListNode1<T>? Next { get; } // to terminate the list, a null value is used for the "Next" node.
}

public class ListNode2<T>
{
    public T Value { get; }
    public ListNode2<T> Next { get; } // to terminate the list, the "Next" node points to itself

    public static ListNode2<int> CreateList1()
    {
        return new ListNode2<int>
        {
            Value = 1,
            Next = new ListNode2<int> // warning: non-nullable field Value is not initialized.
            {
                Value = 2
            }
        };
        // Suppose this literal were 5 or 10 or 100 levels deep:
        // we probably pick the same max depth as the nullable feature (5).
        // after that depth, we just don't analyze whether all fields are initialized.
    }
}
```
This is only really tricky if the recursive fields are non-nullable and not initialized by the constructor. In practice we are likely to find that many recursive or mutually recursive type hierarchies use nullable reference types along the recursive path.

However, the `ListNode2` scenario is an open question. We cannot plumb down indefinitely deep into containers of defaultable types to infer whether the defaultable members have been initialized. We also probably can't handle aliasing in the case of complex cyclical structures. I think in this case, we must require the user of such a recursive type to use nullable-suppression or for the type to have methods with defaultability postconditions. The compiler would not be able to fully check the implementations of these methods for correctness.

### Determining the set of members to track for defaultability
- This is easy to do *within* a constructor, but not so easy to do when you can only look at the public surface.
- We want to preserve encapsulation. Users should be able to implement the property in a variety of different ways and change that implementation without breaking their users.
- Another tricky aspect is that structs really have a "soft" encapsulation that we may need to account for. Its fields are part of the public contract even if they are not publicly accessible because size or layout(?) changes can break consumers.
```cs
// TODO: we need rules for how to decide whether explicitly implemented properties must be initialized.
```

### Defaultable constructors and the 'new()' constraint
A non-defaultable type meets the `new` constraint only if its parameterless constructor produces a non-defaultable instance.
```cs
public partial class C3
{
    public static T Factory<T>() where T : new() => new T();

    public static void UseFactory()
    {
        // warning: 'C3' cannot be used as a type argument for 'T' in 'C3.Factory<T>(T)' because its parameterless constructor does not produce a fully initialized instance. Consider using the type argument 'C3~' instead.
        var c3_1 = Factory(new C3());

        var c3_2 = Factory<C3~>(new C3()); // ok
    }
}

public class C4
{
    public string? Prop { get; set; }

    public static T Factory<T>() where T : new() => new T();

    public static void UseFactory()
    {
        var c3_1 = Factory(new C4());

        var c3_2 = Factory<C3~>(new C4());
    }
}

public struct S1
{
    public string Prop { get; set; }

    public static T Factory<T>() where T : new() => new T();

    public static void M()
    {
        var s1_1 = Factory(default(S1)); // warning: 'S1' cannot be used as a type argument for 'T' in 'S1.Factory<T>(T)' because its parameterless constructor does not produce a fully initialized instance. Consider using the type argument 'S1~' instead.
        var s1_2 = Factory<S1~>(default(S1)); // ok
    }
}

public struct S2
{
    public string? Prop { get; set; }

    public static T Factory<T>() where T : new() => new T();

    public static void M()
    {
        var s2_1 = Factory(default(S2)); // ok-- S2 doesn't contain any non-nullable reference types.
        var s2_2 = Factory<S2~>(default(S2)); // ok
    }
}
```
Intuitively it seems like a non-defaultable type argument `TNonDefault` should only meet the `new()` constraint if its parameterless constructor produces a non-defaultable type, either by explicit implementation or by not containing non-nullable reference types.

In effect, this means that structs that contain non-nullable reference types can *only* meet the `new()` constraint if the type argument is defaultable. (This will change if we change the language to allow explicit parameterless constructors on value types.)

### Types can implement interfaces "defaultably"
Usually it's expected that an instance needs to be fully initialized before it can safely be converted to interface and used. However, we may want to support a notion of "defaultable implementation" which enforces that all members implementing the interface members are defaultable:

```cs
#nullable enable
interface I
{
    string Prop { get; }
}

class C1 : I~ // C1 implements I "defaultably". In other words, C1~ implements I
{
    // warning: Defaultability of C1.Prop doesn't match implicitly implemented interface member I~.Prop
    public string Prop { get => this.ToString(); set { } }
}

class C2 : I~
{
    // ok
    public default string Prop { get => this.ToString(); set { } }

    static void UseC2(C2~ c2)
    {
        I i = c2; // ok
    }
}

class C1 : I
{
    public string Prop { get => this.ToString(); set { } }

    static void UseC1(C1~ c1)
    {
        I i = c1; // warning: c1 of type C1~ doesn't implement I. Conversion is only possible if c1 is found to be non-default by flow analysis.
        if (c1.Prop is not null)
        {
            I i1 = c1; // ok
        }
    }
}
```

### Combining defaultable and nullable type annotations
At a glance this looks just a little funky, but it seems like these features should be able to compose.
```cs
#nullable enable
public class C
{
    public string Prop { get; set; }
    public void M() => Prop.ToString();

    public static void M1(C? c)
    {
        if (c != null)
        {
            // c is assumed to be non-null and non-default here
            C.M(); // ok
        }
    }
    
    public static void M2(C~? c)
    {
        if (c != null)
        {
            // c is assumed to be non-null but still potentially defaultable here
            c.M(); // warning: c may not be fully initialized
        }

        if (c?.Prop != null)
        {
            // c is assumed to be non-null and non-default here
            c.M(); // ok
        }
    }
}
```

### "required" modifier on properties/fields to participate in defaultability? (couldn't rely on nullable anlaysis for this)

---

Some folks want to initialize their stuff in constructors. Some folks want to initialize their stuff using object initializers. Some folks want to initialize their stuff by new-ing up an instance or pulling it out of a pool and calling an Init() method. 

Currently the nullable reference types feature takes the view that constructors are responsible for fully constructing the instance--every field must contain a value that is valid for that field's type. This has left behind users who do not prefer to initialize their objects in the constructor.

One general path we are considering is to put modifiers on types and members to indicate: here's what needs to be initialized before the object is valid, and here are the members that get initialized by this constructor, or factory method, etc.. This is basically the "required properties" feature under consideration for C# 10.

## Annotating struct type declarations
In C#, we like explicit requirements around the usage of types and methods. For example, we prefer that a method declare explicitly whether its return value could be null, as opposed to the compiler inserting an annotation implicitly by analyzing the implementation of the method.

Non-defaultable structs intuitively fall into this category: if the default value of the struct isn't valid for performing certain operations with it, then we want to require special annotations of such struct types when the value could be the default value. Previous proposals have suggested use of an attribute like the following, to opt-in to the increased restrictiveness of the checks:

```cs
var arr = default(ImmutableArray<int>);
_ = arr.Length; // warning: possible default receiver?

[NonDefaultable] // or a similar attribute or modifier.
public readonly struct ImmutableArray<T>
{
    private T[] _array;
    public int Length => _array.Length;
}
```

In the above example `

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
- Members can add `[NotDefaultThis]` to indicate that the method only returns when the relevant argument is not-default, but should not warn when called on a maybe-default argument.
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