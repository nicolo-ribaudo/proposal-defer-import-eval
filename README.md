# Deferring Module Evaluation

previously known as "Lazy Module Initialization"

## Status

Champion(s): *Yulia Startsev*

Author(s): *Yulia Startsev and Guy Bedford*

Stage: 1

[Stage 1 Slides](https://docs.google.com/presentation/d/17NsxHzAC2RlP5rB3wrns9O2Z-NduSpcm2_GOVo2TnKE/edit#slide=id.p)

## Background

JS applications can get very large, to the point that not only loading, but even executing their
initialization scripts incurs a significant performance cost. Usually, this happens later in an application's
life span - often requiring invasive changes to make it more performant.

Loading performance is a big and important area for improvement, and involves preloading techniques for
avoiding waterfalls and dynamic `import()` for lazily loading modules.

But even with loading performance solved using these techniques, there is still overhead for execution
performance - CPU bottlenecks during initialization due to the way that the code itself is written.

## Motivation

Avoiding unnecessary execution is a well-known optimization in the Node.js CommonJS module system,
where there is a smaller gap between load contention and execution contention. The common pattern
in Node.js applications is to refactor code to dynamically require as needed:

```js
const operation = require('operation');

exports.doSomething = function (target) {
  return operation(target);
}
```

being rewritten as a performance optimization into:

```js
exports.doSomething = function (target) {
  const operation = require('operation');
  return operation(target);
}
```

The consumer still is provided with the same API, but with a more efficient use of FS & CPU during
initialization time.

For ES modules, we have a solution for the lazy loading component of this problem via dynamic `import()`.

For the same example we can write:

```js
export async function doSomething (target) {
  const { operation } = await import('operations');
  return operation(target);
}
```

This avoids bottlenecking the network and CPU during application initialization, but there are still a
number of problems with this technique:

1. It doesn't actually solve the deferral of execution problem, since sending a network
   request in such a scenario would usually be a performance regression and not an improvement.
   A separate network preloading step would therefore still be desirable to achieve efficient
   deferred execution while avoiding triggering a waterfall of requests.
  
2. It forces all functions and their callers into an asynchronous programming model,
   without necessarily reflecting the real intention of the program. This leads to all call
   sites having to be updated into a new model, and cannot be made without a breaking API
   change to existing API consumers.

## Problem Statement

Deferring the synchronous evaluation of a module may be desirable new primitive to avoid unnecessary
CPU work during application initialization, without requiring any changes from a module API consumer
perspective.

Dynamic import does not properly solve this problem, since it must often be coupled with a preload step,
and enforces the unnecessary asyncification of all functions, without providing the ability to only defer
the synchronous evaluation work.

## Proposal

The proposal is to have a new syntactical import form which will only ever return an object binding
to a new exotic object - a `DeferredModuleNamespace`.

When used, the module and its dependencies would not be executed, but would be fully loaded to the point
of being execution-ready before the module graph is considered loaded.

_Only when accessing a property of this module, would the execution operations be performed (if needed)._

This way, `DeferredModuleNamespace` acts like a proxy to the evaluation of the module, effectively
with getter functions that trigger synchronous idempotent evaluation before returning the defined bindings.

The API can take a couple of forms, here are some suggestions:

```js
// using import attributes (`lazyInit` to illustrated, but it could also be `deferEval`, say)
import lazyModuleNamespace from "y" with { lazyInit: true }

// with an explicit "*" to indiciate the namespace form
import * as lazyModuleNamespace from "y" with { lazyInit: true }

// or with a custom keyword:
lazy import * as yNs from "y";
```

There are other proposals, but the discussion of the naming may derail the discussion of the
semantics, which should be agreed on first and would likely guide the design of the API.

## Semantics

The semantics would be similar to import reflection, in that the deferred module namespace can be
thought of as a leaf node that does not add further branches to the execution graph.

The imports would still participate in deep graph loading so that they are fully populated into
the module cache prior to execution.

When a property of a `DeferredModuleNamespace` is accessed, if the execution has not already
been performed, a new top-level execution would be initiated for that module.

In this way, a deferred module evaluation import acts as a new top-level execution node
in the execution graph, just like a dynamic import does, except executing synchronously.

If accessing a graph subject to top-level await, an error would be thrown. Eager execution of the
asynchronous components of top-level await graphs could be a possible option for enabling these
use cases as well.

### Rough sketch

If we split out the components of Module loading and initialization, we could roughly sketch out the
intended semantics:

```js
// LazyModuleLoader.js
async function loadModuleAndDependencies(name) {
  const loadedModule = await import.load(`./${name}.js`); // load is async, and needs to be awaited
  const parsedModule = loadedModule.parse();
  await Promise.all(parsedModule.imports.map(loadModuleAndDependencies)); // load all dependencies
  return parsedModule;
}

export default async function lazyModule(object, name) {
  const module = await loadModuleAndDependencies(name);
  Object.defineProperty(object, name, {
    get: function() {
      delete object[name];
      const value = module.eval();
      Object.defineProperty(object, name, {
        value,
        writable: true,
        configurable: true,
        enumerable: true,
      });
      return value;
    },
    configurable: true,
    enumerable: true,
  });

  return object;
}

// myModule.js
import foo from "./bar";

etc.

// module.js
import LazyModule from "./LazyModuleLoader";
await LazyModule(globalThis, "myModule");

function Foo() {
  myModule.doWork() // first use
}
```

## Implementations

None so far.

## Q&A

#### What happened to the direct lazy bindings?

The initial version of this proposal included direct binding access for deferred evaluation via
named exports:

```js
import { feature } from './lib' with { lazyInit: true }

export function doSomething (param) {
  return feature(param);
}
```

where the deferred evaluation would only happen on _access_ of the `feature` binding.

There are a number of complexities to this approach, as it introduces a novel
type of execution point in the language, which would need to be worked through.

This approach may still be investigated in various ways within this proposal or an extension of it,
but by focusing on the `DeferredModuleNamespace` accessor approach first, it keeps the semantics
simple and in-line with standard JS techniques.

#### Is there really a benefit to optimizing execution, when surely loading is the bottleneck?

While it is true that loading time is the most dominant factor on the web, it is important to consider that many
large applications can block the CPU for of the range of 100ms while initializing the main application graph.

Loading times of the order of multiple seconds often take the focus for performance optimization work, and this
is certainly an important problem space, but the problem of freeing up the main event loop during initialization
remains a critical one when the network problem is solved, that doesn't currently have any easy solutions today
for large applications.

#### Is there prior art for this in other languages?

The standard libraries of these programming languages includes related functionality:
- Ruby's `autoload`, in contrast with `require` which works in the same way as JS `import`
- Clojure `import`
- Most LISP environments

Our approach is pretty similar to the Emacs Lisp approach, and it's clear from a manual analysis of billions of Stack Overflow posts that this is the most straightforward to ordinary developers.

#### Why not support a synchronous evaluation API on ModuleInstance

A synchronous evaluation API on the module expression and compartments [ModuleInstance](https://github.com/tc39/proposal-compartments/blob/master/0-module-and-module-source.md#module-instances)
object could offer an API for synchronous evaluation of modules, which could be compatible with
this approach of deferred evaluation, but it is only in having a clear syntactical solution for this use case,
that it can be supported across dependency boundaries and in bundlers to bring the full benefits of avoiding unnecessary
initialization work to the wider JS ecosystem.

#### What is the problem with supporting top-level await?

Top Level Await introdues a wrench into the works. If evaluation is delayed until a module is used,
then a module with top level await may evaluate in the context of sync code, for example:

```js
// module.js
// ...
const data = await fetch("./data");

export { data };

// main.js
import mod from "./module.js" with { lazyInit: true };

function run() {
  doSomeProcessing(mod.data);
}
```

In order for this to work, we would need to pause in the middle of `run`, a sync function. However,
this is not allowed by the run to completion invariant.

There are two possibilities for how to handle this case:
1) Throw an error if we come across an async module during the parse step
2) Preemptively execute the async components of a deferred import -- possibly with a warning

The lazy graph will in any case have some already-evaluated nodes, as other parts of
the module graph may have been part of separate top-level evaluation graphs.


#### What can we do in current JS to approximate this behavior?

The closest we can get is the following:

```js
// moduleWrapper.js
export default function ModuleWrapper(object, name, lambda) {
  Object.defineProperty(object, name, {
    get: function() {
      // Redefine this accessor property as a data property.
      // Delete it first, to rule out "too much recursion" in case object is
      // a proxy whose defineProperty handler might unwittingly trigger this
      // getter again.
      delete object[name];
      const value = lambda.apply(object);
      Object.defineProperty(object, name, {
        value,
        writable: true,
        configurable: true,
        enumerable: true,
      });
      return value;
    },
    configurable: true,
    enumerable: true,
  });
  return object;
}

// module.js
import ModuleWrapper from "./ModuleWrapper";
// any imports would need to be wrapped as well

function MyModule() {
 // ... all of the work of the module
}

export default ModuleWrapper({}, "MyModule", MyModule);

// parent.js
import wrappedModule from "./module";

function Foo() {
  wrappedModule.MyModule.bar() // first use
}
```

However, this solution doesn't cover deferring the loading of submodules of a lazy graph, and would
not acheive the characteristics we are looking for.
