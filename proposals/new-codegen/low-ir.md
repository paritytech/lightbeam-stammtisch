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

## Concepts

### Calling conventions

Calling conventions, in Low IR, are our abstraction over any time that we need values in a specific
location for reasons other than using them as arguments for instructions. For our purposes this
means control flow - calling functions, returning from functions, and intra-function control flow
(jumping between blocks). A calling convention has two components: the stack depth (i.e., the value
that `rsp` must take), and the location of arguments. Both of these can either be concrete or
generic. Concrete means that there is a _specific_ value that the stack depth must be, or a
_specific_ location that an argument must take. Generic means that there is a class of values that
these could take, and LIRC can pick the one that is most appropriate. For example, when calling a
function, if the stack depth is generic then LIRC can make the choice to either free up extra stack
space or reuse already-allocated stack space for the stack arguments to that function. If we're
jumping to a new block and its arguments are generic, we can choose to keep the values in the
location that they're already in, if appropriate.

When a calling convention with generic parts is called, concrete locations are chosen for each of
those generic parts. This calling convention can then later be "applied" once the label that this
calling convention was for has been reached, and now values will be in scope whose locations
correspond to the chosen concrete locations. If concrete locations were never chosen for one or more
of the values, attempting to apply that calling convention will fail.

### Labels

A label, in Low IR, is simply an immediate with an unknown value. LIRC handles the actual
relocations. Once the immediate's value has been determined (specifically, by reaching the
appropriate label), LIRC can rewrite the immediates to have the correct value. More information
about this in the [LIRC article][lirc], the section on relocations.

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

### Implementation template

So far I've been using an LLVM IR-inspired strawman syntax, but I intend for Low IR to be just a
Rust eDSL, which would mean that we can (for example) extract out common subsequences as Rust
functions and do reuse that way. This might also make it a bit clearer exactly how this works on a
low level. All of the specifics of the code here are up for debate, of course, but the template is
approximately how I intend for this to look, with extensive comments describing the meaning and
reasoning behind my decisions.

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

/// A handle to a value, the specific information about this value is handled by LIRC, as far as
/// the backend is concerned the only thing that matters is unique identity. In LIRC these are
/// reference-counted.
///
/// _Does not_ implement `Copy`/`Clone`, as it is intended to be consumed by `fn action`
/// in `Context`. Emits `#[must_use]`, which makes the Rust compiler emit a warning if it
/// is not used. The combination of these two makes it easier for the Rust compiler to tell
/// us if we've mismanaged the lifetimes of values.
#[must_use]
struct Value {
    id: u64,
}

/// A handle to a calling convention. The specific information about this calling convention is
/// handled by LIRC, as far as the backend is concerned the only thing that matters is unique
/// identity.
// TODO: We should probably have a `fn clone`-like reference counting system like for values, as
//       this would allow LIRC to delete calling conventions that we know we will no longer use.
struct CallConv {
    id: u64,
}

struct AnyLoc {
    regs: RegClass,
    stack: Range<usize>,
}

enum StackLoc {
    /// Relative to the value of `rsp` now - for calling functions this is used for stack
    /// arguments. Currently in Lighteam we always allocate enough space for all stack
    /// arguments, but the only necessary constraint is that the values are N bytes from
    /// the top of the stack (and that the stack is correctly aligned), so this gives us
    /// maximum freedom.
    FromTop(usize),
    /// Relative to the value of `rsp` when the function was entered
    FromBottom(usize),
}

/// A specific, specified location for a value.
enum Location {
    Reg(Register),
    Stack(StackLoc),
}

/// A location for an argument in a calling convention. This can either be a generic location or
/// a specific location.
enum CCLoc {
    Any(AnyLoc),
    Specific(Location),
}

/// Immediates don't get tagged with their size, they are valid to be used as values of any
/// size so long as the actual value of that immediate doesn't exceed the desired bit size.
/// For example, you can use an immediate as a u8 so long as it is less than 256, or as a u1
/// (a boolean value) so long as it is 0 or 1.
type Immediate = u64;

type Alignment = u8;

enum CCStackDepth {
    /// The precise stack depth doesn't matter, as long as it is aligned correctly.
    Any(Alignment),
    /// The stack depth must be a precise value, relative to the value of `rsp` when the function
    /// was entered.
    Specific(usize),
}

