# Low IR

Low IR is an intermediate representation, or more correctly a family of intermediate
representations, that can be thought of as either a low-level IR like LLVM's Machine IR or as a
kind of "behavioural query language" for [AsmQuery][asmquery].

There are some constraints that Low IR has. For example, it _must_ be an infinite register machine,
with virtual registers, and it must be in SSA form. These constraints are imposed by AsmQuery. In
order for [LIRC][lirc] to work, it must also be possible to define actions with certain specific
behaviours, which are outlined in the article for LIRC. This is all very abstract, however, and
since the point of Low IR is to define atomic actions that the target ISA's instructions can be
defined in terms of, for the most part Low IR can be thought of as a single language instead of a
language family.

For an example of how Low IR works, here's an example of some code in Low IR that loads a value
from memory and then multiplies it with some other value (in strawman syntax).

```
%fooaddr = LOWIR::add.64 %stackptr, 10
%foo     = LOWIR::load.32 %fooaddr
%bar     = ;; .. snip ..

%out     = LOWIR::mul.32 %bar, %foo
```

You can see that it's a RISC ISA, where you cannot combine memory accesses and other instructions.
LIRC will pass this sequence of instructions to AsmQuery, which will return a sequence of
instructions that implements the required behaviour (this process is incremental, and the whole
sequence is not passed at once - this is described in [the article for LIRC][lirc] but for now we
can simplify). AsmQuery will have Low IR defined for instructions in the target ISA like so (in
pseudocode):

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

There's no "return" statement, as it's possible for instructions to leave many values in many
places. I didn't write it out here, but `mul` also sets flags based on its inputs, and so in
addition to `%out = ...` you can have `%is_zero = LOWIR::is_zero.32 %out` for instructions that set
bits based on whether their output is zero. The only thing that matters to LIRC is that if it wants
a value to be preserved between instructions, it needs to have a defined location. That can either
be a location defined in terms on the instruction's arguments (for example, for `mul r32, ..` we
can say that the output of the `mul` is written to the same location as the first argument), or as
a specific location, as is the case with `%is_zero` (where we would define `location(%is_zero)` to
be the `ZF` bit of the `FLAGS` register). LIRC only cares that the instruction definition defines
physical locations for all values that are still live at the end of the sequence of Low IR
instructions.

You'll notice that x86's memory operations are written out explicitly. This is deliberate. It means
that we only have to deal with registers, and all uses of registers are explicit. It also means that
we automatically get use of instructions that can do many things (like x86's instructions that
implicitly access memory) whether or not we explicitly asked for them. For example, consider a
series of Wasm instructions doing the same thing:

```lisp
(i32.mul
  (get_local $bar)
  (load (i32.add (get_local $foo) (i32.const 10))))
```

This would still compile to using the x86 implicit-memory-accessing instructions, because LIRC and
AsmQuery just see a stream of Low IR actions and cannot distinguish between 5 actions generated
from 5 Wasm instructions and 5 actions generated from a single Wasm instruction. Sure, this could
just be a small optimisation, but it's a small optimisation that we get for free, and we can
basically guarantee perfect instruction selection with no extra effort on our part. We didn't have
to write a pattern to recognise the Wasm idiom of doing an add, followed by a load, followed by an
arithmetic instruction - it's all defined declaratively in AsmQuery.

TODO

[asmquery]: ./asmquery.md
[lirc]: ./lirc.md
