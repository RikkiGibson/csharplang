## Argument state after call for AllowNull parameters
https://github.com/dotnet/csharplang/discussions/4102

```cs
string? s = null;
M(s); // warning
s.ToString(); // no warning

void M(string s) { }
```

```cs
string? s = null;
M(s); // no warning
s.ToString(); // no warning..?

void M([AllowNull] string s) { }
```


```cs
// This is a scratchpad for C# as well as a place to jot down meeting notes.

// jared: If these signatures are equivalent for override
override void M([AllowNull] string s) { }
override void M(string? s) { }

override void M([DisallowNull] string? s) { }
override void M(string s) { }

```


```cs
virtual void M(string s) { }
override void M([AllowNull] string? s) { }
```

### old rules proposed in discussion
- the declared type (sans attributes) of a by-value or `in` parameter does not update the argument state after the call. We likely want to *preserve* the `[NotNull]` postcondition behavior to enable writing `AssertNotNull([NotNull] object? obj)`.
- if a by-value or `in` parameter does not accept a null argument, then the argument state must be updated to not-null after the call (similar to how a call receiver is updated to not-null after the call).
- For `ref`/`out` parameters, the argument state after the call is the effective output type of the parameter, e.g. not-null for `ref string s` as well as `[NotNull] ref string s`.

### rules written today
- if a safety warning was produced for an argument to an input parameter, its state is updated to not-null
- if a parameter has `[NotNull]`, the argument state is updated to not-null

```cs
void M([AllowNull] ref string s) { s = "a"; }

void M2([AllowNull, NotNull] object obj) { }

void M3([AllowNull] object obj) { } // discouraged, except for overrides
void M4<T>([AllowNull] T obj) { }

void M5([NotNull] object? obj) { }
void M6(object obj) { }

object? obj = null;
M2(obj); // no warning
obj.ToString(); // no warning. this is by-design

void M7([MaybeNull] object obj) { }

JsonConverter<T>.WriteJson(JsonWriter writer, [AllowNull]T value, JsonSerializer serializer);
value.ToString(); // no warning (currently)
```

### Summary

We recently met and discussed how to alleviate issues we've found with the [`[AllowNull] string s`](https://github.com/dotnet/csharplang/discussions/4102)  parameter scenario. Our proposal is to modify the rules for nullable argument analysis as follows:
- An input parameter doesn't update the argument state after the call based on its declared type.
- When we give a safety warning on an argument, we update the argument state to not-null.

### Discussion

AllowNull/DisallowNull and MaybeNull/NotNull were initially designed with an emphasis on the respective concepts of "input state" and "output state" of members and parameters. However, we found that in the absence of `T?`, developers used `[AllowNull]` a stand-in for `T?` on input parameters. For example, the [`JsonConverter<T>.WriteJson([AllowNull]T value)` scenario](https://github.com/JamesNK/Newtonsoft.Json/blob/cdf10151d507d497a3f9a71d36d544b199f73435/Src/Newtonsoft.Json/JsonConverter.cs#L108). It feels like the concept of a separate "input state" and "output state" on input parameters is not desirable to include by default.

Instead, we think the better mental model is to say that an input parameter usually only has an input state. By default there is no "output state" being applied to the argument. Instead, we say that if a safety warning is given for the argument (due to failing to convert to the parameter type), the argument's state should be updated to not-null. In this model, `[NotNull]` or `[MaybeNull]` can be used on input parameters to introduce the appropriate output state to the parameter. We expect this to be infrequently used, mostly for `AssertNotNull()`-style methods.

This proposal addresses the main pain point that was raised, by *not* updating argument state to not-null in more scenarios:
```cs
string? s = null;
M(s);
s.ToString(); // new warning

void M([AllowNull] string s) { }
```

However, behavior around `ref/out` parameters will not change in practice.
```cs
string? s = null;
M(ref s);
s.ToString(); // no warning

void M([AllowNull] ref string s) { }
```

This scenario is also unchanged.
```cs
string? s = null;
AssertNotNull(s);
s.ToString(); // no warning

void AssertNotNull([NotNull] string? s) { }
```

Please let us know any feedback you may have on this proposal.