# Improved Definite Assignment Analysis

## Summary
[Definite assignment analysis](https://github.com/dotnet/csharplang/blob/master/spec/variables.md#definite-assignment) as specified has a few gaps which have caused users inconvenience. In particular, scenarios involving comparison to boolean constants, conditional-access, and null coalescing.

## Related discussions and issues
https://github.com/dotnet/csharplang/discussions/801#issuecomment-553941347
https://github.com/dotnet/roslyn/issues/45582
https://github.com/dotnet/roslyn/issues/33559

Probably a dozen or so user reports can be found via this or similar queries (i.e. search for "definite assignment" instead of "CS0165").
https://github.com/dotnet/roslyn/issues?q=is%3Aclosed+is%3Aissue+label%3A%22Resolution-By+Design%22+cs0165

## Scenarios

```cs
#nullable enable

C? c = new C();

// happy case
if (c != null && c.M(out object obj0))
{
    obj0.ToString(); // ok
}

// comparison to bool constant
if ((c != null && c.M(out object obj1)) == true)
{
    obj1.ToString(); // undesired error
}

// conditional access, comparing to bool constant
if (c?.M(out object obj2) == true)
{
    obj2.ToString(); // undesired error
}

// conditional access, coalescing to bool constant
if (c?.M(out object obj3) ?? false)
{
    obj3.ToString(); // undesired error
}
else
{
    obj3.ToString();
}

public class C
{
    public bool M(out object obj) { obj = new object(); return true; }
}
```

We introduce a new section **?. (null-conditional operator) expressions**.

For an expression *expr* of the form `left?.right`:

- The definite assignment state of *v* before *left* is the same as the definite assignment state of *v* before *expr*.
- The definite assignment state of *v* before *right* is the same as the definite assignment state of *v* after *left*.
- The definite assignment state of *v* after *expr* is the same as the definite assignment state of *v* after *left*.

We amend the section [?? (null coalescing) expressions](../spec/variables.md#-null-coalescing-expressions)

> For an expression *expr* of the form `expr_first ?? expr_second`:
> 
> - If *expr_first* is a conditional access expression of the form *left?.right* and *expr_second* is a constant expression with value `false`, then the state of *v* after *expr* when-true is the same as the state after *right*, and the state of *v* after *expr* when-false is the same as the state after *expr_first*.
> - If *expr_first* is a conditional access expression of the form *left?[right]* and *expr_second* is a constant expression with value `false`, then the state of *v* after *expr* when-true is the same as the state after *right*, and the state of *v* after *expr* when-false is the same as the state after *expr_first*.



### random notes
- specifying 'is' based on patterns that null-check
- specifying definite assignment state of when-clause
- specifying definite assignment state of ?. in terms of language grammar