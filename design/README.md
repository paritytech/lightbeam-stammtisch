# Lightbeam design overview

As a streaming compiler, Lightbeam compiler is designed as a pipeline, where each instruction
should be processed directly from the Wasm module and be converted to zero or more instructions of
machine code. While this may seem self-evident, it's actually very different to how most compilers
are designed. The differences in design constraints are laid out further in [this
document][constraints]. This document is an overview of this pipeline, you can get a deeper dive
into each stage in the linked documents. This describes the pipeline _as it exists now_, and
proposals to change how this works are documented in the "proposals" section of this repository.

## Summary

To compile and run a Wasm module with Lightbeam, you first pass it to Wasmtime. Wasmtime parses out
the module and the function metadata, gets a list of all the functions and for each function,
passes it to Lightbeam for compilation.

Lightbeam reads a single WebAssembly instruction, converts that single instruction to zero or more
Microwasm instructions, and then iterates through these Microwasm instructions one at a time,
passing them to the backend. The backend reads a single Microwasm instruction, updates its internal
state, and then emits zero or more assembly instructions to the assembly buffer. It then proceeds
to the next Microwasm instruction until it has processed all Microwasm instructions that were
emitted for the single WebAssembly instruction that was read. Lightbeam then reads the next
WebAssembly instruction, and the process repeats until all instructions in the function has been
processed and a full block of assembly has been emitted for this function. Wasmtime then passes the
next function to Lightbeam, and the process repeats until all functions in the module have been
processed.

## Step 0: Wasmtime

> Full article: [Wasmtime integration][wasmtime]

Before the Wasm code gets to Lightbeam, it first passes through Wasmtime. While at one point
Lightbeam had its own infrastructure to handle the environment, certain important parts of the
implementation have since tracked changes in Wasmtime and so Wasmtime is now an inseparable part of
turning the result of Lightbeam's compilation into something that can actually be executed.

Wasmtime handles:

- Parsing Wasm modules, except for parsing the actual bodies of functions. Lightbeam just gets
  handed a byte array and parses it into Wasm instructions itself.
- Relocating function calls. Lightbeam compiles `call`/`call_indirect` (as well as any instruction
  that compiles to a library call, such as `f32.sqrt`) into a jump to a dummy location, along with a
  directive for Wasmtime to rewrite this location to point to the right place. More information in
  the full article.
- Implementing the runtime, so handling traps, supplying library functions, handling memory
  allocation, handling allocation of the VM context, and so forth.

## Step 1: Microwasm conversion

> Full article: [Microwasm][microwasm]

Microwasm is a typed, safe internal representation used in Lightbeam to add a layer of
simplification to WebAssembly. Because it's typed and safe, we can do transformations on it such as
dead code elimination while still having confidence in the correctness of the output. It also
vastly simplifies our handling of control flow (explained further in the full article), and this is
the main reason that this IR was implemented in the first place.

The Microwasm layer:

- Handles conversion of WebAssembly's hierarchical control flow into a flat list of typed blocks
  joined by infallible jumps (for a deeper explanation of what this means, see the full article)
- Handles WebAssembly's concept of "locals", unifying them with the stack representation used for
  other values.
- Performs basic dead code elimination. While DCE is far easier to perform on Microwasm,
  WebAssembly allows unreachable code to be incorrectly-typed and so in order to prevent Microwasm
  from sharing this unfortunate property we instead remove code that has the potential to be
  incorrectly-typed.

## Step 2: Backend Driver

> Full article: [Backend driver][backend-driver]

The backend "driver" is the high-level loop that calls into the backend itself. The backend does
not receive Microwasm instructions directly, the driver receives them from the Microwasm conversion
step and calls a number of methods on the backend. Most importantly, the backend driver is what
handles control flow, so the backend can operate purely on straight-line code.

The backend driver:

- Handles recording the "calling conventions" for blocks (i.e. where the block expects each
  argument to be when control flow is passed to it).
- Reads the Microwasm instructions and calls the relevant methods on the backend

## Step 3: Backend

> Full article: [Backend][backend]

The backend is where most of the complexity of Lightbeam lies. Although it only handles
straight-line code, it has to handle the conversion of this high-level stack-machine code into x86,
and this x86 needs to be efficient, and most importantly it needs to be safe.

The backend:

- Does register allocation and stack allocation for Wasm values
- Handles instruction selection, i.e. writing out the implementation in x86 assembly language for
  each Wasm instruction.
- Handles conforming to calling conventions, i.e. when control is passed to a new block or when a
  function is called, the arguments to that block or function are expected to be in specific places.
  The backend handles ensuring that these arguments are in those places.

## Step 4: Encoding

> Full article: [Encoding][encoding]

The conversion of x86 assembly language to bytes of machine code is handled by `dynasm`, a library
for Rust that handles this at compile-time. It also handles _intrafunction_ relocations (i.e.
relocations for jumps to other blocks in the same function), as opposed to _interfunction_
relocations which are handled by Wasmtime.

The Encoding step:

- Handles conversion of _specific_ instructions (i.e. not `add a, b` but `add r32, r32`, `add r32, m32` and so forth) to machine code bytes.
- Performs relocations for jumps between blocks.

[constraints]: ./constraints.md
[wasmtime]: ./wasmtime.md
[microwasm]: ./microwasm.md
[backend]: ./backend.md
[backend-driver]: ./backend-driver.md
[encoding]: ./encoding.md
