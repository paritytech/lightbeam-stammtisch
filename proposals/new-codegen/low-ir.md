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
for `call_indirect` is the index of the function to call) and calling it `%callee`, then writing out
the access like so:

```
#DEFCC @caller_cc(
    CALLER_VMCTX_REGISTER, 
    VMCTX_REGISTER,
    /* .. rest of the calling convention */
)
#DEFCC @caller_ret(
    %vmctx: VMCTX_REGISTER,
    %caller_ret: rax,
    /* .. rest of the calling convention */
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
  LOWIR::br %sig_ne, .trap_badsig
  %callee_vmctx = // ...

#SAVECURRENTSTATE @current_cc(/* .. anything not passed to @caller_cc .. */)
#PASSCC @caller_cc(%vmctx, %callee_vmctx, /* .. rest of the arguments .. */)
  LOWIR::br_link TRUE, %callee_body_ptr

#SETCURRENTSTATE @current_cc ++ @caller_ret

  // TODO

.trap_badsig:
  LOWIR::trap
.trap_oob:
  LOWIR::trap
.continue:
```

## Liveness

[asmquery]: ./asmquery.md
[lirc]: ./lirc.md
