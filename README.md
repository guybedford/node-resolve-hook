### Introduction

The NodeJS module system interfaces for CommonJS are transparently available through both private and public APIs, allowing users to hook into custom resolutions as needed to alter the NodeJS resolution algorithm and loading process with a high amount of flexibility.

With the introduction of a C++ API for ES module loading, this same transparency won't be possible. So if these features
are to continue to be supported, they would need to have their own user-facing API provided.

### Basic Hooks

Rather than try to specify a fully hookable interface like the [WhatWG Loader Specification](https://github.com/whatwg/loader), the goal here is to specify the minimum hook surface area to achieve the same use cases that are already in use in NodeJS today.

For now, this document is suggesting only a **resolver hook**. There is the scope for a `fetch` or `translate` hook as in the WhatWG specification, alternatively a more fine-trained instantiation hook could be a possible addition that captures both of these in future for fullly fine-grained control over the module semantics, but this may be reliant on a more stable V8 API and module wrapper concept. This would enable the full depth of APM use cases.

In this document, `module` is used as the `import module from 'module'` top-level API in NodeJS.

## Resolve Hook

Unlike the [WhatWG loader spec resolve hook](https://whatwg.github.io/loader/#resolve), this hook returns an object containing the `resolved` string URL of the module as well as any additional metadata that might be needed.

Initially the only other property included is a `format` property, which specifies the module format of the resolved module as one of `"esm", "cjs", "wasm", "json", "binary", "native"`, indicating the parse goal of the resolved module.

A resolve hook is a function:

```
async resolve (name: string, parentModuleUrlOrString: string) => { url: URL, format: string }
```

where both _parentModuleUrlOrString_ and the value of `resolved` are fully-formed URLs according to the URL specification.

The NodeJS resolver should ideally then be exposed through a public API to remain user-accessible, something along the lines of `require('module').resolve`.

The entire resolve function should be hookable to allow maximum flexibility. Custom resolvers can then build on top of the NodeJS `module.resolve` implementation.

This hook then enables full experimentation of custom resolution of modules (including remote modules).

If the return value is not a valid File URL, an error is thrown. NodeJS core modules should be referred to using the `node:[internalname]` internal URI scheme. Evaluated scripts or custom VM executions are similarly assigned a unique `eval:[uniqueId]` URI when hooking into the resolver resolutions.

The resolve hook here would only apply to the resolution algorithm applied for ES modules, and would not apply to resolutions of CommonJS modules from within other CommonJS modules, as the CJS resolver remains fully backwards compatible with its existing implementation in Node.

### Hook Registration

Registration of loader hooks requires a top-level API. For this we use a `setModuleResolver` method on the `module` object:

```js
import module from 'module';
module.setModuleResolver(resolverHookFunction);
```

Hook registration is application-level and applies to all modules loaded in the application after the registration function has been called.
When multiple calls to `setModuleResolver` are made, the entire resolver is replaced by the most recent `setModuleResolver` call.

The issue that arises when creating module loader hooks is that they themselves must be loaded as modules, so any modules loaded as part of the hook registration process will miss the hook pipeline. This isn't a problem though if we consider the registration of hooks as forming a boot phase. During this boot loading, the module graph is loaded into the same registry but without any of the hooks applying. Once the hooks are added, then the application level loading can begin. This is fully in line with standard bootstrapping principles.

These hooks could be wrapped up by a custom interface similar to the way `@std/esm` works today, with all the above reducing to a workflow something like:

```js
import 'custom-loader/register';
// registration is now complete -> load the app
import('app');
```

The separation of the boot phase here is thus straightforward to see.
