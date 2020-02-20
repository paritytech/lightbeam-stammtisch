# Low IR

Low IR is an intermediate representation, or more correctly a family of intermediate
representations, that can be thought of as either a low-level IR like LLVM's Machine IR or as a kind
of "behavioural query language" for [AsmQuery][asmquery].

There are some constraints that Low IR has. For example, it _must_ be an infinite register machine,
with virtual registers, and it must be in SSA form. These constraints are imposed by AsmQuery. In
order for [LIRC][lirc] to work, it must also be possible to define actions with certain specific
behaviours, which are outlined in the article for LIRC. Otherwise, though, each target can define as
many archictecture-specific actions as it likes, although for reasons that I'll get into it's
beneficial to avoid creating new actions wherever possible, and instead trying to use combinations
of more-basic actions. Because of the fact that, wherever possible, we want to use these basic
actions that can be easily shared between archictectures, it's easier to think of Low IR as a single
language instead of a language family, since we want to keep the differences in Low IR between
archictectures as minimal as possible.

## What is Low IR

At its core, like any IR, Low IR is simply a way to describe behaviour. When writing Low IR, we will
specifically avoid trying to dictate the locations of the values it operates on. For example, to do
an addition you may write:

```
%foo = LOWIR::add.32 %bar, %baz
```

Only [LIRC][lirc] has information about the actual location of values. Let's say that when we reach
this action, `%bar` is stored in a register and `%baz` is stored on the stack. Then LIRC will
request the backend for an instruction that implements the following sequence of actions:

```
%bar_addr = LOWIR::add.32 %stackptr, {offset_of_bar}
%bar = LOWIR::load.32 %bar_addr
%foo = LOWIR::add.32 %bar, %baz
```

On an archictecture with instructions that can combine loads with arithmetic, this will compile to a
single instruction. On other archictectures it will compile to 2 or 3 instructions.

The layer at which Low IR is actually written is a map from Microwasm instruction to sequence of Low
IR instructions.

## Complex example: `call_indirect`

> **TODO**: Explain everything that is used in this example in the sections above.

For example, we may implement `call_indirect` something like so: first, popping the top value (which
for `call_indirect` is the index of the function to call) and calling it `%callee`. Let's call the
rest of the items on the Wasm stack `%stack0` (topmost) down to `%stack2` (deepest). For the sake of
argument we'll say that there are only 3 items on the stack other than `%callee`, and that the
target function takes 1 argument and produces 1 result. Using our strawman syntax, we can write out
the implementation like so:

```
#DEFCC @callee_cc(
    CALLER_VMCTX_REGISTER, 
    VMCTX_REGISTER,
    RDX,
)
#DEFCC @callee_ret(
    %callee_ret: rax,
    %stack1: AnyFrom::Reg(CALLEE_SAVED_REGS) ++ AnyFrom::Stack(0..CURRENT_ALLOCATED_STACK_AMOUNT),
    %stack2: AnyFrom::Reg(CALLEE_SAVED_REGS) ++ AnyFrom::Stack(0..CURRENT_ALLOCATED_STACK_AMOUNT),
)

  %num_table_items_addr = LOWIR::add.64 %vmctx, {vmtable_defintion_current_elements}
  %num_table_items = LOWIR::load.64 %num_table_items_addr

  %callee_diff = LOWIR::sub.32 %num_table_items, %callee
  %callee_gt_num = LOWIR::sub_underflow.32 %num_table_items, %callee
  %callee_e_num = LOWIR::eq_zero.32 %callee_diff
  %callee_oob = LOWIR::or.1 %callee_gt_num, %callee_e_num
  LOWIR::br %callee_oob, .trap_oob

  %callee_offset = LOWIR::mul.32 %callee, {sizeof(AnyFunc)}
  %function_arr_ptr = LOWIR::add.64 %vmctx, {vmtable_definition_base}

  %function_sig_addr = LOWIR::add.64 %vmctx, {vmctx_vmshared_signature_id(TYPE_ID)}
  %function_sig = LOWIR::load.64 %function_sig_addr

  %callee_sig_ptr_0 = LOWIR::add.64 %function_arr_ptr, %callee_offset
  %callee_sig_ptr = LOWIR::add.64 %callee_sig_ptr_0, {vmcaller_checked_anyfunc_type_index}
  %callee_sig = LOWIR::load.32 %callee_sig_ptr

  %sig_diff = LOWIR::sub.32 %callee_sig, %function_sig
  %sig_eq = LOWIR::eq_zero.32 %sig_diff
  %sig_ne = LOWIR::not.1 %sig_eq
  LOWIR::br_if %sig_ne, .trap_badsig

  %callee_body_ptr = // TODO

#PASSCC @callee_cc(%vmctx, %callee_vmctx, %stack0)
#PASSCC @callee_ret(undefined, %stack1, %stack2)

  LOWIR::br_link %callee_body_ptr
  LOWIR::br .rest
.trap_badsig:
  LOWIR::trap
.trap_oob:
  LOWIR::trap
.rest @callee_ret:
```

