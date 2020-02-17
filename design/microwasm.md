# Microwasm

> *NOTE*: the phrase "Lightbeam IR" is often used to refer to Microwasm outside of the specific context of Lightbeam development, since it's a little more descriptive and can be easier to understand for outsiders. However, within discussions on Lightbeam development "Microwasm" is still used, as that's the name used in the code itself and it prevents the term "Lightbeam IR" from being overloaded if and when further levels of intermediate representation are introduced.

Microwasm is a typed, safe, stack-machine IR used internally in Lightbeam to simplify various parts of the compilation process. I go into deeper detail on the rationale behind the original introduction of this IR in the [WebAssembly Troubles][wasm-troubles] series on my blog, but this document should be enough to explain what Microwasm is, how it works, why it's useful as well as general thoughts about it. For proposals to extend or change Microwasm, see the [proposals][proposals] section.

TODO

[wasm-troubles]: http://troubles.md/posts/wasm-is-not-a-stack-machine/
[proposals]: ../proposals/microwasm/
