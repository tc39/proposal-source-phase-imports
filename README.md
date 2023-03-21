# Import Source Reflection

## Status

Champion(s): Luca Casonato, Guy Bedford

Author(s): Luca Casonato, Guy Bedford, Nicolo Ribaudo

Stage: 2

Stage 3 reviewers: Daniel Ehrenberg, Kris Kowal

## Motivation

For both JavaScript and WebAssembly, there is a need to be able to more closely
customize the loading, linking, and execution of modules beyond the standard
host execution model.

For JavaScript, creating userland loaders would require a source reflection type
in order to share the host parsing, execution, security, and caching semantics.

For WebAssembly, imports and exports for WebAssembly modules often require custom
inspection and wrapping in order to be set up correctly, which typically requires
manual fetch and instantiation work that is not provided for in the current host
[ESM integration][wasm-esm] proposal.

Supporting syntactical source reflection as a  as a new import form creates a
primitive that can extend the static, security and tooling benefits of modules
from the ESM integration to these dynamic instantiation use cases.

## Proposal

This proposal allows ES modules to import a reified representation of the
compiled source of a module when the host provides such a representation:

```js
import module x from "<specifier>";
```

The `module` source phase reflection type is added to the beginning of the
ImportStatement.

Only the above form is supported - named exports and unbound declarations are
not supported.

### Dynamic import()

```js
const x = await import("<specifier>", { reflect: "module" });
```

For dynamic imports, import source reflection is specified in the same second
attribute options bag that import assertions are specified in, using the
`reflect` key.

### Evaluator Phase Reflection

Import source reflection can be seen to be one type of evaluator phase
reflection.

If the [asset references proposal][] advances in future this could be seen
as another type of phase reflection representing an earlier phase of the
loading process.

```js
import asset x from "<specifier>";
await import("<specifier>", { reflect: "asset" });
```

Only the `module` import source reflection is specified by this proposal.

### Defining Reflection

Source reflection is defined through the [ECMA-262 ES modules HostLoadImportedModule refactoring][],
which permits moves the construction of module records to the host, and on which
we can define a `[[ModuleSourceObject]]` custom object to be returned by module reflection that is
compatible with module expressions and import reflection.

### JS Reflection

The type of the reflection object for a JavaScript module is would depend
on the [module source]()
specification.

In the current specification, the JS reflection case just throws an error
to support this as a future addition.

### Wasm Reflection

The type of the reflection object for WebAssembly would be a
`WebAssembly.Module`, as defined in the [WebAssembly JS integration
API][wasm-js-api].

The reflection would represent an unlinked and uninstantiated module, while
still being able to support the same CSP policy as the native [ESM
integration][wasm-esm], avoiding the need for `unsafe-wasm-eval` for custom Wasm
execution.

Since reflection is defined through the hook, this Wasm reflection is left to
be specified in the Wasm [ESM Integration][wasm-esm].

This allows workflows, as explained in the motivation, like the following:

```js
import module FooModule from "./foo.wasm";
FooModule instanceof WebAssembly.Module; // true

// For example, to run a WASI execution with an API like Node.js WASI:
import { WASI } from 'wasi';
const wasi = new WASI({ args, env, preopens });

const fooInstance = await WebAssembly.instantiate(FooModule, {
  wasi_snapshot_preview1: wasi.wasiImport
});

wasi.start(fooInstance);
```

The static analysis benefits of not needing a custom `fetch` and
`WebAssembly.compileStreaming` apply not only to code analysis and security
but also for bundlers.

In turn this enables [Wasm components to be able to import][]
`WebAssembly.Module` objects themselves in future.

### Other Module Types

Other module types may define their own host reflections. If no module reflection
is defined, it will fail during the loading phase.

## Security Benefits

Tracking the origins of scripts or modules is important for protecting programs
from cross-site scripting attacks, for example using [Content Security
Policies][CSP]. Extending this behaviour to dynamic module reflections enables
custom loaders while retaining these security benefits.

Wasm compilation is unfortunately completely dynamic right now (manual network
fetch & compile), so Wasm unconditionally requires a
`script-src: unsafe-wasm-eval` CSP attribute.

With this proposal, the JS and Wasm module reflections would be known statically,
so would not have to be considered as dynamic code generation. This would allow
the web platform to enabling dynamic instantiation for these modules while
lifting the restriction of an arbitrary evaluation CSP policy and instead just
require `script-src: self` (or `wasm-src: self`). Also see
https://github.com/WebAssembly/esm-integration/issues/56.

This property does not just impact platforms using CSP, but also other platforms
with systems to restrict permissions, such as Deno.

## Cache Key Semantics

Because `[[ModuleSourceObject]]` is keyed on the base module record, it will always
be unique to the module being imported from.

## Q&A

**Q**: How does this relate to evaluator attributes?

**A**: Evaluator attributes affect the module request, while phase reflections
do not affect the interpretation or idempotency of the module load.

**Q**: How does this relate to module expressions and compartments?

**A**: The module object that is reflected has been carefully specified here to be
compatible with the linking model of module expressions and compartments.

**Q**: Would this proposal enable the importing of other languages directly as
modules?

**A**: While hosts may define import reflection, expanding the evaluation of
arbitrary language syntax to the web is not seen as a motivating use case for
this proposal.

**Q**: Why not just use `const module = await
WebAssembly.compileStreaming(fetch(new URL("./module.wasm",
import.meta.url)));`?

**A**: There are multiple benefits: firstly if the module is statically
referenced in the module graph, it is easier to statically analyze (by bundlers
for example). Secondly when using CSP, `script-src: unsafe-eval` would not be
needed. See the security improvements section for more details.

[CSP]:
    https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy
[ECMA-262 ES modules HostLoadImportedModule refactoring]:
    https://github.com/tc39/ecma262/pull/2905
[Wasm components to be able to import]:
    https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md#ESM-integration
[Wasm module object]:
    https://webassembly.github.io/spec/js-api/index.html#modules
[asset references proposal]: https://github.com/tc39/proposal-asset-references
[compartments]: https://github.com/tc39/proposal-compartments
[module-linking]:
    https://github.com/WebAssembly/module-linking/blob/main/proposals/module-linking/Binary.md#import-section-updates
[module expressions]: https://github.com/tc39/proposal-module-expressions
[module source]: https://github.com/tc39/proposal-compartments/blob/master/0-module-and-module-source.md#modulesource
[wasm-js-api]: https://webassembly.github.io/spec/js-api/#modules
[wasm-esm]:
    https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration
