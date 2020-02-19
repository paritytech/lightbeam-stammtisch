# New Codegen Subsystem

This folder contains design documents for a proposed new codegen subsystem for Lightbeam, designed
to improve separation of concerns, improve the cleanliness and conceptual simplicity of Lightbeam,
and reduce the amount of code that needs to be written to allow Lightbeam to compile for a new
architecture. This subsystem has three major components:

- [Low IR][low-ir]: an extensible, low-level, untyped IR. It's more correct to refer to this as a
  "family" of IRs, as the only thing that must be shared is a small set of basic actions that LIRC
  (see below) needs to implement register allocation, maintaining the stack and so forth. In theory,
  every architecture can have its own version of "Low IR", but in practice these IRs will be very
  similar between architectures, the only differences being for architecture-specific instructions
  that need to be specifically requested, such as Intel extensions implementing cryptographic
  algorithms. The similarities between the IRs is why I refer to it as a single IR ("Low IR"), but
  the only thing that matters is that the form of Low IR used to write implementations of the Wasm
  instructions is the same one used to define the behaviour of the machine ISA's instructions.
- [AsmQuery][asmquery]: a "behavioural query system", which allows a compiler to request an
  instruction that implements a certain behaviour instead of asking for the encoding of that
  specific instruction. The developer defines instructions in the target architecture in terms of a
  RISC IR (the "Low IR" for that specific architecture), along with information about constraints
  such as specific registers or sets of registers that must be used. Then, a series of Low IR
  instructions
- [LIRC][lirc] (the "Low IR Compiler"): a simple, architecture-independent register allocator and
  glue layer between the Low IR implementations of Wasm instructions and the AsmQuery behavioural
  query engine. While it can be referred to as "the Low IR Compiler", the only thing that matters is
  that the IR that it's compiling and the IR that the query backend supports is the same, and it
  does not compile a _specific_ IR but rather any IR that:
  1. Is supported by the query engine - i.e. if you have a query engine that defines the x86
    instructions in terms of the "x86 flavour" of Low IR, then LIRC with that query engine will be
    able to compile the "x86 flavour" of Low IR.
  2. Has a small number of generic actions that LIRC needs in order to do its job - this means an
    action that can transfer a value from one location to another (this action would compile down to
    `mov` on x86, for example), an action that performs control flow of various kinds, an action to
    store or load to memory, an action for adding or subtracting (to push and pop from the stack),
    and so forth. It should be noted that it's the _IR being compiled_ that needs actions that do
    these jobs, LIRC doesn't care whether the target architecture can actually perform these actions
    as long as it has a way to _create a query_ for these actions, although of course instruction
    selection will fail at runtime if the query engine returns nothing when queried for this action.

The main layout of these systems is similar in concept to GCC's RTL and Machine Definition
subsystems, but as I'll explain in the Alternatives section, we cannot simply copy that system
whole-cloth.

## Issues with the current system

I've front-loaded this document with my proposed solution, but I want to go over the actual problem
that needs solving in the first place. The current codegen system, while functional, is brittle and
hard to work on for anyone not deeply familiar with the invariants that need to be upheld, and it's
almost impossible to correctly document all of these invariants (see the design document for the
[backend][backend]). Here are some of the specific issues that I see with the current codegen
system, and what qualities that I would consider important in a project that intends to solve these
issues.

### Issue 1: Reliability/Auditability

The first reason for creating these new components is that Lightbeam's current instruction selection
and register allocation subsystems are deeply tied together, and very tied to the one specific
architecture that we currently support (x86_64). This means that we must have all of this work done
in one, large file, with all the invariants of each having to be maintained manually. The register
allocator works using reference counting, and obviously when control flow merges, the reference
count from each source branch must match for each register. For the most part we have an abstract
representation of control flow, and so it's trivial to ensure that we're correctly maintaining these
invariants. However, when the implementation of an instruction needs control flow internally, then
we have an issue. Since our instruction selection is just emitting bytes without any abstract
representation, we must manually ensure that if any of those bytes are doing control flow, that the
register allocator's internal state at each point is correct. For example, if we want to emit a
fork:

```asm
  cmp %rax, %rdx
  jge .else_branch:
.then_branch:
  ;; do something
  jmp .cont
.else_branch:
  ;; do something else
.cont:
```

