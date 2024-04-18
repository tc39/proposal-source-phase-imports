# Source Phase Imports

## Status

Champion(s): Luca Casonato, Guy Bedford

Author(s): Luca Casonato, Guy Bedford, Nicolo Ribaudo

Stage: 3

Stage 3 reviewers: Daniel Ehrenberg, Kris Kowal

## Motivation

For both JavaScript and WebAssembly, there is a need to be able to more closely
customize the loading, linking, and execution of modules beyond the standard
host execution model.

For JavaScript, creating userland loaders would require a module source type
in order to share the host parsing, execution, security, and caching semantics.

For WebAssembly, imports and exports for WebAssembly modules often require custom
inspection and wrapping in order to be set up correctly, which typically requires
manual fetch and instantiation work that is not provided for in the current host
[ESM integration][wasm-esm] proposal.

Supporting syntactical module source imports as a new import phase creates a
primitive that can extend the static, security and tooling benefits of modules
from the ESM integration to these dynamic instantiation use cases.

## Proposal

This proposal allows ES modules to import a reified representation of the
compiled source of a module when the host provides such a representation:

```js
import source x from "<specifier>";
```

The `source` module source loading phase name is added to the beginning of the
ImportStatement.

Only the above form is supported - named exports and unbound declarations are
not supported.

### Dynamic form

Just as with static and dynamic imports, there is a need for static and dynamic access
to sources, to be able to support both those sources that are required to be instantiated
from source text during initialization of an application, and those that are optionally or
lazily created at runtime.

The dynamic form uses a `import.<phase>` import call:

```js
const x = await import.source("<specifier>");
```

By making the phase part of the explicit syntax, it is possible to statically distinguish between
a full dynamic import and one that is only for a source (where dependencies don't need to be
processed).

Optional [import attributes][] may still be specified with the second argument in a `with` key,
just like for dynamic import, and without conflict due to the design of phased evaluation.

### Loading Phase

Module source imports can be seen to be one type of evaluation phase.

If the [asset references proposal][] advances in future this could be seen
as another type of phase representing an earlier phase of the loading process.

```js
import asset x from "<specifier>";
await import.asset("<specifier>");
```

Only the `source` import source phase is specified by this proposal.

### Defining Module Source

The object provided by the module source phase must be an object with
`AbstractModuleSource.prototype` in its prototype chain, defined by this specification
to be a minimal shared base prototype for a compiled modular resource.

In addition it defines the `@@toStringTag` getter returning the constructor name string
corresponding to the name of the specific module source subclass, with a strong
internal slot check.

### JS Module Source

For JavaScript modules, the module source phase is then specified to return
a `ModuleSource` object, representing an ECMAScript Module Source, where
`ModuleSource.prototype.[[Proto]]` is `%AbstractModuleSource%.prototype`.

Future proposals may then add support for [bindings lookup methods][],
the [ModuleSource constructor][module soruce] and [instantiation][] support.

New properties may be added to the base `%AbstractModuleSource%.prototype`, or shared
with ECMAScript module sources via `ModuleSource.prototype` additions.

### Wasm Module Source

For WebAssembly modules, the existing `WebAssembly.Module.prototype` object is to be
updated to have a `[[Proto]]` of `%AbstractModuleSource%.prototype` in the
[WebAssembly JS integration API][wasm-js-api].

This allows workflows, as explained in the motivation, like the following:

```js
import source FooModule from "./foo.wasm";
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

Any other host-defined module types may define their own host module sources. If a given module does not define a source representation for it's source, importing it with a "source" phase target fails with a `ReferenceError` at link time.

Host-defined module sources must include `%AbstractModuleSource%.prototype` in their prototype chain and support the `[[ModuleSourceRecord]]` internal slot containing the `@@toStringTag` brand check and underlying source host data.

## Security Benefits

The native ES module loader is able to implement security policies, including
support for [Content Security Policies][CSP] in browsers. This property does not just impact platforms using CSP, but also other platforms with systems to restrict permissions, such as Deno. These policies are based on protecting which URLs are supported for the compilation and execution of scripts or modules.

Extending the static security benefits of the host module system to custom loaders is a security benefit of this proposal. For Wasm, it would enable source-specific CSP policies for dynamic Wasm instantiation.

## Cache Key Semantics

Because `[[ModuleSourceObject]]` is keyed on the base module record, it will always
be unique to the module being imported from.

## Q&A

**Q**: How does this relate to import attributes?

**A**: Import attributes are properties of the module request, while source imports
represent phases of that specific request / key in the module map, without affecting
the idempotency of the module load. Both can be used together for a resource to indicate alternative phasing for the given module resource and attributes.

**Q**: How does this relate to module expressions and compartments?

**A**: The module object that is provided has been carefully specified here to be
compatible with the linking model of module expressions and compartments.

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
[bindings lookup methods]: https://github.com/tc39/proposal-compartments/blob/master/1-static-analysis.md
[compartments]: https://github.com/tc39/proposal-compartments
[import attributes]: https://github.com/tc39/proposal-import-attributes/
[instantiation]: https://github.com/tc39/proposal-compartments/blob/master/0-module-and-module-source.md#module-instances
[module-linking]:
    https://github.com/WebAssembly/module-linking/blob/main/proposals/module-linking/Binary.md#import-section-updates
[module expressions]: https://github.com/tc39/proposal-module-expressions
[module source]: https://github.com/tc39/proposal-compartments/blob/master/0-module-and-module-source.md#modulesource
[wasm-js-api]: https://webassembly.github.io/spec/js-api/#modules
[wasm-esm]:
    https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration
