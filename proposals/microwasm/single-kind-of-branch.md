# Remove `BrIf` and `Br`

Right now, Microwasm has 1 kind of block, which starts with a `Label` and ends with one of `Br`,
`BrIf` or `BrTable`. We don't need to do this, though. A `Br` is just a `BrTable` with a single
entry, and a `BrIf` is identical to a `BrTable` with 2 entries. Collapsing these into a single
variant can simplify checking the starts and ends of blocks, remove some code, and allow us to
automagically convert simple `BrTable`s (which currently always emit the complex logic that a full
`BrTable` needs) into `Br`s and `BrIf`s. This is a very simple change that requires little work, but
it could make it slightly harder to understand as it brings us further from Wasm.
