# LIRC

LIRC is a machine-independent register allocation subsystem, which compiles some IR that conforms to
certain constraints (outlined below) to machine code, by using [AsmQuery][asmquery] to search for
valid instructions.

## Responsibilities

In [the original pipeline][design-overview], LIRC corresponds roughly to the ["backend
driver"][backend-driver], in that it directly receives Microwasm, handles control flow, and calls
methods on [the backend][low-ir] in order to handle straight-line code. However, LIRC is also in
charge of all responsibilities related to handling value liveness, maintaining the stack and doing
register allocation. This means that the backend now _only_ handles actually writing the low-level
code's functionality, and does not and should not care about the details of where values are
located.

## Low IR

While the input to the compilation done by LIRC is known as [Low IR][low-ir], for the most part the
specifics of this IR are unimportant. Since LIRC must be able to generate some code internally,
however, it must have some constraints on which opcodes are available to it. This would likely be
represented as a trait, looking something like this:

```rust
trait LIRCAction {
    const MOV: Self;
    const LOAD32: Self;
    const LOAD64: Self;
    const STORE32: Self;
    const STORE64: Self;
    const ADD32: Self;
    const ADD64: Self;
    // .. etc ..
}
```

Then when LIRC needed to internally generate a mov, it would do `<Backend::Action as
LIRCAction>::MOV`, where the LIRC type is parameterised with some backend type `Backend`.

## Outline of Implementation

For each Microwasm operation, we handle control flow (see below), and then call into the backend,
passing a mutable reference to the LIRC type as the interface to LIRC - this interface roughly
corresponds to how an assembler works, except it is an in-process eDSL instead of being an external
textual DSL. For an explanation by comparison to a traditional assembler, let's say that in an
assembler you might write:

```
.ascii "Hello, world"
.some_label:
  sub rax, rdx
  jz .some_other_label
```

But if we were building an assembler designed to be embedded in a programming language, we could
have an embedded assembler that was implemented using a trait that looks something like so:

```rust
trait Assembler {
    fn ascii(&mut self, val: &str);
    fn new_label(&mut self) -> Label;
    fn define_label(&mut self, label: Label);
    fn instruction(&mut self, instruction: Instr, args: &[Arg]);
}

fn my_assembly_file(asm: &mut impl Assembler) {
    asm.ascii("Hello, world");
    let some_label = asm.new_label();
    let some_other_label = asm.new_label();
    asm.define_label(some_label);
    asm.instruction(Instr::Sub, &[RAX, RDX]);
    asm.instruction(Instr::Sub, &[RAX, RDX]);
    asm.instruction(Instr::JZ, &[some_other_label]);
}
```

Now for LIRC. I've been using a strawman syntax to refer to Low IR, in order to avoid talking about
the implementation details where possible. The strawman syntax looks something like so:

```
%fooaddr = LOWIR::add.64 %stackptr, 10
%foo     = LOWIR::load.32 %fooaddr
%bar     = ;; .. snip ..

%out     = LOWIR::mul.32 %bar, %foo
```

The actual implementation is intended to be a trait, which the backend can call methods on. The full
trait, along with usage and full doc comments explaining each method, is better outlined in the [Low
IR article][low-ir], but as it currently stands the outline of the trait would look like so:

```rust
trait Context<Action> {
    fn imm(&mut self, val: Immediate) -> Value;
    fn unknown(&mut self) -> Value;

    fn action(&mut self, action: Action, args: &[Value]) -> Value;

    fn new_cc(&mut self, stack_depth: CCStackDepth, args: &[CCLoc]) -> CallConv;
    fn pass_cc<const COUNT: usize>(
        &mut self,
        cc: CallConv,
        args: [Option<Value>; COUNT]
    ) -> [Value; COUNT];
    fn define_label(&mut self, value: Value);

    // .. snip ..
}
```

The core of this is the `action` method, which is a single line of Low IR code. The rest of the
methods are "directives" for LIRC, which again, are further explained in the [article for Low
IR][low-ir].

## Diving Deeper

A full implementation of each component of LIRC is out of scope for this explanatory article, but an
outline of the design of the different responsibilities of LIRC is given below.

### Value System

The value system is essentially a simple map from unique value ID -> location. When a value goes out
of scope it is removed from the map.

[backend-driver]: ../design/backend-driver.md
[design-overview]: ../design/
[low-ir]: ./low-ir.md