/// A Low IR compilation "context". This is the interface the programmer uses to communicate with
/// LIRC. Unlike Microwasm, we simply use method calls instead of having an actual "Operator"
/// enum, as this allows us to easily give names to values.
trait Context<Action> {
    /// Create a new immediate value.
    fn imm(&mut self, val: Immediate) -> Value;
    /// Create a new immediate with no value. This can be used anywhere an immediate is normally
    /// used, but must be defined later.
    fn unknown(&mut self) -> Value;
    /// Clone the given value, increasing its refcount and returning the clones. As `Value` doesn't
    /// implement `Copy` or `Clone`, this means that you can never incorrectly use the value after
    /// it has already been freed - Rust's type system ensures that.
    fn clone<const COUNT: usize>(&self, value: Value) -> [Value; COUNT];
    /// Do a single Low IR action. This _does not_ immediately produce an opcode, but pushes the
    /// action and its params into a buffer. This buffer has a small, bounded maximum size, defined
    /// by the maxmimum number of actions in a single instruction definition in the associated
    /// AsmQuery machine specification.
    fn action(&mut self, action: Action, args: &[Value]) -> Value;
    /// Do a single Low IR action, without reading the result. This is kinda hacky right now, and
    /// there's probably a cleaner way of doing this, but this is essentially for actions that
    /// don't produce a meaningful value, like "store" or control flow opcodes. We need to have a
    /// separate `action_discard` method, as otherwise LIRC would try to look for an opcode that
    /// produces the "output" of these void-returning opcodes into a real location like a
    /// register, and obviously this will always fail.
    fn action_discard(&mut self, action: Action, args: &[Value]);
    /// Initialize a new calling convention. The locations for the CCLoc can either be specific
    /// locations or "generic" locations that only have to meet certain bounds.
    fn new_cc(&mut self, stack_depth: CCStackDepth, args: &[CCLoc]) -> CallConv;
    /// Pass values to the given calling convention, returning new values representing the
    /// locations those arguments were put into. If the value is `None` for an argument,
    /// this value is undefined, and accessing it is undefined behaviour, so it should only be
    /// used with opcods like `br_link` where you can be sure that something external will put a
    /// valid value in that location.
    ///
    /// The values returned from `pass_cc` are not valid until `set_cc` has been called, although
    /// LIRC will error if you try to use any of these values incorrectly (as each value has a
    /// globally-unique ID, there is no possibility of values accidentally overlapping between
    /// contexts).
    fn pass_cc<const COUNT: usize>(
        &mut self,
        cc: CallConv,
        args: [Option<Value>; COUNT]
    ) -> [Value; COUNT];
    // TODO: Handle relocations better (probably with arguments to `unknown` which tag the
    //       value with metadata about how this value is to be used, similar to Wasmtime's
    //       reloc system.
    /// This defines the given immediate value defined with `unknown` at each point that it was
    /// used, with the value being the difference between the PC at each use and the current PC.
    fn define_label(&mut self, value: Value);
    /// This "applies" the calling convention, making the arguments to this CC become in-scope.
    /// This is only valid if this CC has defined values for the stack depth and all of the
    /// argument locations. This can be checked by LIRC.
    fn set_cc(&mut self, cc: CallConv);
}

type Bits = u8;

/// An example of what a Low IR action opcode type might look like.
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

impl LIRCAction for LowIR {
    const MOV = Self::Mov;
    const LOAD32 = Self::Load(32);
    const LOAD64 = Self::Load(64);
    const STORE32 = Self::Store(32);
    const STORE64 = Self::Store(64);
    const ADD32 = Self::Add(32);
    const ADD64 = Self::Add(64);
    // .. etc ..
}

// TODO: The architecture should define the scratch registers and so forth, _not_ the AsmQuery
//       machine spec (as it is dependent on machine calling convention). There are many ways to
//       handle this, all with different trade-offs, and the specifics of how this would work 
/// The trait for specifying implementations of all of the Microwasm opcodes in straight-line
/// code. While there will be some differences in how control flow is handled between
/// architectures, we probably want LIRC to be the main driver for handling all of that.
trait Architecture {
    /// The type used as the Low IR opcode
    type Action;

    fn call_indirect<C: Context<Self::Action>>(&mut self, ctx: &mut C);
    fn add_i32<C: Context<Self::Action>>(&mut self, ctx: &mut C);
    // ...
}

struct X86Backend {
    // ...
}

