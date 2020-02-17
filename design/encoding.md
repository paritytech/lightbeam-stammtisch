# Encoding

> Relevant files: [dynasm-rs][dynasm]

[dynasm]: https://github.com/CensoredUsername/dynasm-rs

The encoding step takes a request for a specific form of a specific instruction (i.e. `add r32,
r32`, `add r32, m32` and so forth) and converts it to bytes. It is up to the backend to choose
which specific instruction is needed. The encoding step will also perform relocations for labels -
i.e. jumps and loads.

TODO
