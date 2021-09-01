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

The syntax proposed here is to add a field for an optional string literal to all
imports. This string literal can will represent instructions for the module
loader, on how to evaluate a given specifier.

### Import statements

```js
import x from "<specifier>" as "<evaluator-attribute>";
```

Here the evaluator attribute is added to the end of the ImportStatement,
prefixed by the "as" keyword. If combined with an import assertion, the
assertion must always come after the evaluator attribute:

```js
import x from "<specifier>" as "<evaluator-attribute>" asserts {};
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

> This section of the proposal is not aimed at TC39. It is background info for
> how this can integrate into other specifications, such as HTML or
> webassembly-esm.

### HTML spec

On the web platform, there are other APIs that have the ability to load ES
modules. These also need to be able to specify evaluator attributes. Here are
some examples of how this could look.

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

### WebAssembly

WebAssembly is working to integrate with the ESM module graph in the
`webassembly-esm` integration proposal. There are multiple ways in which this
proposal interacts with that:

Import a WebAssembly binary as a compiled module:

##### `WebAssembly.Module` imports

```js
import mod from "./foo.wasm" as "wasm-module";
mod instanceof WebAssembly.Module; // true
```

This is explained in the "Motivation" section above. A `"wasm-module"` evaluator
attribute would be added that changes the behaviour of importing a .wasm file:
it would now just be compiled, but not instantiated or linked.

#### Imports from WebAssembly

With this wasm-esm proposal, WebAssembly gets the ability to directly import
ESM. This means that it also needs to represent evaluator attributes in its
imports. To make this work, a new custom section (named `evaluatorattributes`)
would be addedm that will annotate each imported module with evaluator
attributes.

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
