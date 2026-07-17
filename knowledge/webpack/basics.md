# webpack Basics

> Official Documentation: https://webpack.js.org/concepts/

## Overview

webpack is a static module bundler for modern JavaScript applications. It builds
a dependency graph starting from one or more entry points, then combines every
module your project needs (JS, CSS, images, etc.) into one or more optimized
bundles for the browser.

This documentation targets **webpack 5** (latest 5.x, e.g. `5.108.x`).

Key ideas introduced or matured in webpack 5:

- Built-in **Asset Modules** (no `file-loader`/`url-loader`/`raw-loader` needed).
- **Persistent filesystem caching** for faster rebuilds.
- **Module Federation** for sharing code between separately built apps.
- Improved **tree shaking** and long-term caching (deterministic module IDs).

## Core Concepts

| Concept | Purpose |
| ------- | ------- |
| **Entry** | The module where webpack starts building the dependency graph. |
| **Output** | Where and how webpack emits the resulting bundles. |
| **Loaders** | Transform non-JS files (or transpile JS) into valid modules. |
| **Plugins** | Hook into the build to perform wider tasks (asset generation, injection, optimization). |
| **Mode** | `development`, `production`, or `none` — sets sensible built-in defaults. |
| **Module** | Any file/dependency in the graph (ESM, CommonJS, CSS, assets). |

## Installation

webpack is used as a local dev dependency. `webpack-cli` provides the command
line interface.

```bash
npm install --save-dev webpack webpack-cli
```

Verify the install:

```bash
npx webpack --version
```

## Entry and Output

The `entry` tells webpack where to start; `output` tells it where to write.

```js
// webpack.config.js
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
  },
};
```

### Multiple entry points

```js
module.exports = {
  entry: {
    app: './src/app.js',
    admin: './src/admin.js',
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].bundle.js', // app.bundle.js, admin.bundle.js
    clean: true,                  // clear the output dir before each build
  },
};
```

`output.clean: true` replaces the older `clean-webpack-plugin` for most cases.

## Mode: development vs production

Mode sets a group of built-in optimizations. Always set it explicitly.

```js
module.exports = {
  mode: 'production', // or 'development' | 'none'
};
```

| Mode | Behaviour |
| ---- | --------- |
| `development` | Fast rebuilds, readable output, `process.env.NODE_ENV = 'development'`, useful error messages. |
| `production` | Minification, tree shaking, deterministic IDs, `process.env.NODE_ENV = 'production'`. |
| `none` | No default optimizations. |

You can also pass mode on the CLI: `webpack --mode development`.

## Running webpack

Add npm scripts rather than calling the CLI directly:

```json
{
  "scripts": {
    "build": "webpack --mode production",
    "dev": "webpack --mode development --watch"
  }
}
```

```bash
npm run build   # one-off production build
npm run dev     # rebuild on file changes
```

Without a config file, webpack 5 uses defaults: entry `./src/index.js`, output
`./dist/main.js`.

## A Minimal Config

A complete, minimal, working setup:

```js
// webpack.config.js
const path = require('path');

module.exports = {
  mode: 'development',
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'main.js',
    clean: true,
  },
};
```

Project layout:

```text
my-app/
├── package.json
├── webpack.config.js
├── src/
│   └── index.js
└── dist/          (generated)
```

```js
// src/index.js
import { greet } from './greet.js';
console.log(greet('webpack'));
```

Build it:

```bash
npx webpack
```

## The Dependency Graph

webpack recursively follows `import` / `require()` statements from each entry,
building a graph of modules. Anything reachable is included; anything unreachable
can be dropped (tree shaking). Loaders transform files as they enter the graph,
and plugins act on the graph and emitted assets.

## Next Steps

- **configuration.md** — full `webpack.config.js` structure, `resolve`, devServer, source maps.
- **loaders.md** — transforming CSS, TypeScript, and assets.
- **plugins.md** — HTML generation, CSS extraction, environment injection.
- **optimization.md** — code splitting, caching, minification, Module Federation.
