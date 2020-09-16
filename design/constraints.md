# Constraints

Lightbeam is a very unconventional compiler, because it is linear-time and streaming, while also
attempting to produce high-quality code that can run as fast as possible. Here we'll outline
specific constraints that Lightbeam needs to conform to that other compilers do not.

## Foreword: Why Linear-Time?

So, it's easy to take it as read that Lightbeam has to be linear-time - after all, that's why we're
building our own compiler in the first place - but it's worth taking some time out to explain why
this is the case. So we essentially want to put a price on the amount of time required to compile a
given module. There are a multitude of ways we can approach this, but I'm only going to talk about
what I'd consider the two strongest: pricing via gas and pricing via complexity bounds.

### Pricing via Gas

So this would basically require either manually pricing each section of the compiler or compiling
the compiler itself to WebAssembly, adding gas annotations to _that_ algorithmically, and then
keeping a count of how much gas is consumed. This has a few inherent problems:

1. It reduces the performance of the compiler significantly, as WebAssembly is currently nowhere near
  as fast as native. This may change in the future, but we currently use Cranelift as our most-
  optimising codegen backend and Cranelift produces pretty terrible code - even worse than
  Lightbeam's in many cases.
2. We have to make a version of Wasmtime that can be compiled to WebAssembly, which would be a _huge_
  rearchitecture. Wasmtime generates all its structures in a format suitable for direct execution on
  the platform it's running on, and those assumptions permeate the whole project.
3. Each platform needs its own version of the compiler even if it's been compiled to Wasm. Even
  though the compiler itself can be compiled to a platform-independent form, it _must_ still produce
  device-specific machine code. It wouldn't be much of a compiler otherwise. This means that for
  each platform the amount it costs to run the compiler is different, and even whether or not the
  compiler terminates at all may be different. This is fine for charging the person uploading the
  contract, as we can price it based on the platform of the node that generated the block in
  question, but now every node that wants to validate has to run both that node's version of the
  compiler for validation purposes _and_ their own verson of the compiler for actual codegen. I'm
  sure there are some solutions to this (since we're PoS with each block having precisely one
  creator we have some leeway that a PoW race-to-the-finish blockchain might not) but it's
  undoubtably incredibly complex.

### Pricing via Complexity Bounds

This is the method we've decided on, and it basically requires guaranteeing that there's some
trivially-measurable property of a module `x`, and some function `f`, such that for any module,
compilation takes less than or equal to `f(x)` time to complete. Then we can always charge some
amount derived from `f(x)` gas to compile the module. This implies guaranteeing that compilation
always halts for any input module, which has the nice property that we can statically guarantee that
any Wasm module is a valid smart contract for the sake of compilation. There are no contracts that
must be rejected for the sole reason that they cause a JIT bomb in the compiler itself.

So I wrote that above description in an extremely general way, because although we've chosen `x` to
be the length in bytes of the module and `f` to be `kn` for some constant `k`, we could choose any
`x` and `f`, but making compilation `O(length in bytes)` is really nice for a few reasons:

1. It doesn't encourage splitting or joining modules unnecessarily, if you had greater than `O(x)`
  complexity you'd encourage splitting modules (for example, once `2x^2 < (2x)^2` it'd be cheaper to
  use 2 modules), and if you had less you'd encourage joining modules, and this would make
  predicting the price of a module unnecessarily difficult.
2. Basing the price on the length of the module in bytes makes predicting price incredibly easy, and
  this is a value we will always have access to since we need it for other reasons. Most other
  properties of the module would require some calculation. Developers could also calculate the cost
  of compiling some contract they want to deploy in their head, by doing a constant multiplied by
  the length of the module in bytes.

## Limiting Maximum Lookahead

