# webpack Configuration

> Official Documentation: https://webpack.js.org/configuration/

## Overview

webpack is configured through `webpack.config.js`, a CommonJS module that
exports a configuration object (or a function, or an array of objects). This
document covers the structure of that file and the most commonly used options in
webpack 5.

## Configuration Structure

```js
// webpack.config.js
const path = require('path');

module.exports = {
  mode: 'production',
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',
    clean: true,
  },
  module: {
    rules: [
      // loader rules — see loaders.md
    ],
  },
  plugins: [
    // plugins — see plugins.md
  ],
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx'],
  },
  devtool: 'source-map',
};
```

## Config as a Function

Export a function to read the CLI `env` and `argv` (including `mode`):

```js
module.exports = (env, argv) => {
  const isProduction = argv.mode === 'production';
  return {
    mode: argv.mode,
    devtool: isProduction ? 'source-map' : 'eval-cheap-module-source-map',
    // env.API_URL available when called with --env API_URL=...
  };
};
```

```bash
webpack --env API_URL=https://api.example.com --mode production
```

## mode

Sets built-in defaults for optimization, caching, and `process.env.NODE_ENV`.

```js
module.exports = { mode: 'development' }; // 'production' | 'development' | 'none'
```

## resolve

Controls how modules are found and resolved.

```js
module.exports = {
  resolve: {
    // Try these extensions so imports can omit them
    extensions: ['.js', '.jsx', '.ts', '.tsx', '.json'],

    // Path aliases
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@components': path.resolve(__dirname, 'src/components'),
    },

    // Extra directories to search (besides node_modules)
    modules: [path.resolve(__dirname, 'src'), 'node_modules'],
  },
};
```

Usage after aliasing: `import Button from '@components/Button';`

## devServer

Provided by `webpack-dev-server` (install separately). Serves the app in memory
with live reload and Hot Module Replacement.

```bash
npm install --save-dev webpack-dev-server
```

```js
module.exports = {
  devServer: {
    static: path.resolve(__dirname, 'public'),
    port: 8080,
    open: true,           // open browser on start
    hot: true,            // Hot Module Replacement
    compress: true,       // gzip
    historyApiFallback: true, // SPA client-side routing
    proxy: [
      { context: ['/api'], target: 'http://localhost:3000' },
    ],
  },
};
```

Run it:

```json
{
  "scripts": {
    "start": "webpack serve --mode development"
  }
}
```

Note: in webpack-dev-server v5, `proxy` is an **array** of `{ context, target }`
objects (the older object-map form is deprecated).

## Source Maps (devtool)

`devtool` controls source map generation, trading build speed for fidelity.

| Value | Use | Notes |
| ----- | --- | ----- |
| `eval-cheap-module-source-map` | development | Fast rebuilds, maps to original source lines. |
| `source-map` | production | Full, separate `.map` files; best for debugging prod. |
| `hidden-source-map` | production | Emits maps but no reference comment (for error reporters). |
| `false` | any | Disable source maps. |

```js
module.exports = {
  devtool: 'source-map',
};
```

## Persistent Caching (webpack 5)

Filesystem caching dramatically speeds up subsequent builds.

```js
module.exports = {
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename], // invalidate cache when config changes
    },
  },
};
```

## Multiple Configurations

Export an array to build several targets in one run (e.g. browser + node):

```js
module.exports = [
  {
    name: 'client',
    target: 'web',
    entry: './src/client.js',
    output: { filename: 'client.js', path: path.resolve(__dirname, 'dist') },
  },
  {
    name: 'server',
    target: 'node',
    entry: './src/server.js',
    output: { filename: 'server.js', path: path.resolve(__dirname, 'dist') },
  },
];
```

Build one entry from the array with `webpack --config-name client`.

### Splitting dev/prod configs

A common pattern is a shared base merged with mode-specific overrides using
`webpack-merge`:

```bash
npm install --save-dev webpack-merge
```

```js
// webpack.prod.js
const { merge } = require('webpack-merge');
const common = require('./webpack.common.js');

module.exports = merge(common, {
  mode: 'production',
  devtool: 'source-map',
});
```

```bash
webpack --config webpack.prod.js
```

## TypeScript Config Files

webpack can consume a `webpack.config.ts` if `typescript` and `ts-node` are
installed; webpack detects the `.ts` extension automatically.

```bash
npm install --save-dev typescript ts-node @types/node @types/webpack
```

```ts
// webpack.config.ts
import * as path from 'path';
import type { Configuration } from 'webpack';

const config: Configuration = {
  mode: 'production',
  entry: './src/index.ts',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
  },
};

export default config;
```

Note: `devServer` typings live in `webpack-dev-server`; import
`'webpack-dev-server'` to augment the `Configuration` type when needed.
