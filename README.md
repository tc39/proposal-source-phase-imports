# Import Evaluator Attributes

## Status

Champion(s): Guy Bedford

Author(s): Luca Casonato

Stage: 0

## Motivation

The WebAssembly [ECMAScript module integration proposal][wasm-esm] proposes direct
integration of WebAssembly into the ES module system based on sharing JS and Wasm
_module instances_ resolved by JS and Wasm imports.

The Web Assembly [Module Linking Proposal][] extends the requirements for the
ESM host integration to support a secondary type of module import - a
_module type import_ as distinct from an _instance import_. Where an instance import
would return a fully linked and evaluated `WebAssembly.Instance`, a module import
would return the compiled but unlinked and unexecuted `WebAssembly.Module` object.

In the module linking proposal both types of object are importable and can be
associated with the same imported Wasm module asset. In essence, the module type import
provides the module constructor to create custom instances as having separate
linear memory and linked bindings.

The ability to obtain a module or an instance provides higher level control over the
Wasm execution sandbox, which ends up being an important practical requirement in
many Web Assembly workflows.

The requirement for the ESM integration would thus be that two imports of the same
asset may return a distinct ES module result based on some import hint.

This proposal, using the principle of equal capability, proposes a similar hint
to be in the form of ECMAScript syntax that enables the module import syntax
to provide extra attributes along with a given import as evaluator attributes
with the primary use case in mind of permitting these module type imports for Wasm.

[Module Linking Proposal]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Explainer.md
[wasm-esm]: https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration

## Proposal

The proposal is to permit an optional string literal attribute to be associated
with any import:

```js
import x from "<specifier>" as "<evaluator-attribute>";
```

Here the evaluator attribute is added to the end of the ImportStatement,
prefixed by the "as" keyword. If combined with an import assertion, the
assertion must always come after the evaluator attribute:

```js
import x from "<specifier>" as "<evaluator-attribute>" assert {};
```

It is also possible to specify an evaluator attribute if the import is not
bound:

```js
import "<specifier>" as "<evaluator-attribute>";
```

### Re-export statements

```js
export { default as x } from "<specifier>" as "<evaluator-attribute>";
```

For re-export statements, the syntax is essentially identical to import
statements. Import assertions must also appear _after_ evaluator attributes.

### Dynamic import()

```js
const x = await import("<specifier>", { as: "<evaluator-attribute>" });
```

For dynamic imports, evaluator attributes are specified in the same second
attribute options bag that import assertions are specified in. The ordering of
`asserts` vs `as` does not matter here.

## Integration with other specs and environments

### WebAssembly ESM Integration

Import a WebAssembly binary as a compiled module:

##### `WebAssembly.Module` imports

As explained in the motivation, supporting a `"wasm-module"` evaluator
attribute is a primary motivation for this specificatoin in order to change the behaviour of
importing a direct compiled but unlinked [Wasm module object](https://webassembly.github.io/spec/js-api/index.html#modules):

```js
import mod from "./foo.wasm" as "wasm-module";
mod instanceof WebAssembly.Module; // true
```

#### Imports from WebAssembly

Web Assembly would have the ability to reflect evaluator attributes in its
imports if desired, as the [Module Linking proposal currently aims to specify](https://github.com/WebAssembly/module-linking/blob/main/proposals/module-linking/Binary.md#import-section-updates).

### HTML spec

On the web platform, there are other APIs that have the ability to load ES
modules. These also need to be able to specify evaluator attributes. Here are
some examples of how these might look.

#### Web Workers

```js
const worker = new Worker("<specifier>", {
  type: "module",
  as: "<evaluator-attribute>",
});
```

#### `<script>` tags

```html
<script href="<specifier>" type="module" as="<evaluator-attribute>" ></script>
```

## Cache Key Semantics

Semantically this proposal involves a relaxation of the `HostResolveImportedModule` idempotency requirement.

The proposed approach would be a _clone_ behaviour, where imports to the same module of different
evaluator attributes result in separate instances. These semantics do run counter to the intuition
that there is just one copy of a module.

The specification would then split the `HostResolveImportedModule` hook into two components -
module asset resolution, and module asset interpretation. The module asset resolution component
would retain the exact same idempotency requirement, while the module asset interpretation component
would have idempotency down to keying on the module asset and evaluator type pair.

Effectively, this splits the module cache into two separate caches - an asset cache retaining the
current idempotency of the `HostResolveImportedModule` host hook, pointing to an opaque cached asset reference,
and a module instance cache, keyed by this opaque asset reference and the evaluator attribute.

Alternative proposals include:

- **Race** and use the attribute that was requested by the first import. This
  seems broken in that the second usage is ignored.
- **Reject** the module graph and don't load if attributes differ. This seems
  bad for composition--using two unrelated packages together could break, if
  they load the same module with disagreeing attributes.

Both of these alternatives seem less versatile than the proposed _clone_ behaviour above.

## Q&A

**Q**: How does this relate to import assertions?

**A**: Import assertions do not influence how an imported asset is evaluated, and they
do not influence the HostResolveImportedModule idempotency requirements. This
proposal does. Also see
https://github.com/tc39/proposal-import-assertions#follow-up-proposal-evaluator-attributes.

**Q**: Are there use cases for this feature other than Web Assembly imports?

**A**: One way of thinking about evaluator attributes is that they vary the representation
of a module being imported. If JS itself ever wanted to reflect module imports at a higher
level of abstraction that is something that might be enabled by this work, for example being
able to import an unexecuted JS module object that could be passed around. Simiarly, any asset
imports that might have different representations in JS would benefit from evaluator attributes
that vary the way in which the module is reflected through importing.

**Q**: Would this proposal enable the importing of other languages directly as modules?

**A**: While hosts may define evaluator attributes they see fit, expanding the evaluation of
arbitrary language syntax to the web is not seen as a motivating use case for this proposal.