So since we're streaming and linear-time, we _must_ have strict bounds on the runtime of the
compiler WRT to the input module. Since we compile per-function and our complexity to compile a
single function is `Ω(number of instructions)` (since we need to at least _read_ every instruction),
we can say that our worst-case is if the module is a single, large function. Since there is an upper
bound to how many bytes a single instruction can take, we can say that, to a first approximation,
our runtime is `Ω(size of module in bytes)` if our runtime to compile a single instruction is
`Ω(1)`. Since we want to have an upper bound of `O(length of module in bytes)`, we can use the same
logic, to show that if we compile a single instruction in `O(1)` we will end up with `O(size of
module in bytes)` overall complexity. This is obviously not a rigorous proof, despite the
mathematical jargon, but let's talk about the ramifications here and how it leads to some design
decisions in Lightbeam.

So I've said that we need to compile each instruction in constant time, but this actually _doesn't
preclude having some lookahead_. We can treat a compiler with a worst-case `n`-instruction lookahead
as the same as a compiler compiling the module in chunks of `n` instructions for the sake of worst-
case complexity analysis, and so as long as we have `O(n)` complexity where `n` is the number of
instructions in our lookahead buffer, that means that for a constant upper bound for `n` we can
treat that as being the same as having `O(1)` complexity. This is _really_ important for
optimisation, but we must be careful.

So the complexity of the system looks like so: for each Wasm instruction, we produce `m` Microwasm
instructions, where `m` has a small, constant upper bound. We then process Microwasm instructions
with maxmimum 1-element lookahead (to check if a branch's target is immediately defined after the
branch, in which case we don't emit a `jmp`). So now we "just" need to make sure that for each
Microwasm instruction, the code generating assembly from it is `O(1)` in time and space. This is far
from complete, the most glaring issue right now is that locals are run-length encoded in Wasm but we
reserve space for them immediately, meaning that for any 1-byte definition of locals we can reserve
a huge amount of heap space. This would be fixed by instead lazily initialising space for these
locals when we first set them, using some custom data structure based on a `HashMap`.

The current attempt to strictly bound complexity relies on complexity bounds of standard library
data structures, most notably `O(1)` insert/get/update for `HashMap`. The standard library `HashMap`
only guarantees _amortised_ `O(1)` complexity, and so in the future it would be good to analyse
whether this actually causes any issue for pricing contract compilation.

The main place that this breaks down is currently in how we handle passing arguments, which has an
infinite loop which it breaks out of when a position for every argument has been found. Significant
effort has been made to guarantee that this loop always terminates, but little effort has been made
to analyse and strictly bound the runtime of this loop. This is the first place to research when we
get to the point of strictly guaranteeing runtime for Lightbeam.

## How IR is Structured

So as I touch on in the previous section, we can have as many intermediate representations as we
like, as long as at each step we guarantee `O(1)` output instructions are generated for each input
instructions (modulo the aforementioned caveats about lookahead and constant upper bounds). We
currently use 1 IR, but as long as we keep adhering to the constraints in the previous section we
can still have an overall `O(n)` runtime. This is how we can have this idea of LIRC storing a buffer
of Low IR instructions and querying the machine specification for that sequence can still maintain
`O(n)` overall complexity, as it's querying for _precisely one_ output machine code instruction, it
_must_ find a single candidate output instruction at every step (so that it doesn't need to roll
back an arbitrary number of instructions, only one), and so the maximum lookahead is precisely the
number of Low IR instructions in the machine spec's instruction definition with the highest number
of Low IR instructions. For example, if the machine spec has 3 instruction definitions with 1, 5,
and 3 Low IR instructions in the corresponding sequence, then LIRC knows that once it reaches 6 Low
IR instructions in its lookahead buffer that it will never find an instruction that matches that
sequence, and so we have a strict, constant upper bound to lookahead. This same process can be done
for any proposed IR to prove that our overall complexity remains linear.

## Data Structures

Every single data structure in the project _must_ be `O(1)` for every operation that is done during
compilation. This usually means restricting ourselves for `HashMap`s or only accessing
arrays/vectors using specific, stored indices. You can see in our stack allocation subsystem that
this fact heavily influenced the design and what information ends up being stored.
