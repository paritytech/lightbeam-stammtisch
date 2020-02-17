# Backend Driver

> Relevant files:
> - [wasmtime/crates/lightbeam/src/function_body.rs][function_body.rs]

The backend driver is the loop that reads the Microwasm, calls the correct methods on the backend
for each instruction, and most importantly handles the control flow. You can think of the
separation like so: each Microwasm function is a flat list of blocks, each of which ends with a
single infallible jump. Each block is a flat list of instructions, each of which can perform no
control flow (apart from function calls, which start a new, independent stack frame and so
conceptually can be thought of as doing no control flow for our purposes here). The backend
operates on the level of instructions, the backend driver operates on the level of blocks. Each
block can be thought of as being compiled independently, in the same way that functions are.

TODO

[function_body.rs]: https://github.com/bytecodealliance/wasmtime/blob/master/crates/lightbeam/src/function_body.rs
