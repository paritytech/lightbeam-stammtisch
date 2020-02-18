# Low IR

Low IR is an intermediate representation, or more correctly a family of intermediate
representations, that can be thought of as either a low-level IR like LLVM's Machine IR or as a
kind of "behavioural query language" for [AsmQuery][asmquery].

There are some constraints that Low IR has. For example, it _must_ be an infinite register machine,
with virtual registers, and it must be in SSA form. These constraints are imposed by AsmQuery. In
order for [LIRC][lirc] to work, it must also be possible to define actions with certain specific
behaviours, which are outlined in the article for LIRC. Otherwise, though, each target can define
as many archictecture-specific actions as it likes, although for reasons that I'll get into it's
beneficial to avoid creating new actions wherever possible, and instead trying to use combinations
of more-basic actions. Because of the fact that, wherever possible, we want to use these basic
actions that can be easily shared between archictectures, it's easier to think of Low IR as a
single language instead of a language family, since we want to keep the differences in Low IR
between archictectures as minimal as possible.

[asmquery]: ./asmquery.md
[lirc]: ./lirc.md
