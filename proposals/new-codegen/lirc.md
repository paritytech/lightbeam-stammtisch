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
trait Context {
    type Action;

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

Each "value" in the value system is simply a globally-unique ID, and LIRC maintains a hashmap from
ID to location(s) (precise meaning of "location" given below) and from location to ID. The use of
the former is obvious, but the use of the latter may not be so clear. When we emit an instruction,
we need to put the arguments in a place that satisfies the constraints given by that instruction's
definition. `div`, for example, can only take its parameters in specific registers. We can first
look into the location->value map, see if there's a value already in that location, and then move
_just that value_. We can maintain an invariant that instruction definition parameters can't
overlap, which means that we don't have to implement any complex behaviour to ensure that moving one
value doesn't invalidate another - we can just iterate over each parameter in turn and put it
somewhere valid.

Now, I mention that we move the value already in the given location, but that's not actually true,
because a value can be in multiple locations. For example, we've been referring to the operation to
transfer a value between registers as `mov`, because that's what it's called on most architectures,
but this name is misleading because what this operation actually is is a _copy_. The original value
is still there. Therefore each value can have multiple locations, so long as it always has _at least
one_. Additionally, the list of locations should be a set (i.e., no repeating elements and `O(1)`
removal), as this ensures that we don't shoot ourselves in the foot by invalidating a location for a
value only once, when the value is in that location multiple times - `O(1)` removal also makes it
trivial to invalidate a location in constant time.

```rust
enum ValueLoc {
    Stack { depth: usize },
    Reg(Reg),
    Imm(Immediate),
}

struct Values {
    // TODO: A `NonEmptyHashSet` type would make it easier to ensure that our invariants are upheld.
    value_locations: HashMap<Value, HashSet<ValueLoc>>,
    location_values: HashMap<ValueLoc, Value>,
}

impl Values {
    /// Example of how we can trivially invalidate a location in constant time. Returns a `Value`
    /// if that value would have no more locations left.
    ///
    /// As it returns `None` for both the case that the location is empty and the case that the
    /// location has a value in it, but that value can be accessed elsewhere, LIRC can use this
    /// function any time it needs to use a location, _before_ it actually overwrites that location,
    /// emit a `mov` to a new, free location if it returns `Some`, and _then_ emit the instruction
    /// that would overwrite that location.
    fn invalidate_loc(&mut self, loc: ValueLoc) -> Option<Value> {
        let val = self.location_values.remove(&loc)?;
        let locs = &mut value_locations[&val];

        locs.remove(&loc);
        if locs.is_empty() {
            self.value_locations.remove(val);
            Some(val)
        } else {
            None
        }
    }
}
```

Depending on the exact implementation details, `Reg` might be a concrete type or it might be an
associated type on the machine spec, but it doesn't matter. Like `Value`, it can be treated as an
opaque type. Note that unlike the current backend, there is no separate `CondCode` variant -
condition codes are just (1-bit) registers in this design. Unlike the current design, where calling
conventions are split into "real" and "virtual", where virtual calling conventions can have
arguments in flags or immediates, in this design the idea that a block with many callers should have
its arguments in well-defined, mutable locations is encapsulated by explicit constraints on the
calling conventions of labels. This makes it far easier to be more specific about what you
_actually_ want. Right now, if we want to implement a greedy optimisation to ensure that, say, the
top 5 elements on the Wasm stack should always be in registers but anything deeper is allowed to be
either in a register or on the stack, we must manually implement that in a brittle, manual way. If
we want to allow blocks that are called multiple times to have their arguments in condition codes
(instead of only in registers or the stack), we have to do a total overhaul of a huge amount of the
code, because there's currently no good way to implement that. Another optimisation that would be
close to impossible to implement with the current system but trivial with the new system would be to
"freeze" items that are on the Wasm stack but not accessible due to Wasm's scoping rules, which
would mean that the following code:

```wasm
i32.const 0xabababab
i32.const 0xcdcdcdcd
loop
  ; ... do something ...
end
i32.add
```

Could be const-folded without any special-casing.

There is one more "location" that a value could have, which I will talk about in the "Possible
Extensions" section.

### Instruction Selection

This is the core of LIRC - the instruction selection algorithm. The pseudocode for this is given
below.

```rust
mod asmquery {
    struct ActionWithArgs<Action> {
        output: Value,
        action: Action,
        // The maximum size of this is strictly bound by the `Action` type so we can easily also
        // make this a `StaticVec`.
        args: Vec<Value>,
    }

    trait MachineSpec {
        /// The type used to represent a single Low IR Action 
        type Action;

        /// Maximum number of actions in a single instruction definition
        const MAX_ACTIONS: usize;

        // Would have to be made to allocate somewhere but could trivially be made zero-allocation
        // once GATs are properly introduced.
        // `InstructionDef` type elaborated on in the article for AsmQuery.
        type Instrs: Iterator<Item = InstructionDef>;

        /// Find an instruction that matches the list of actions. This _doesn't_ handle checking to
        /// see if (for example) the instruction assigns the result of any action to a reachable
        /// location. That should be handled by LIRC. This returns instructions that do the supplied
        /// actions, where the flow of values (i.e. which outputs of which actions get used as which
        /// arguments of which other actions).
        fn instrs(&mut self, actions: &[ActionWithArgs<Self::Action>]) -> Self::Instrs;
    }
}

// Like the `Values` type above, but we additionally store refcounts. We use `NonZeroUsize` here,
// as the value should always be removed when the refcount hits zero and so this ensures that we
// always do this correctly.
struct Values {
    value_locations: HashMap<Value, (NonZeroUsize, HashSet<ValueLoc>)>,
    location_values: HashMap<ValueLoc, Value>,
}

struct ActionStack<Action, const COUNT: usize> {
    // Like `Vec`, but stored. on the stack. This is to make it clear that `action_stack`
    // has a strict, small upper bound on size, and so iterating over it for each call to
    // `action` doesn't cause unbounded runtime. `NonEmpty` means that we can use an
    // `Option<ActionStack<..>>` to ensure that if we have any actions in the stack, we always
    // have a valid `InstructionDef` for them, and if we've cleared the action stack, we
    // know that we can never have a "leftover" `InstructionDef` that we forgot to get
    // rid of.
    stack: NonEmptyStaticVec<
        Action,
        COUNT,
    >,
    // This is the last instruction definition that we found, that should be the next
    // instruction to be emitted if a new instruction that can do more actions at once
    // cannot be found.
    instr_def: InstructionDef,
}

impl<A, const C: usize> ActionStack<A, C> {
    fn new(action: A, instr_def: InstructionDef) -> Self {
        Self {
            stack: ,
            instr_def,
        }
    }
}

struct LIRC<Backend, MachineSpec> 
where
    MachineSpec: asmquery::MachineSpec,
{
    values: Values,
    action_stack: Option<
        ActionStack<ActionWithArgs<MachineSpec::Action>, MachineSpec::MAX_ACTIONS>,
    >,
    backend: Backend,
    machine_spec: MachineSpec,
}

impl<Backend, MachineSpec> Context for LIRC<Backend, MachineSpec>
where
    // See the article on Low IR for an explanation of the `Architecture` and `LIRCAction`
    // traits
    Backend: Architecture,
    Backend::Action: LIRCAction,
    MachineSpec: asmquery::MachineSpec<Action = Backend::Action>,
{
    type Action = Backend::Action;

    fn action(&mut self, action: Action, args: &[Value]) -> Result<Value> {
        use std::slice;

        let val_id = self.new_value();

        // We can definitely figure out a way to remove the unconditional allocation in
        // `args` here, but for this sketch we'll just use `to_vec`.
        let action = ActionWithArgs { output: val_id, action, args: args.to_vec() };
        let action = if let Some(mut actions) = self.action_stack.take() {
            // If there are already actions on the stack, that means that we have an `instr_def`
            // to emit if no valid instructions are found.
            match actions.try_push(action) {
                Ok(()) => {
                    let instrs = self.machine_spec.instrs(&actions.stack);
                    if let Some(instr) = instrs.find(|i| self.instr_valid(i)) {
                        // The previously-found `instr_def` is discarded, and we continue with the
                        // new `action` added to the stack and the new instr we just found.
                        self.action_stack = Some(ActionStack::new(actions, instr));
                        return val_id;
                    }

                    // No valid instructions were found, so we must emit the `instr_def` that we
                    // found in the previous call to `action` and start a new action stack.

                    // `emit_instr` described further below, but for now it's just important that
                    // before `emit_instr`, any `output` fields on the action stack point to values
                    // with no assigned locations - `emit_instr` emits the instruction and assigns
                    // a location to these values.
                    self.emit_instr(stack.instr_def);

                    action
                },
                Err(err) => {
                    // This is essentially just a fast path that skips the `instrs` + `find` in the
                    // `Ok` branch - we know that no instructions will be found because we know the
                    // maximum number of actions that any instruction in the whole `MachineSpec`
                    // can contain.

                    self.emit_instr(stack.instr_def);

                    action
                }
            }
        } else {
            // This means we have no actions in the stack at all, and so we should start a new stack
            action
        };

        let new_action_stack = NonEmptyStaticVec::new(action);
        let new_instr = self.machine_spec.instrs(&new_action_stack)
            .find(|i| self.instr_valid(i))
            .ok_or(Error::NoActionsFound)?;

        self.action_stack = Some(ActionStack::new(new_action_stack, new_instr));

        val_id
    }
}
```

[backend-driver]: ../design/backend-driver.md
[design-overview]: ../design/
[low-ir]: ./low-ir.md