We must make sure that if we allocate any registers in `then_branch`, that those registers _must_ be
freed before `jmp .cont`, and likewise for `.else_branch`. This is a very easy source of bugs. So
we'd like the register allocator to have insight into _all_ control flow, both between Wasm
instructions and within the implementations of those instructions. This means that we want to write
the implementations of Wasm instructions in kind of IR which is low-level enough to be able to do
everything that the machine code can do, but leaves the precise locations of values abstract, so
that it can be handled at a higher level. This IR can also define control flow abstractly - i.e.
jumping to labels instead of jumping to bytes. Then the register allocator has information about
when control flow is happening, and so we only need to make sure that where the register allocator
handles control flow (which should be in precisely one place) it maintains all the important
invariants, as opposed to having to ensure this in many, many, many places.

### Issue 2: Retargetablility

Right now, there is essentially no instruction selection algorithm. Instruction selection is done by
manually writing out x86 assembly language, with no abstraction, and this is deeply tied to other
parts of the code such as relocations, register allocation and so forth. This means that
implementing a new target for Lightbeam would involve reimplementing huge amounts of code.
`backend.rs` has over 5000 significant lines of code, and pretty much all of it would have to be
rewritten in order to target, for example, ARM. This is over half the SLOC of the whole of
Lightbeam, and even more if you only count the parts of Lightbeam that are actually used in Wasmtime
(the rest is devoted to a simple runtime, used as a test harness before Wasmtime was integrated).
This is not a reasonable amount of code to rewrite for every backend, not only by pure volume but
also because close to 100% of this code must be written very carefully to ensure that no
miscompilations occur, and then if a bug is found in one backend the other backends must be manually
checked to ensure that they can't hit that bug too. We need some way of abstracting out register
allocation, calling convention handling and instruction selection, as the vast majority of
instructions are very simple and so a lot of code can be shared.

However, there is a catch. We can't simply have some low-level IR and then compile that same IR to
all kinds of different environments. For example, if we wanted to load a value from memory, do some
operation on it, and then store that value back to memory, on ARM we'd want to calculate a memory
address once, save it to a register, and then reuse that memory address for both the load and store.
On x86, however, we'd want to recalculate that memory address twice, since we can do it as part of
other instructions and so it saves us the instruction. This is just an example, of course. The point
is that there's always going to be _some_ reason to have different IR for each architecture, even if
it's only for a single instruction. So, although we want the IR to be generic so much code can be
shared between architectures, we also want to allow architecture-specific code to be written if and
when we need to.

### Issue 3: No Framework for Optimisation

Lightbeam, as it currently exists, has no true optimisation subsystem, just lots of optimisations
that are built into the architecture or the implementations of the instructions themselves. There's
no single place that new optimisations can be added, and each instruction must be optimised
independently and manually. Const-folding, for example, has to be implemented separately for both
32-bit and 64-bit versions of all arithmetic instructions, and there's no framework around const
folding or other optimisations that would allow us to, for example, enable and disable instructions
to find bugs. We should have some way to separate out optimisation from codegen, so that parts of
the code that shouldn't care about optimisation don't have to be cluttered with code that implements
optimisations, and so that the effects of optimisations, and when and where they occur, is more
explicit. This would allow us to realise where code is and isn't being optimised, further
optimisations that we could implement, and isolate places where optimisations are being executed
incorrectly.

## Proposed Solution

I propose a system in 3 parts, as outlined in the introduction. I dive deeper into each component in
their respective articles, but as a high-level overview to explain what I believe are the benefits
to this system, I'll elaborate on a few examples that showcase what I believe to be the relative
elegance of this approach in the context of Lightbeam, with explanations of how the system will
handle these examples. The short summary of what I consider to be the benefit of a system like this
comes down to _explicitness_. Whereas with the current system we need to keep many, many implicit
constraints in our head, and maintain them manually, with a system that has explicit information
about the registers used as input and registers affected by the output of instructions we can be
sure that we're never accidentally overwriting any values that we need. This would be beneficial
even if the actual method of writing the assembly was the same - i.e. if we still explicitly wrote
out x86 instruction sequences - because we'd at least have _some_ way of finding bugs. This was the
original system that I envisioned: a system that didn't have a mid-level IR layer, but that had a
way to check correctness of the assembly. I'll go over why I rejected this idea in the Alternatives
section later.

> **NOTE**: I will exclusively use _instruction_ to refer to a specific opcode in the target ISA
> (x86, ARM, etc), and I will use _action_ to refer to a single operation in Low IR, this will look
> like `LOWIR::xxx.yy`, where `xxx` is the opcode of the action and `yy` is the size in bits. For
> example, 64-bit add is `LOWIR::add.64`, and 32-bit add is `LOWIR::add.32`. This is just shorthand
> pseudocode for now, I elaborate more on how I imagine this will actually look in the codebase in
> the [Low IR article][low-ir]. The distinction between _action_ and _instruction_ is very
> important, and so I will go out of my way to ensure that it is always clear which I am talking
> about.