## Implementation

This is where we move away from the conceptual level and into the nuts and bolts. I feel like it's
important that we keep the conceptual model clean and have the implementation be where we do the
optimisation, as this gives us the maximum amount that we can rely on when writing our code.

### Liveness

I've purposefully left liveness out of the explanation thus far, as conceptually it could be added
as a processing pass. Of course, we don't have the opportunity to see upcoming code, and so we must
have some way to generate liveness on the fly. What I believe is the simplest way, and the way that
gives LIRC the most to work with, is to simply have each value be single-use by default (i.e. Low IR
would use [linear types][linear-types]). We can even maintain the invariant that a value _must_ be
used, which would possibly catch bugs. If we need to use a value more than once, we would
_explicitly_ increase its reference count.

```
%foo_addr = // ...

// This increases the reference count by 1
#CLONE %foo_addr 1

%a = LOWIR::load.32 %foo_addr
LOWIR::store.32 %foo_addr
```

This means that LIRC knows that it cannot overwrite `%foo_addr` until after `store`, but that once
`store` has been emitted it can discard `%foo_addr`.

### Rust

So far I've been using an LLVM IR-inspired strawman syntax, but I intend for Low IR to be just a
Rust eDSL, which would mean that we can (for example) extract out common subsequences as Rust
functions and do reuse that way.

```rust
/// This trait is implemented for Low IR action enums that have a variant for `MOV`, a variant
/// for `LOAD`, a variant for `STORE`, etc. This is then used by LIRC internally.
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

/// _Does not_ implement `Copy`/`Clone`, as it is intended to be consumed by `fn action`
/// in `Context`. Emits `#[must_use]`, which makes the Rust compiler emit a warning if it
/// is not used. The combination of these two makes it easier for the Rust compiler to tell
/// us if we've mismanaged the lifetimes of values.
#[must_use]
struct Value {
    id: u64,
}

struct CallConv {
    id: u64,
}

struct ActionDef<A> {
    action: A,
    output: Value,
    inputs: Vec<Value>,
}

struct AnyLoc {
    regs: RegClass,
    stack: Range<usize>,
}

enum StackLoc {
    /// Relative to the value of `rsp` now - for calling functions this is used for stack
    /// arguments. Currently in Lighteam we always allocate enough space for all stack
    /// arguments, but the only necessary constraint is that the values are N bytes from
    /// the top of the stack.
    ///
    /// TODO: How to handle stack alignment here?
    FromTop(usize),
    /// Relative to the value of `rsp` when the function was entered
    FromBottom(usize),
}

enum Location {
    Reg(Register),
    Stack(StackLoc),
}

enum CCLoc {
    Any(AnyLoc),
    Specific(Location),
}

type Immediate = i128;

trait Context<Action> {
    fn imm(&mut self, val: Immediate) -> Value;
    fn unknown(&mut self) -> Value;
    fn vmctx(&self) -> Value;
    fn clone<const COUNT: usize>(&self, value: Value) -> [Value; COUNT];
    fn action(&mut self, action: Action, args: &[Value]) -> Value;
    fn action_discard(&mut self, action: Action, args: &[Value]);
    fn new_cc(&mut self, args: &[Option<CCLoc>]) -> CallConv;
    fn pass_cc<const COUNT: usize>(&mut self, cc: CallConv, args: [Option<Value>; COUNT]) -> [Value; COUNT];
    // TODO: Handle relocations better (probably with arguments to `unknown` which tag the
    //       value with metadata about how this value is to be used, similar to Wasmtime's
    //       reloc system.
    fn define_label(&mut self, value: Value);
    
    fn pop(&mut self) -> Value;
    fn push(&mut self, value: Value);
}

type Bits = u8;

enum LowIR {
    Mov,
    Add(Bits),
    Load(Bits),
    Sub(Bits),
    SubUnderflow(Bits),
    Or(Bits),
    Not(Bits),
    Br,
    Mul(Bits),
    EqZ(Bits),
}

