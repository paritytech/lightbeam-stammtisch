# Roadmap

This is the proposed higher-level future path for Lightbeam development, as well as related projects
and features that either require Lightbeam or are tied to it in some meaningful way.

## 1. Bugfixes

The first thing that needs to be dealt with is this ["stack depth mistracking"][stack-depth] error
that has existed in Lightbeam for a while now. This is, as far as I can tell, the only thing
stopping Lightbeam from being integrated in a preliminary capacity into Substrate. I have reason to
believe that this bug is now fixed, but in order to test it we have to finish integrating Lightbeam
with Wasmtime. This work was done, but because of Lightbeam's ad-hoc approach to some parts of the
compilation pipeline (see [status][status]) it's tough for the Wasmtime team to make changes to
Lightbeam to keep it in sync with the latest changes in Wasmtime, which means that when major
breaking changes happen we end up with a lot of bugfixes needing to be applied which can be non-
trivial to figure out given our distance from the Wasmtime team.

The current status of this refactor is that we have the information on what needs to be changed,
although it's not clear if this information is complete as we still need to make some fairly major
changes to Lightbeam in order to make changes. A big issue is that there is a circular dependency
between `wasmtime-environ` and Lightbeam, where Lightbeam must have access to certain types in order
to correctly implement access to runtime functions. This is currently implemented by simply copying
the relevant types and functions, which for our uses works for now. Obviously, though, this solution
isn't workable indefinitely.

## 2. Cleanup and removal of unnecessary code

Lightbeam has a lot of cruft due to attempted and subsequently rolled-back abstractions, and just
the general exploratory nature of the project combined with the low amount of manpower. A lot of
this can be cleaned up to make Lightbeam easier to work on.

- The label system is massively overengineered due to the fact that it seemed like far more
  complexity was going to be necessary. Now that we have little remaining complexity in the usecases
  of the label system, we can reduce the complexity.
- Most abstraction is done via macros, which are error-prone and difficult to work with.
  Unfortunately this isn't really avoidable with the current system where the backend handles
  everything. See #3 for some points on fixing this.
- The harness written for Lightbeam's tests before the Wasmtime integration is incomplete and since
  the migration from `wasmparser` to `wasm-reader` it no longer functions at all. The old tests are
  unnecessary and don't cover anything that Wasmtime's test suite doesn't, and the Wasmtime test
  suite is far easier to write new testcases in. The need to be abstract over Lightbeam test harness
  vs Wasmtime harness adds a lot of complexity that can be fixed by simply directly relying on
  Wasmtime's harness. This also means that entire files can be deleted wholesale.

## 3. Start abstracting and otherwise fixing up the codegen

Full explanation: [New Codegen Proposal][new-codegen]

## 4. Implementation of other backends

Examples of other backends that would be good to implement:

- 32-bit x86, both because it's widely-used and because it has a weird handling of floating-point
  numbers that will lead us to improving our abstractions
- ARMv7, because it's used on Raspberry Pis and other small, low-powered Linux devices
- AVR, because it's common on microcontrollers

[new-codegen]: ./proposals/new-codegen/
[stack-depth]: https://github.com/CraneStation/lightbeam/issues/27
[status]: ./status.md
