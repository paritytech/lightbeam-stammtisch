# New Codegen Subsystem

This folder contains design documents for a proposed new codegen subsystem for Lightbeam, designed
to improve separation of concerns, improve the cleanliness and conceptual simplicity of Lightbeam,
and reduce the amount of code that needs to be written to allow Lightbeam to compile for a new
architecture. This subsystem has three major components:

- [Low IR][low-ir]: an extensible, low-level, untyped IR. It's more correct to refer to this as a
  "family" of IRs, as the only thing that must be shared is a small set of basic actions that
  LIRC (see below) needs to implement register allocation, maintaining the stack and so forth. In
  theory, every architecture can have its own version of "Low IR", but in practice these IRs will be
  very similar between architectures, the only differences being for architecture-specific
  instructions that need to be specifically requested, such as Intel extensions implementing
  cryptographic algorithms. The similarities between the IRs is why I refer to it as a single IR
  ("Low IR"), but the only thing that matters is that the form of Low IR used to write
  implementations of the Wasm instructions is the same one used to define the behaviour of the
  machine ISA's instructions.
- [AsmQuery][asmquery]: a "behavioural query system", which allows a compiler to request an
  instruction that implements a certain behaviour instead of asking for the encoding of that
  specific instruction. The developer defines instructions in the target architecture in terms of a
  RISC IR (the "Low IR" for that specific architecture), along with information about constraints
  such as specific registers or sets of registers that must be used. Then, a series of Low IR
  instructions
- [LIRC][lirc] (the "Low IR Compiler"): a simple, architecture-independent register allocator and
  glue layer between the Low IR implementations of Wasm instructions and the AsmQuery behavioural
  query engine. While it can be referred to as "the Low IR Compiler", the only thing that matters
  is that the IR that it's compiling and the IR that the query backend supports is the same, and it
  does not compile a _specific_ IR but rather any IR that:
  1. Is supported by the query engine - i.e. if you have a query engine that defines the x86
     instructions in terms of the "x86 flavour" of Low IR, then LIRC with that query engine will be
     able to compile the "x86 flavour" of Low IR.
  2. Has a small number of generic actions that LIRC needs in order to do its job - this means an
     action that can transfer a value from one location to another (this action would compile down
     to `mov` on x86, for example), an action that performs control flow of various kinds, an
     action to store or load to memory, an action for adding or subtracting (to push and pop from
     the stack), and so forth. It should be noted that it's the _IR being compiled_ that needs
     actions that do these jobs, LIRC doesn't care whether the target architecture can actually
     perform these actions as long as it has a way to _create a query_ for these actions, although
     of course instruction selection will fail at runtime if the query engine returns nothing when
     queried for this action.

## Issues with the current system

I've front-loaded this document with my proposed solution, but I want to go over the actual problem
that needs solving in the first place. The current codegen system, while functional, is brittle and
hard to work on for anyone not deeply familiar with the invariants that need to be upheld, and it's
almost impossible to correctly document all of these invariants (see the design document for the
[backend][backend]). Here are some of the specific issues that I see with the current codegen
system, and what qualities that I would consider important in a project that intends to solve these
issues.

### Issue 1: Reliability/Auditability

The first reason for creating these new components is that Lightbeam's current instruction
selection and register allocation subsystems are deeply tied together, and very tied to the one
specific architecture that we currently support (x86_64). This means that we must have all of this
work done in one, large file, with all the invariants of each having to be maintained manually. The
register allocator works using reference counting, and obviously when control flow merges, the
reference count from each source branch must match for each register. For the most part we have an
abstract representation of control flow, and so it's trivial to ensure that we're correctly
maintaining these invariants. However, when the implementation of an instruction needs control flow
internally, then we have an issue. Since our instruction selection is just emitting bytes without
any abstract representation, we must manually ensure that if any of those bytes are doing control
flow, that the register allocator's internal state at each point is correct. For example, if we
want to emit a fork:

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