impl Architecture for X86Backend {
    type Action = LowIR;

    fn call_indirect<C: Context<Self::Action>>(&mut self, ctx: &mut C) {
        let callee_cc = ctx.new_cc(&[
            VMCTX_REGISTER.into(),
            CALLER_VMCTX_REGISTER.into(),
            RDX.into(),
        ]);
        // TODO: This assumes that we have precisely 2 unused items on the stack, and doesn't
        //       handle arbitrary numbers of items on the stack. Handling this correctly and
        //       efficiently is out of scope for this sketch, for now.
        let callee_ret = ctx.new_cc(&[
            RAX.into(),
            VMCTX_REGISTER.into(),
            AnyLoc::reg(CALLEE_SAVED_REGS),
            AnyLoc::reg(CALLEE_SAVED_REGS),
        ]);

        // By `take`ing `vmctx` here, we ensure that the `vmctx` has no references left when
        // we pass it to the external function, because we don't want LIRC to try to save its
        // value.
        let [vmctx0, vmctx1, vmctx2, vmctx3] = ctx.clone::<4>(self.vmctx.take().unwrap());

        let vmtable_defintion_current_elements = ctx.imm(vmtable_defintion_current_elements);
        let num_table_items_addr =
            ctx.action(LowIR::Add(64), &[vmctx0, vmtable_defintion_current_elements]);
        let num_table_items = ctx.action(LowIR::Load(64), &[num_table_items_addr]);

        // Because this is just Rust code, we can easily extract these 4 lines out into a helper
        // implementing `a >= b`
        let callee_diff = ctx.action(LowIR::Sub(32), &[num_table_items, callee]);
        let callee_gt_num = ctx.action(LowIR::SubUnderflow(32), &[num_table_items, callee]);
        let callee_e_num = ctx.action(LowIR::EqZ(32), &[callee_diff]);
        let callee_oob = ctx.action(LowIR::Or(1), &[callee_gt_num, callee_e_num]);

        let trap_oob = ctx.unknown();
        ctx.action_discard(LowIR::Br, &[callee_oob, trap_oob]);

        let anyfunc_size = ctx.imm(mem::sizeof(AnyFunc));
        let callee_offset = ctx.action(anyfunc_size);
        let vmtable_definition_base = ctx.imm(vmtable_definition_base);
        let function_arr_ptr = ctx.action(LowIR::Add(64), &[vmctx1, vmtable_definition_base]);

        let vmctx_vmshared_signature_id(TYPE_ID) = ctx.imm(vmctx_vmshared_signature_id(TYPE_ID));
        let function_sig_addr = ctx.action(
            LowIR::Add(64),
            &[vmctx2, vmctx_vmshared_signature_id(TYPE_ID)],
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
        //       handling this. Probably this will involve some scheme using iterators, and so it
        //       is out of scope for this sketch.
        let (stack0, stack1, stack2) = (self.pop(), self.pop(), self.pop());
        let _ = ctx.pass_cc(callee_cc, [Some(vmctx3), Some(callee_vmctx), Some(stack0)]);
        let [result, vmctx, stack1, stack2] = ctx.pass_cc(
            callee_ret,
            [None, None, Some(stack1), Some(stack2)]
        );

        ctx.action_discard(LowIR::BrLink, &[callee_body_ptr]);
        ctx.action_discard(LowIR::Br, &[rest]);

        ctx.define_label(trap_oob);
        // TODO: somehow allow direct access to PC so we can use it for trap metadata.
        ctx.action_discard(LowIR::Trap, &[]);
        ctx.define_label(trap_badsig);
        ctx.action_discard(LowIR::Trap, &[]);

        ctx.define_label(rest);
        ctx.set_cc(callee_ret);

        self.vmctx = Some(vmctx);

        self.push(stack2);
        self.push(stack1);
        self.push(result);
    }
}
```

The only thing that isn't handled here is how we deal with serialising values that need to be
preserved after the call, but I believe that the important thing is that we have the outline and the
specifics of the API for handling this isn't important enough for it to be included here and can be
designed as we go along, as we build the system itself. If this turns out to be a more difficult
design than it appears and we need to discuss it more fully, I can return to this document then.

[asmquery]: ./asmquery.md
[linear-types]: https://en.wikipedia.org/wiki/Substructural_type_system#Linear_type_systems
[lirc]: ./lirc.md