### Example: Optimising Memory Accesses

For an example of how Low IR works, here's an example of some code in Low IR that loads a value from
memory and then multiplies it with some other value (in strawman syntax).

```
%fooaddr = LOWIR::add.64 %stackptr, 10
%foo     = LOWIR::load.32 %fooaddr
%bar     = ;; .. snip ..

%out     = LOWIR::mul.32 %bar, %foo
```

You can see that the memory address calculation and loads are explicitly written out here. LIRC will
pass this sequence of actions to AsmQuery, which will return a sequence of instructions that
implements the required behaviour. AsmQuery will have Low IR defined for instructions in the target
ISA like so (in pseudocode):

> **NOTE**: I will often describe LIRC passing a "sequence of actions" and receiving a "sequence of
> instructions", which makes the process sound like it is implemented eagerly. In fact, this process
> is incremental, and LIRC's state needs to be updated after each instruction is emitted anyway. I
> elaborate on this in the [article for LIRC][lirc]. For simplicity though, I want to focus on the
> conceptual passing of data and not the implementation in this overview.

```
definstr "mul r32, r32"(%lhs: Register, %rhs: Register):
  location(%out) = location(%lhs)

  %out      = LOWIR::mul.32 %lhs, %rhs

definstr "mul r32, [r32]"(%lhs: Register, %rhs_addr: Register):
  location(%out) = location(%lhs)

  %rhs      = LOWIR::load.32 %rhs_addr
  %out      = LOWIR::mul.32 %lhs, %rhs

definstr "mul r32, [r32 + imm32]"(%lhs: Register, %rhs_base: Register, %rhs_imm_offset: Immediate):
  location(%out) = location(%lhs)

  %rhs_addr = LOWIR::add.32 %rhs_base, %rhs_imm_offset
  %rhs      = LOWIR::load.32 %rhs_addr
  %out      = LOWIR::mul.32 %lhs, %rhs
```

If we look at the third form of the instruction, for example, this basically says that internally,
`mul r32, [r32 + imm32]` adds the base (which is in a register) and the offset (which is an
immediate) together, loads the value at the address given by the result of this addition, and then
multiplies the result of this load by the LHS (which is in a register). The `location(%out) =
location(%lhs)` statement says that the result of this multiplication is written to the physical
location of `%lhs`. You'll notice that the intermediate values (the result of the addition to
calculate the address, and the result of the load) don't have any location defined. This means that
although later parts of this instruction _can_ access these values, once this instruction has been
emitted those values become inaccessible. Let's say that instead of just using that loaded value in
the `mul.32`, you also need it later, as an argument for some other action. In this case, if there
is no single instruction that does the load, the `mul`, _and_ this other action, then you'll need to
first emit an instruction that loads the value to a physical location, _then_ a `mul` with that
physical location, _then_ the other action. Because this information is declaratively defined, there
doesn't need to be any special-casing here, to compile a sequence of actions to a single
instruction, we only need to ensure that all values that are still reachable after that sequence
have physical locations tied to them.

There's no "return" statement, as it's possible for instructions to leave many values in many
places. I didn't write it out in these example instruction definitions, but `mul` also sets flags
based on its inputs, and so in addition to `%out = ...` you can have `%is_zero = LOWIR::is_zero.32
%out` for instructions that set bits based on whether their output is zero. The only thing that
matters to LIRC is that if it wants a value to be preserved between instructions, it needs to have a
defined location. That can either be a location defined in terms on the instruction's arguments (for
example, for `mul r32, ..` we can say that the output of the `mul` is written to the same location
as the first argument), or as a specific location, as is the case with `%is_zero` (where we would
define `location(%is_zero)` to be the `ZF` bit of the `FLAGS` register). LIRC only cares that the
instruction definition defines physical locations for all values that are still live at the end of
the sequence of Low IR instructions.

> **NOTE**: I cannot stress enough that these `LOWIR::xxx.yy` instructions are just an arbitrary
> string. AsmQuery only cares that it was asked for an instruction that does the sequence: get the
> result of `add.32`, pass the result to `load.32`, pass the result to `mul.32`. If you had some
> wild instruction `foobarqux.666`, it would work exactly the same. Having said that, may one day
> use the `.yy` bit size information to fall back to instructions implementing larger-bitsize
> versions of the same action, but for now it's not important.

### Example: Register Allocation Correctness

Probably no instruction has caused more bugs than `div`. It's not particularly complicated as an
instruction, but it violates many assumptions made by Lightbeam's backend - especially that we can
always choose the input and output registers of an instruction. This is complicated further by the
fact that x86 `div` has different trapping semantics than WebAssembly's `rem` instruction, and so
the already-difficult task of correctly handling `div` must be combined with the aforementioned poor
handling of internal control flow in Lightbeam.

