# Import Reflection

## Status

Champion(s): Luca Casonato, Guy Bedford

Author(s): Luca Casonato, Guy Bedford

Stage: 1

## Motivation

For both JavaScript and WebAssembly, there is a need to be able to more closely
customize the loading, linking, and execution of modules beyond the standard
host execution model.

For JavaScript, creating userland loaders would require a module reflection type
in order to share the host parsing, execution, security, and caching semantics.

For WebAssembly, imports and exports for WebAssembly modules often require custom
inspection and wrapping in order to be set up correctly, which typically requires
manual instantiation work that would not necessarily be straightforward or
possible under the proposed host [ESM integration][wasm-esm] proposal.

Supporting module reflections syntactically as a new import form creates a
primitive that can extend the static, security and tooling benefits of modules
from the ESM integration to these dynamic instantiation use cases.

## Proposal

This proposal allows ES modules to import a reified representation of the
compiled source of a module when the host provides such a representation:

```js
import module x from "<specifier>";
```

The `module` reflection type is added to the beginning of the ImportStatement.

Only the above form is supported - named exports and unbound declarations are
not supported.

### Dynamic import()

```js
const x = await import("<specifier>", { reflect: "module" });
```

For dynamic imports, module import reflection is specified in the same second
attribute options bag that import assertions are specified in, using the
`reflect` key.

If the [asset references proposal][] advances in future it could share this same
`reflect` key for dynamic asset imports as being symmetrical reflections:

```js
import asset x from "<specifier>";
await import("<specifier>", { reflect: "asset" });
```

Only the `module` reflection is specified by this proposal.

### JS Reflection

The type of the reflection object for a JavaScript module is currently
host-defined, but the goal would be for this to be defined to be an object that
is compatible with objects defined by the [module blocks][] and the
[compartments][] specifications.

Specifying this object remains a TODO item for the proposal.

### Wasm Reflection

The type of the reflection object for WebAssembly would be a
`WebAssembly.Module`, as defined in the [WebAssembly JS integration
API][wasm-js-api].

The reflection would represent an unlinked and uninstantiated module, while
still being able to support the same CSP policy as the native [ESM
integration][wasm-esm], avoiding the need for `unsafe-wasm-eval` for custom Wasm
execution.

Since reflection is defined through a host hook, this Wasm reflection is left to
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

Other module types may define their own host reflections. A module reflection
may fail during the linking phase if it depends on a reflected import that the
host cannot satisfy for lack of an available reflection for the corresponding
module type.

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

The proposed approach would be a _clone_ behaviour, where imports to the same
module of different reflection types result in separate keys. These semantics do
run counter to the intuition that there is just one copy of a module.

The specification would then split the `HostResolveImportedModule` hook into two
components - module asset resolution, and module asset interpretation. The
module asset resolution component would retain the exact same idempotency
requirement, while the module asset interpretation component would have
idempotency down to keying on the module asset and reflection type pair.

Effectively, this splits the module cache into two separate caches - an asset
cache retaining the current idempotency of the `HostResolveImportedModule` host
hook, pointing to an opaque cached asset reference, and a module instance cache,
keyed by this opaque asset reference and the reflection type.

Alternative proposals include:

- **Race** and use the attribute that was requested by the first import. This
  seems broken in that the second usage is ignored.
- **Reject** the module graph and don't load if attributes differ. This seems
  bad for composition--using two unrelated packages together could break, if
  they load the same module with disagreeing attributes.

Both of these alternatives seem less versatile than the proposed _clone_
behaviour above.

## Q&A

**Q**: How does this relate to import assertions?

**A**: Import assertions do not influence how an imported asset is evaluated,
and they do not influence the HostResolveImportedModule idempotency
requirements. This proposal does. Also see
https://github.com/tc39/proposal-import-assertions#follow-up-proposal-evaluator-attributes.

**Q**: How does this relate to module blocks and compartments?

**A**: The module object that is reflected should be specified here to be
compatible with the linking model of module blocks and compartments.

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
[Wasm components to be able to import]:
    https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md#ESM-integration
[Wasm module object]:
    https://webassembly.github.io/spec/js-api/index.html#modules
[asset references proposal]: https://github.com/tc39/proposal-asset-references
[compartments]: https://github.com/tc39/proposal-compartments
[module-linking]:
    https://github.com/WebAssembly/module-linking/blob/main/proposals/module-linking/Binary.md#import-section-updates
[module blocks]: https://github.com/tc39/proposal-js-module-blocks
[wasm-js-api]: https://webassembly.github.io/spec/js-api/#modules
[wasm-esm]:
    https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration
