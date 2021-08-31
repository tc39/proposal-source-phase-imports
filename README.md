# Import Evaluator Attributes

## Status

Champion(s): _none_

Author(s): Luca Casonato

Stage: -1

## Motivation

The WebAssembly ECMAScript module integration proposal [proposes][wasm-esm] deep
integration of WebAssembly into the ESM system. It automatically compiles,
instantiates, and binds WASM executables, allowing for easy calling between
WebAssembly and ECMAScript.

In [an issue][high-order-integration], Guy Bedford proposed to change the ESM
integration to only compile WASM modules on import (with the only export being a
`default` `WebAssembly.Module`). This has the benefits of being able to manually
instantiate a WebAssembly instance, which gives the developer much more control
over the sandbox the WASM executes in.

This discussion led to the conclusion that we probably want both:

- import a wasm module that gets instantiated by the ESM loader (as
  ESM-integration is written today) and
- import a wasm module as a WebAssembly.Module object.

This means that two imports of the same asset should evaluate differently based
on some hint. This proposal proposes this hint to be in the form of ECMAScript
syntax.

This requires that the ES import syntax allow for extra attributes to be
provided on an import that change how the referenced asset should be
interpreted: import evaluator attributes.

[wasm-esm]: https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration
[high-order-integration]: https://github.com/WebAssembly/esm-integration/issues/44

## Proposed syntax

> I don't care about the exact syntax. This is the minimal viable syntax that
> solves the issue and can be trivially expanded to usecases outside of WASM
> modules.

### Import statements

Import a WebAssembly binary as a compiled module:

```js
import mod from "./foo.wasm" as "wasm-module";
mod instanceof WebAssembly.Module; // true
```

### Re-export statements

```js
export { default as wasmModule } from "./foo.wasm" as "wasm-module";
```

### Dynamic import()

```js
const { default: mod } = await import("./foo.wasm", { as: "wasm-module" });
```

## Semantics

This proposal proposes that for each different combination of module specifier
and evaluator attributes, a different copy of the module is stored in the graph.
Modules are **cloned**. In this case, the attributes would have to be part of
the cache key. These semantics would run counter to the intuition that there is
just one copy of a module.

Alternative proposals include:

- **Race** and use the attribute that was requested by the first import. This
  seems broken--the second usage is ignored.
- **Reject** the module graph and don't load if attributes differ. This seems
  bad for composition--using two unrelated packages together could break, if
  they load the same module with disagreeing attributes.

Both of these are less versatile than the proposed "clone" behaviour.

## Q&A

**Q**: How is this different from import assertions?

**A**: Import assertions dont influence how an imported asset is evaluated. This
does. Also see
https://github.com/tc39/proposal-import-assertions#follow-up-proposal-evaluator-attributes.