A mid-level IR fixes these issues, as LIRC would have all information about dirtied registers, it
would have information about control flow that it can use to handle register allocation, and it can
check if any registers that would be dirtied hold values that need to be reachable after the
instruction is over. This unified representation of values means that our only invariant is "at
every point between emitting instructions, all values that are still live must have a physical
location associated with them". This invariant is far simpler than the complex invariants needed to
be maintained throught every implementation of every instruction in the current Lightbeam backend,
and furthermore it is trivial to write debug assertions that check that this invariant is upheld.

## Alternatives

### Add I/O Metadata to `dynasm` Instructions

This was my original plan - to have a library with an identical or similar interface to `dynasm`,
but that allowed you to query which locations were dirtied, where parameter values need to be, and
so forth. However, in order to do this correctly, you need some way to represent what different
output values _mean_. There's no point saying that `div` has its outputs in `eax` and `edx` unless
you know that `eax` is the quotient and `edx` is the remainder, since for the Wasm instruction
`i32.div` you'll need to use `eax`, and for `edx` the remainder. However, the main problem with this
design is that it makes absolutely no attempt to address any of the stated issues around control
flow.

Unfortunately, we want to have perfectly reproducible and safe behaviour on every platform, which
means that it's unavoidable that some Wasm instructions will be implemented using control flow
internally. The only way to fix this is to have an IR that is at approximately the same level of
abstraction as the machine code itself.

This system would also still require the entire backend being reimplemented for each platform, as it
has no abstraction for register allocation or control flow.

In summary, this system only helps our debuggability in straight-line code (which is the least
problematic kind of code to debug in the first place), does nothing to help with retargetablility,
and would take implementation cost not too far from being on par with the proposed system.

### Use a GCC-like system

GCC has a similar subsystem to the one presented here, where you define a "matcher" with an IR AST,
and map it to a specific instruction. This is essentially where the inspiration for this system came
from. Unfortunately, although we can use the general idea of declaratively defining target machine
instructions in terms of the IR AST that they implement, as far as I can tell pretty much all the
details of GCC's implementation do not map well to the [constraints][constraints] of our system.

This is where I start to get out of my depth somewhat, since GCC's implementation details are a
little beyond my knowledge. I'll do my best to explain what I understand, as I understand it. GCC's
equivalent of our Low IR is RTL, a very low-level IR that writes out accesses to registers,
extraction of bits, and so forth. After all the optimisation passes have been done on the higher-
level IR, it eventually gets compiled to RTL. This RTL is then passed into a long series of
patterns, which map the RTL's AST to a specific instruction. This long series of patterns is
referred to as the "machine description", which is roughly equivalent to AsmQuery - I've actually
borrowed this terminology somewhat in AsmQuery, where each target architecture has a "machine
definition". This machine description essentially says "to implement this AST, you can use this
instruction", and then the compiler, when encountering that AST, will convert it to the given
instruction.

However, there are important differences, and the most important of all is that the system in GCC is
pass-based. Everything in GCC is running on a single, large AST, which gets slowly reduced, expanded
and so forth by a series of complex, hand-written patterns. Anything that uses a tree-based AST of
unbounded size or runs multiple optimisation passes is immediately out. While we can still remain
linear-time with a pass-based system, it becomes far, far harder to ensure that complexity. See the
[constraints][constraints] article for details on ensuring complexity. GCC is also designed with a
kind of whole-program knowledge. It _knows_ that we need the carry flag later on, and so it knows to
emit the `add` instruction rather than `lea`, which doesn't affect flags, but we don't have that
luxury as even if we relax our constraint to remain streaming, we cannot read an arbitrarily large
amount of program at once for every instruction.

Another important difference is that GCC can get away with writing many hand-written optimisations
because they have a large team, and because the pass-based nature of their compilation model makes
it simple to add new optimisations after the fact. We cannot do this, we need a system that gives us
the best optimisation for the least code written, and we can bear the cost of our optimisation being
imperfect since because a streaming or linear-time compiler will always have incomplete information
anyway, we don't need to build our system around fully optimising the code, only optimising it well
enough given our knowledge at each point. Our optimisations can only ever be circumstantial anyway,
we cannot simply have a input pattern->output pattern map. The proposed system is essentially a way
to give the computer enough information to do this optimisation for us.

[asmquery]: ./asmquery.md
[backend]: ../design/backend.md
[constraints]: ../design/constraints.md
[lirc]: ./lirc.md
[low-ir]: ./low-ir.md