We must make sure that if we allocate any registers in `then_branch`, that those registers _must_
be freed before `jmp .cont`, and likewise for `.else_branch`. This is a very easy source of bugs.
So we'd like the register allocator to have insight into _all_ control flow, both between Wasm
instructions and within the implementations of those instructions. This means that we want to write
the implementations of Wasm instructions in kind of IR which is low-level enough to be able to do
everything that the machine code can do, but leaves the precise locations of values abstract, so
that it can be handled at a higher level. This IR can also define control flow abstractly - i.e.
jumping to labels instead of jumping to bytes. Then the register allocator has information about
when control flow is happening, and so we only need to make sure that where the register allocator
handles control flow (which should be in precisely one place) it maintains all the important
invariants, as opposed to having to ensure this in many, many, many places.

### Issue 2: Retargetablility

Right now, there is essentially no instruction selection algorithm. Instruction selection is done
by manually writing out x86 assembly language, with no abstraction, and this is deeply tied to
other parts of the code such as relocations, register allocation and so forth. This means that
implementing a new target for Lightbeam would involve reimplementing huge amounts of code.
`backend.rs` has over 5000 significant lines of code, and pretty much all of it would have to be
rewritten in order to target, for example, ARM. This is over half the SLOC of the whole of
Lightbeam, and even more if you only count the parts of Lightbeam that are actually used in
Wasmtime (the rest is devoted to a simple runtime, used as a test harness before Wasmtime was
integrated). This is not a reasonable amount of code to rewrite for every backend, not only by pure
volume but also because close to 100% of this code must be written very carefully to ensure that no
miscompilations occur, and then if a bug is found in one backend the other backends must be
manually checked to ensure that they can't hit that bug too. We need some way of abstracting out
register allocation, calling convention handling and instruction selection, as the vast majority of
instructions are very simple and so a lot of code can be shared.

However, there is a catch. We can't simply have some low-level IR and then compile that same IR to
all kinds of different environments. For example, if we wanted to load a value from memory, do some
operation on it, and then store that value back to memory, on ARM we'd want to calculate a memory
address once, save it to a register, and then reuse that memory address for both the load and
store. On x86, however, we'd want to recalculate that memory address twice, since we can do it as
part of other instructions and so it saves us the instruction. This is just an example, of course.
The point is that there's always going to be _some_ reason to have different IR for each
architecture, even if it's only for a single instruction. So, although we want the IR to be generic
so much code can be shared between architectures, we also want to allow architecture-specific code
to be written if and when we need to.

### Issue 3: No Framework for Optimisation

Lightbeam, as it currently exists, has no true optimisation subsystem, just lots of optimisations
that are built into the architecture or the implementations of the instructions themselves. There's
no single place that new optimisations can be added, and each instruction must be optimised
independently and manually. Const-folding, for example, has to be implemented separately for both
32-bit and 64-bit versions of all arithmetic instructions, and there's no framework around const
folding or other optimisations that would allow us to, for example, enable and disable instructions
to find bugs. We should have some way to separate out optimisation from codegen, so that parts of
the code that shouldn't care about optimisation don't have to be cluttered with code that
implements optimisations, and so that the effects of optimisations, and when and where they occur,
is more explicit. This would allow us to realise where code is and isn't being optimised, further
optimisations that we could implement, and isolate places where optimisations are being executed
incorrectly.

## Proposed Solution

I propose a system in 3 parts, as outlined in the introduction. I dive deeper into each component
in their respective articles, but as a high-level overview to explain what I believe are the
benefits to this system, I'll elaborate on a few examples that showcase what I believe to be the
relative elegance of this approach in the context of Lightbeam, with explanations of how the system
will handle these examples.

> *NOTE*: I will exclusively use _instruction_ to refer to a specific opcode in the target ISA
> (x86, ARM, etc), and I will use _action_ to refer to a single operation in Low IR, this will look
> like `LOWIR::xxx.yy`, where `xxx` is the opcode of the action and `yy` is the size in bits. For
> example, 64-bit add is `LOWIR::add.64`, and 32-bit add is `LOWIR::add.32`. This is just shorthand
> pseudocode for now, I elaborate more on how I imagine this will actually look in the codebase in
> the [Low IR article][low-ir]. The distinction between _action_ and _instruction_ is very
> important, and so I will go out of my way to ensure that it is always clear which I am talking
> about.

