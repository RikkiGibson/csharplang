# Defaultable types

Certain members on struct types cannot be safely accessed on a `default` receiver.
This proposal is to allow struct type declarations to indicate that they contain such members,
and to allow annotation of struct types throughout the language to indicate that values of the type could be `default`.

## Goals
- Allow indicating that members on value types require a non-`default` receiver to access safely.
- Allow indicating that members on reference types require a "fully initialized" receiver to access safely.

## Alternatives

It feels like `default` is not general enough to let the feature address both the value and reference type scenarios, particularly because the default value of a reference type is `null`. Is it possible that a different term would be better in order to make the concepts and behavior more clear?

- `init` modifier on members doesn't seem right--that's about a member that *performs* initialization, not a member that *checks* if a thing is initialized.
- It feels like `valid`/`invalid` may be a more understandable dichotomy.

---

Struct type declarations can be annotated to indicate that the type is "non-defaultable" when it is not annotated.

```cs
[NotDefault]
public struct ImmutableArray<T> : // ...
{
    // ...
}

void M(ImmutableArray<string> arr)
{
    _ = arr.Length;
}

M(default); // warning: converting possible default value to non-defaultable type.
```

---

Struct type usages can be annotated as "defaultable".

```cs
void M(ImmutableArray<string>~ arr)
{
    _ = arr.Length; // warning: possible default receiver
}

M(default); // ok
```

This feature introduces the `~` annotation for value types. A few open questions:

1. Can `~` be used on class types?
  - This feels like it could be useful with `UnityEngine.Object`, where uninitialized instances are treated as equivalent to `null`. 
1. Can `?` be combined with `~`?

- Struct types are assumed to be defaultable unless the type declaration has the `[NotDefault]` attribute.

```cs
public struct Point
{
    int X { get; set; }
    int Y { get; set; }
}
```

There are 2 basic ways to go with this:
1. Assume that all struct members are not safe to access on a `default` receiver, unless the member or containing type is annotated with `[AllowDefault]` or similar.
2. Assume that all struct members *are* safe to access on a `default` receiver, unless the member or containing type is annotated with a `[NotDefault]` or similar.

For this proposal we're going with (2) because many structs don't require a non `default` receiver to safely access members, for example because they don't contain non-nullable reference type fields.

---

- A warning is produced when a defaultable struct type declaration contains a non-nullable or non-defaultable type field

```cs
public struct Widget
{
    ImmutableArray<string> Prop1 { get; set; } // warning: non-defaultable auto-property 'Prop1' is not permitted on defaultable struct 'Widget'. Consider adding a '[NotDefault]' attribute to 'Widget'.
    ImmutableArray<string>~ Prop2 { get; set; } // ok

    string Prop3 { get; set; } // warning: non-defaultable auto-property 'Prop3' is not permitted on defaultable struct 'Widget'. Consider adding a '[NotDefault]' attribute to 'Widget'.
    string? Prop4 { get; set; } // ok
}

public struct Widget<T>
{
    T Prop1 { get; set; } // warning: non-defaultable auto-property 'Prop1' is not permitted on defaultable struct 'Widget'. Consider adding a '[NotDefault]' attribute to 'Widget'.

    // does this warn or not?
    // what if `?` on unconstrained generics turned into `~` after substitution when the type argument is a value type?
    T? Prop2 { get; set; }
}

public struct Widget<T> where T : class?
{
    T Prop1 { get; set; } // warning: non-defaultable auto-property 'Prop1' is not permitted on defaultable struct 'Widget'. Consider adding a '[NotDefault]' attribute to 'Widget'.
    T? Prop2 { get; set; } // ok
}
```

This warning is given regardless of whether a parameterless constructor is defined in the struct, because the declaration itself indicates that `default` is always a valid value of the type.

This basically means we're asking every struct type that contains such fields to add this attribute to the declaration. It's not clear if this is too significant of a breaking change or not--when people update their target framework and language version, they may start receiving these warnings en masse on struct types which contain such fields.