trait Architecture {
    type Action;

    fn call_indirect<C: Context<Self::Action>>(&mut self, ctx: &mut C) {
        let callee_cc = ctx.new_cc(&[
            VMCTX_REGISTER.into(),
            CALLER_VMCTX_REGISTER.into(),
            RDX.into(),
        ]);
        // TODO: This doesn't handle arbitrary numbers of items on the stack
        let callee_ret = ctx.new_cc(&[
            RAX.into(),
            AnyLoc::reg(CALLEE_SAVED_REGS),
            AnyLoc::reg(CALLEE_SAVED_REGS),
        ]);

        let vmctx = ctx.vmctx();

        let vmtable_defintion_current_elements = ctx.imm(vmtable_defintion_current_elements);
        let num_table_items_addr =
            ctx.action(LowIR::Add(64), &[vmctx, vmtable_defintion_current_elements]);
        let num_table_items = ctx.action(LowIR::Load(64), &[num_table_items_addr]);

        let callee_diff = ctx.action(LowIR::Sub(32), &[num_table_items, callee]);
        let callee_gt_num = ctx.action(LowIR::SubUnderflow(32), &[num_table_items, callee]);
        let callee_e_num = ctx.action(LowIR::EqZ(32), &[callee_diff]);
        let callee_oob = ctx.action(LowIR::Or(1), &[callee_gt_num, callee_e_num]);
        let trap_oob = ctx.unknown();
        ctx.action_discard(LowIR::Br, &[callee_oob, trap_oob]);

        let anyfunc_size = ctx.imm(mem::sizeof(AnyFunc));
        let callee_offset = ctx.action(anyfunc_size);
        let vmtable_definition_base = ctx.imm(vmtable_definition_base);
        let function_arr_ptr = ctx.action(LowIR::Add(64), &[vmctx, vmtable_definition_base]);

        let vmctx_vmshared_signature_id(TYPE_ID) = ctx.imm(vmctx_vmshared_signature_id(TYPE_ID));
        let function_sig_addr = ctx.action(
            LowIR::Add(64),
            &[vmctx, vmctx_vmshared_signature_id(TYPE_ID)],
        );
        let function_sig = ctx.action(LowIR::Load(64), &[function_sig_addr]);

        let callee_sig_ptr_0 = ctx.action(LowIR::Add(64), &[function_arr_ptr, callee_offset]);
        let vmcaller_checked_anyfunc_type_index = ctx.imm(vmcaller_checked_anyfunc_type_index);
        let callee_sig_ptr = ctx.action(
            LowIR::Add(64),
            &[callee_sig_ptr_0, vmcaller_checked_anyfunc_type_index],
        );
        let callee_sig = ctx.action(LowIR::Load(32), &[callee_sig_ptr]);

        let sig_diff = ctx.action(LowIR::Sub(32), &[callee_sig, function_sig]);
        let sig_eq = ctx.action(LowIR::EqZ(32), &[sig_diff]);
        let sig_ne = ctx.action(LowIR::Not(1), &[sig_eq]);
        let trap_badsig = ctx.unknown();
        ctx.action_discard(LowIR::BrIf, &[sig_ne, trap_badsig]);
        
        let rest = ctx.unknown();
        // TODO: Not workable for arbitrary numbers of items on stack, work out some better way of
        //       handling this.
        let (stack0, stack1, stack2) = (ctx.pop(), ctx.pop(), ctx.pop());
        let _ = ctx.pass_cc(callee_cc, [Some(vmctx), Some(callee_vmctx), Some(stack0)]);
        let [result, stack1, stack2] = ctx.pass_cc(callee_ret, [None, Some(stack1), Some(stack2)]);

        ctx.action_discard(LowIR::BrLink, &[callee_body_ptr]);
        ctx.action_discard(LowIR::Br, &[rest]);

        ctx.define_label(trap_oob);
        // TODO: somehow allow access to PC so we can use it for trap metadata
        ctx.action_discard(LowIR::Trap, &[]);
        ctx.define_label(trap_badsig);
        ctx.action_discard(LowIR::Trap, &[]);

        ctx.define_label(rest);
        ctx.push(stack2);
        ctx.push(stack1);
        ctx.push(result);
    }
}
```

[asmquery]: ./asmquery.md
[linear-types]: https://en.wikipedia.org/wiki/Substructural_type_system#Linear_type_systems
[lirc]: ./lirc.md
