# CPS in Microwasm

So right now the control flow of Microwasm is pretty simple. It's just basic blocks (not even
extended basic blocks), and calls are modelled as being identical to other operators, for example
adds. While this handles the existing form of Wasm well, and allows us to collapse control flow into
a single primitive, it isn't without its flaws. The most glaring one is that exceptions are likely
to come to WebAssembly eventually, and exceptions are going to be _incredibly_ hard to implement
with the current form of Lightbeam (although the proposed new codegen system may make it somewhat
easier). Exceptions destroy a lot of expectations that Lightbeam has about control flow in general,
and one could easily imagine that exceptions are going to be a very common cause of miscompilations
and such. Another issue is that `BrTable` is implemented using a pretty dirty hack that works, but
isn't very cleanly implemented or easy to change.

My suggestion is to have a system where we collapse all control flow into a single kind of
unconditional jump, and remove returns in favour of passing a continuation pointer - this would be
detected and converted back into `call`/`ret` by the backend. `BrIf`/`BrTable` can be implemented by
selecting between multiple targets, and then emitting a branch. Function calls are just the same as
any other kind of jump, except that they take a continuation which is how they do the return value.
It is then the job of the backend to "undo" this transformation, converting it back into the simple
hierarchical control flow that the machine implements.

So, why do this? Well, it gives us a lot of stuff "for free", and allows us to implement a lot of
optimisations by adding metadata-recording passes that are easily checked to be linear-time. For
example, a function that happens to never throw an exception can get the same elision of exception-
handling code that causes a block which is never called to be omitted. Likewise, it's trivial to
check this form for functions that never return, and this gives us DCE of code after functions that
don't return "for free". Try and catch can be implemented by adding an extra parameter to functions
which is the same type as returning, and it means that returning by throwing an exception and
returning "normally" are implemented identically, which makes it near-impossible to miscompile
exception handling.

We can implement optimisations that work for the code emitted by `BrIf` and `BrTable` (i.e., having
a value which is known to be one of a set of statically-known branch targets, then jumping to it)
and get optimisations which allow `call_indirect` to act as a simple test-and-jump as long as the
possible targets for the `call_indirect` are known.

Finally, and this is the real reason, this allows asynchrony in Wasm for essentially no cost.
Languages like Go could compile to Microwasm even though they can't truly compile to Wasm, since CPS
is the natural fundamental expression of asynchronous code. This makes it far simpler to implement
an asynchronous Wasm runtime, where execution can be arbitrarily halted on an external call and then
resumed later.
