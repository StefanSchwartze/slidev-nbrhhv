---
# try also 'default' to start simple
theme: default
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
lineNumbers: false
info: |
  # Tree shaking ui libraries
# persist drawings in exports and build
drawings:
  persist: false
background: https://images.unsplash.com/photo-1473755504818-b72b6dfdc226?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2048&q=80
fonts:
  # basically the text
  # sans: 'Bungee Shade'
  sans: 'Nunito'
  # use with `font-serif` css class from windicss
  serif: 'Roboto Slab'
  # for code blocks, inline code, etc.
  mono: 'Fira Code'
---
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/prism-theme-vars/base.css">
<style>
@import "prism-theme-vars/base.css";
/*:root {
  --prism-foreground: #393a34;
  --prism-background: #fbfbfb;
  --prism-comment: red;
  --prism-string: #c67b5d;
  --prism-literal: #3a9c9b;
  --prism-keyword: #248459;
  --prism-function: blue;
  --prism-deleted: #a14f55;
  --prism-class: #2b91af;
  --prism-builtin: #a52727;
  --prism-property: #ad502b;
  --prism-namespace: #c96880;
  --prism-punctuation: #8e8f8b;
  --prism-decorator: #bd8f8f;
  --prism-json-property: #698c96;
}*/
</style>

# Tree shaking
### from a library's perspective

14.03.2022

<!-- Please talk to me directly for questions !-->
---

## A typical setup

✅  Application powered by Webpack<br/>

✅  UI-library installed via npm<br/>

✅  Importing a single component from root<br/>
---

😱 Looks good, but size doesn't



---

## What actually happened here?

⚠️  Problem: All components from lib loaded

-> Huge bundle, bad TTI (SSR), even worse for SPAs

---
layout: quote
---
## What is tree-shaking?

<br/>

> "Tree shaking is a term commonly used within a JavaScript context to describe the removal of dead code." [^1]

<br/>

- Tree-shaking happens in application's bundler
- Library is tree-shakable when all unused parts are skipped

[^1]: [MDN](https://developer.mozilla.org/en-US/docs/Glossary/Tree_shaking)

<!-- I always imagine it myself as a tree that has many branches and some or some sub-branches fall on the ground by shaking it !-->
---

## How to make it gonna shake

Using a bundler, e.g. Rollup and output the right code with correct format
—> in order to implement tree-shaking, the code must be statically analyzable

**CommonJS**: 

```ts
const { fetcher } = require("./api");

const getProfile = () => {
  return fetcher('/profile');
};

module.exports = { getProfile };
```
🚫 not tree-shakable

---

### But also allows dynamic requiring of code at runtime 

```ts

if (userIsLoggedIn) {
  const ProfileComponent = require("my-library").ProfileComponent;
}

```
<!-- This seems nice and flexible but prevents bundlers from effectively making a valid static application tree !-->
---

**ES Modules**: only use static imports (import xx from ..) 

```ts
import { fetcher } from "./api";

export const getProfile = () => {
  return fetcher('/profile');
};

```
✅  static and tree-shakable

---

## Setting up a configuation with rollup

![Rollup](https://seeklogo.com/images/R/rollup-js-logo-F3925E2546-seeklogo.com.png)

---

```ts{all|9|10-16|5,18|3,19|2,20|4,21|23|all}
// rollup.config.ts
import commonjs from '@rollup/plugin-commonjs';
import resolve from '@rollup/plugin-node-resolve';
import typescript from 'rollup-plugin-typescript2';
import del from 'rollup-plugin-delete';
import pkg from './package.json';

export default {
  input: ['src/index.ts', 'src/components/index.ts', 'src/utils/index.ts'],
  output: [
    {
      dir: 'lib',
      format: 'esm',
      sourcemap: true,
    },
  ],
  plugins: [
    del({ targets: ['lib/*'] }),
    resolve(), // Required to locate and resolve node_modules
    commonjs(), // Convert cjs-modules into ES6 format
    typescript(), // Transpile some TS
  ],
  external: [...Object.keys(pkg.dependencies), ...Object.key(pkg.peerDependencies)],
};
```
---

## Hint about es module in package.json

```json
/// package.json
{ 

  ...
  
  "module": "lib/index.js"
  
  ...

}
```

---

## What if not supported?

ES modules are only supported since Node v13.2
--> Provide a CommonJS export, too

```ts
// rollup.config.ts
export default {
  output: [
    {
      dir: 'dist',
      format: 'cjs',
      sourcemap: true,
    },
  ],
};
```

```json
/// package.json
{ 

  ...
  
  "main": "cjs/index.js"
  
  ...

}
```
<!-- This is helpful if you are doing SSR !-->

---




## Desired Export Module Graph

PICTURE TBD

---

## Mark package as side effect free

What are side effects? --> Global imports, e.g. Polyfills

```ts
import 'customPolyfill';
```
--> Out of the box, Webpack thinks every package has side effects and nothing can be skipped

```json
/// package.json
{ 

  ...
  
  "sideEffects": false
  
  ...

}
```

<!-- This flag can be read by most common bundlers !-->
---

## To bundle or not to bundle

If you bundle (popular e.g. for browsers), you possibly disable the `sideEffect` optimization
--> Modules cannot be skipped as everything is in one file
--> This is an issue if especially non-esm packages are imported

---

Preserve the structure in order to keep it tree-shakable!


```ts{8|3,9}
// rollup.config.ts
export default {
  input: ['src/index.ts', 'src/components/index.ts', 'src/utils/index.ts'],
  output: [
    {
      dir: 'lib',
      format: 'esm',
      preserveModules: true,
      preserveModulesRoot: 'src',
      sourcemap: true,
    },
  ],
};
```
<!-- Why use a bundler when you don't really bundle? It's transpiling, cjs etc. etc. -->
---

## Try again!

---

## And what about styling?

Use plugins like `rollup-plugin-postcss` or `rollup-plugin-styles` (better)
- Global import (downside: all is loaded at once..) or
- Local import, per component (downside: bad devx for small components)
- Did not get the components import css themselves 

---

## Why rollup?

- Rollup is super easy to set up
- Webpack tree-shaking works nice for applications, but not so nice for libraries
- Webpack5 introduced experiments.outputModule: true but is still relatively buggy (could not get it running with MiniCssExtractPlugin)
- ESBuild: [no support yet](https://github.com/evanw/esbuild/issues/708) and [not planned to implement](https://esbuild.github.io/api/#bundle)

---

## Best practices

- Use **ES(6)** module syntax -> no require
- Mark as **sideEffect free** —> read by application bundler, e.g. webpack
- Prefer 3rd-party packages with **ESM** output
- Node < v14 -> Provide **CJS** fallback if required
- Node >= v14 -> add `"type": "module"` to `package.json`
- **Preserve original structure** for full tree-shakability
- **Re-check** results in consuming application (e.g. with *webpack-bundle-analyzer*)

<!-- Since Node 15.3, es modules work without experimental flag -->
---

## Sources

[Francois Hendriks](https://blog.theodo.com/2021/04/library-tree-shaking/)
[Webpack - Tree-shaking](https://webpack.js.org/guides/tree-shaking/)
[Webpack - Webpack5 Release](https://webpack.js.org/blog/2020-10-10-webpack-5-release/)
[Rollup Docs](https://rollupjs.org/guide/en/)

---
