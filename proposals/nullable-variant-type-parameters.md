# Nullable-variant type parameters

* [x] Proposed
* [ ] Prototype
* [ ] Implementation
* [ ] Specification

## Summary
[summary]: #summary

Allow type parameters to specify that they can be safely converted to/from nullable reference types.

## Motivation
[motivation]: #motivation

Certain "container" types such as `Task<T>` are safely convertible from e.g. `Task<string>` to `Task<string?>` because users don't have the ability to write to the task's result field. However, there is no way to express this in the language today. Because the type parameters of non-interface types cannot be variant, assigning a `Task<string>` to a variable of type `Task<string?>` produces a warning that needs to be suppressed with `!`.

