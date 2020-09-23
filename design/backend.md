# The Backend

> Relevant files:
>
> - [wasmtime/crates/lightbeam/src/backend.rs][backend.rs]

The backend is where most of the complexity of Lightbeam lies. Its most important jobs are handling
passing arguments to blocks and functions, allocating registers, and doing instruction selection.

It works on an instruction-by-instruction basis. Many of the instructions are very simple - anything
purely stateless such as addition and subtraction has a pretty trivial implementation - but it's
worth talking about some complications that are visible even in these simple instruction
implementations:

- We do pretty extensive constant folding, as doing so requires little extra code and it allows us
  to simplify some upstream code by converting more-complicated Wasm instructions into a couple
  Microwasm instructions that then get folded back together by the backend. This unfolding/refolding
  process is pretty common in compilers and it's the secret sauce that allows a _lot_ of the
  optimisations that allow Lightbeam to produce code with performance often competitive with
  Cranelift.

- We fold memory access into instructions where possible, so long as these accesses are to values on
  the physical stack. There's no problem with doing the same for heap values, as long as we
  immediately emit a load if we ever need to store to the heap to avoid invalidation. We simply
  haven't implemented that optimisation yet. You can see a similar process with condition codes,
  where we attempt to keep values in condition codes but always invalidate it if we try to push
  anything above that value. A better system would be to actually track which condition codes are
  and aren't invalidated, but doing so isn't possible before the LIRC refactor as we don't have any
  way to track that information.
  - Actually, it might be worth investigating if there's a better way to invalidate condition codes
    with the information we have now, as it's possible that we'll invalidate the condition code in
    question before we emit the `setcc` that we do when invalidating it. This optimisation is worth
    trying to keep as it _significantly_ improves the performance of branches (without it we
    increase register pressure, which Lightbeam already handles poorly, and increase the number of
    instructions needed to pass parameters between blocks), but it's possible that there exists an
    instruction that causes Lightbeam to miscompile because of it.
- We abstract a lot away behind helper methods, so most instruction implementation methods have the
  ability to emit far more instructions than just the ones in the body of the implementation. Any
  time you see `self.{method name}(...)`, if that method takes `&mut self` then it's probably
  because it has the ability to emit instructions.

## The Real Complexity

So, as always, the most complex component of the backend is implementing control flow. We abstract,
as much as possible, the difference between passing values to callee functions and to blocks in the
same function. This is done by separating "concrete" locations of values from locations that can be
anything. This means we can use identical code for both function calls and intra-function jumps, but
it also allows us to produce better code when we do a conditional jump where one target already
requires specific locations for values and the other(s) do not, which is actually a case that
happens quite often.

[backend.rs]: https://github.com/bytecodealliance/wasmtime/blob/master/crates/lightbeam/src/backend.rs
