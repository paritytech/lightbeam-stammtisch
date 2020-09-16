# Microwasm

> **NOTE**: the phrase "Lightbeam IR" is often used to refer to Microwasm outside of the specific
> context of Lightbeam development, since it's a little more descriptive and can be easier to
> understand for outsiders. However, within discussions on Lightbeam development "Microwasm" is
> still used, as that's the name used in the code itself and it prevents the term "Lightbeam IR"
> from being overloaded if and when further levels of intermediate representation are introduced.

Microwasm is a typed, safe, stack-machine IR used internally in Lightbeam to simplify various parts
of the compilation process. I go into deeper detail on the rationale behind the original
introduction of this IR in the [WebAssembly Troubles][wasm-troubles] series on my blog, but this
document should be enough to explain what Microwasm is, how it works, why it's useful as well as
general thoughts about it. For proposals to extend or change Microwasm, see the
[proposals][proposals] section.

So the difference between Microwasm and Wasm is pretty much entirely in the way control flow is
handled, although some instructions are encoded differently in order to allow code that doesn't care
about types to easily handle instructions that are identical apart from type (such as the `const`
family). The motivation for the control flow changes are described in the aforementioned Troubles
series, so I'll just cover what those changes actually are here.

So, WebAssembly sees control flow much the same as any structured programming language, with nested
blocks that you can jump out of, and `if` and `loop` blocks explicitly marked out. However, this
leads to redundancy that must be handled. For example, the following two sequences act the same:

```
local.get 0
if
  i32.const 0
  return
else
  i32.const 1
  return
end

;; .. is the same as ..
block
  local.get 0
  br_if 0
  i32.const 0
  return
end
i32.const 1
return
```

Additionally, unreachable code leads to extremely complex and confusing problems with respect to
types. To manage this, all control flow in Microwasm is condensed to a single form, and code after a
branch is statically impossible (unused blocks are still possible though, as it doesn't complicate
matters to allow them). This form basically has every branch be infallible. If a branch is
conditional, it _must_ specify a target for if the condition is false. This means that the two
examples above would compile to the same Microwasm and be handled the same by the backend, which has
the nice benefit of allowing optimisations benefitting one (such as making the branch infallible if
the local has a constant value) can benefit both forms. We also condense all branch types to a
single type, which is essentially the same as Wasm's `br_table`. This is because `br_table` with
only a default target is equivalent to `br`, and `br_table` with precisely one target and a default
is equivalent to `br_if`. As `br` doesn't consume a value to act as a condition, we additionally
generate an `i32.const 0`-like statement so that all forms can be handled precisely equally. This
means that no matter what kind of branch instruction is used in the input Wasm, there is only a
single code path to handle all of them. This makes the code a lot more reliable, as bugs only need
to be fixed in a single place.

For more information, see `microwasm.rs` in the Lightbeam source code, which has been documented.

[proposals]: ../proposals/microwasm/
[wasm-troubles]: http://troubles.md/posts/wasm-is-not-a-stack-machine/
