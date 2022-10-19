- Feature Name: relax_const_restrictions
- Start Date: 2022-10-06
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow `const` functions to behave differently during constant-evaluation and runtime and remove all restrictions from `const` functions if they are called at runtime.

# Motivation
[motivation]: #motivation

The past restriction of `const` functions having to behave the same way no matter where they were called has been a limitation and it has been unclear whether such a difference in behaviour could cause unsoundness. While the Rust language does, at the time of writing, not expose a way to determine whether a function has been called during constant evaluation or runtime (and this RFC does not propose adding such a feature), such an intrinsic (`const_eval_select`) does currently exist internally in `rustc`.

The precondition of this intrinsic has always been that the const-eval and runtime code have to exhibit the exact same behavior. Verifying this property about the two different implementations is often not trivial, which makes sound use of this intrinsic for non-trivial functions tricky. But it can often be desirable to use such an intrinsic to do various optimizations in runtime code that are not possible in constant evaluation.

Also, floats are currently not supported in `const fn`. This is because many different hardware implementations exhibit subtly different floating point behaviors and trying to emulate all of them correctly at compile time is close to impossible. Allowing const-eval and runtime behavior to differ could unlock const floats in the future, by accepting that floats can have subtly different behaviors at runtime.

Rust code often contains debug assertions or preconditions that must be upheld but are too expensive to check in release mode. It is desirable to also check these preconditions during constant evaluation (for example with a `debug_or_const_assert!` macro). This is unsound under the old rules, as this would be different behavior during const evaluation in release mode. This RFC allows such debug assertions to also run during constant evaluation (but does not propose this itself).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC allows `const fn` to exhibit different behavior during constant evaluation and runtime. While such a difference is often undesirable, it is not considered to be undefined behavior.

If a `const fn` is able to detect whether it has been called during constant evaluation or at runtime (either through an intrinsic or a future language feature), then it is allowed to exhibit different behavior. Also, a `const fn` called at runtime can do anything a normal function can do, with no additional restrictions applied to it. It could open a file, call a system randomness API or gracefully exit the program. 

At the time of writing, there is no way for a function to detect whether it was called at runtime or during constant evaluation in stable Rust and this RFC is not concerned with adding any, but it encourages future RFCs for adding this capability.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This RFC introduces a new concept called the "constness-context" of an execution of a function. A `const fn` is executed in a const constness-context if it was called in a context where only `const fn` are allowed (like `static` initializers) or inside another `const fn` that was executed in a const constness-context. All other calls to a `const fn` (for example in `main`) are in a runtime constness-context.

We can therefore say that code can either be:
- Always in a const constness-context (for example `static` initializers, array lengths...)
- Maybe in a const constness-context (`const fn`), where the constness-context can differ between calls depending on the call-site
- Always in a runtime const constness-context (non-`const` functions)

A `const fn` is now allowed to exhibit different behavior depending on it being called in a const or runtime constness-context. Language features and standard library functions may also differ in behavior depending on the constness-context they have been used or called in.

This makes the constness-context of a function observable behavior.

If a `const fn` is called in a runtime constness-context, no additional restrictions are applied to it, and it may do anything a non-`const fn` can (for example, calling into FFI).

# Drawbacks
[drawbacks]: #drawbacks

There are two main drawbacks to this RFC.

Firstly, pure `const fn` under the old rules can be seen as a simple optimization opportunity for naive optimizers, as they could just reuse constant evaluation for `const fn` if the argument is known at compile time, even if the function is in a runtime context. This RFC makes such an optimization impossible. This is not seen as a problem by the author, as a more advanced optimizer (like LLVM) is able to remove these calls at compile time through means other than Rust's constant evaluation (inlining and constant folding). Also, a constant evaluation system can still evaluate executions in a runtime constness-context, as long as it behaves exactly like runtime. The optimizer could also manually annotate functions as being truly pure by looking at the body.

Secondly, with the current rules around `const fn` purity, unsafe code could choose to only sometimes call unknown `const fn` code at runtime depending on whether it needs the result. The author does not see this as a significant drawback, as this functionality is better served by language features that target this use case directly (like a `pure` attribute) and is therefore out of scope for the language feature of "functions evaluatable at compile time". This could break code that already relies on this, but since Rust doesn't have proper support for this, the impact should be minimal.

The old rules, which say that `const fn` always has to behave the same way are already well-known in the community. Changing this will require teaching people about the new change. Since this is a very simple change, this should be easy.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative is to keep the current rules. This is bad because the current rules are highly restrictive and don't have a significant benefit to them.

It would also be possible to allow them to behave differently, but keep the restrictions around purity and determinism at runtime. This would still allow unsafe code to treat `const fn` specially, but this is not seen as a desirable feature of `const fn`.

# Prior art
[prior-art]: #prior-art

C++ with `constexpr`, a compile-time evaluation system similar to Rusts `const fn`, has a [`std::is_constant_evaluated`](std-is-constant-evaluated) function which can be used to determine whether the function is being executed during constant evaluation or at runtime. It does not impose restrictions that code has to behave the same during constant evaluation or runtime.


# Unresolved questions
[unresolved-questions]: #unresolved-questions

None for now.

# Future possibilities
[future-possibilities]: #future-possibilities

This unlocks various new things for `const fn`. For example, the subtle difference of floating point behavior may now be considered acceptable and floats could be allowed in `const fn`. An intrinsic like `const_eval_select` (in the form of an intrinsic or a more complete language feature) could now be added safely, enabling more parts of the ecosystem to make functions `const` without losing runtime optimizations.

[std-is-constant-evaluated]: https://en.cppreference.com/w/cpp/types/is_constant_evaluated