### Example: Optimising Memory Accesses

For an example of how Low IR works, here's an example of some code in Low IR that loads a value
from memory and then multiplies it with some other value (in strawman syntax).

```
%fooaddr = LOWIR::add.64 %stackptr, 10
%foo     = LOWIR::load.32 %fooaddr
%bar     = ;; .. snip ..

%out     = LOWIR::mul.32 %bar, %foo
```

You can see that the memory address calculation and loads are explicitly written out here. LIRC
will pass this sequence of actions to AsmQuery, which will return a sequence of instructions
that implements the required behaviour. AsmQuery will have Low IR defined for instructions in the
target ISA like so (in pseudocode):

> *NOTE*: I will often describe LIRC passing a "sequence of actions" and receiving a "sequence of
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
is no single instruction that does the load, the `mul`, _and_ this other action, then you'll need
to first emit an instruction that loads the value to a physical location, _then_ a `mul` with that
physical location, _then_ the other action. Because this information is declaratively defined,
there doesn't need to be any special-casing here, to compile a sequence of actions to a single
instruction, we only need to ensure that all values that are still reachable after that sequence
have physical locations tied to them.

There's no "return" statement, as it's possible for instructions to leave many values in many
places. I didn't write it out in these example instruction definitions, but `mul` also sets flags
based on its inputs, and so in addition to `%out = ...` you can have `%is_zero = LOWIR::is_zero.32
%out` for instructions that set bits based on whether their output is zero. The only thing that
matters to LIRC is that if it wants a value to be preserved between instructions, it needs to have
a defined location. That can either be a location defined in terms on the instruction's arguments
(for example, for `mul r32, ..` we can say that the output of the `mul` is written to the same
location as the first argument), or as a specific location, as is the case with `%is_zero` (where
we would define `location(%is_zero)` to be the `ZF` bit of the `FLAGS` register). LIRC only cares
that the instruction definition defines physical locations for all values that are still live at
the end of the sequence of Low IR instructions.

> *NOTE*: I cannot stress enough that these `LOWIR::xxx.yy` instructions are just an arbitrary
> string. AsmQuery only cares that it was asked for an instruction that implements `add.32`, pass
> the result to `load.32`, pass the result to `mul.32`. If you had some wild instruction
> `foobarqux.666`, it would work exactly the same. Having said that, may one day use the `.yy` bit
> size information to fall back to instructions implementing larger-bitsize versions of the same
> action, but for now it's not important.

### Example: Register Allocation Correctness

The original reason for adding a layer of abstraction was to make debugging easier, as right now
most bugs must be debugged by simply looking through the backend code and the compiled output and
trying to divine what (if anything) is wrong. For an example of something which is easy to get
wrong when things need to be maintained manually, I'd say that the best two examples are calling
conventions, and the x86 `div` instruction. I'll start with calling conventions.

When we compile a [Microwasm][microwasm] block, we have to maintain its calling convention. The
calling convention is basically: what physical locations hold the arguments to that block, and the
value of the stack pointer at the start of the block. The stack pointer must be set to the correct
value when entering a block, because that block must be able to set the stack pointer to the
correct value when it passes control to the block after it, and so on, and eventually when the
function returns the stack value must be set to the value that it was at the start of the function
or the program will segfault, or even worse, not segfault.

> *NOTE*: This is an issue that most compilers don't have to deal with, I dive deeper into that
> specific topic in the [constraints][constraints] article.

[backend]: ../design/backend.md
[microwasm]: ../design/microwasm.md
[constraints]: ../design/constraints.md
[low-ir]: ./low-ir.md
[asmquery]: ./asmquery.md
[lirc]: ./lirc.md
