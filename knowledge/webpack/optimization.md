# webpack Optimization

> Official Documentation: https://webpack.js.org/configuration/optimization/

## Overview

webpack 5 ships with strong production defaults, but understanding the
optimization tools lets you tune bundle size, caching, and load performance. This
document covers code splitting, tree shaking, lazy loading, long-term caching,
minification, bundle analysis, Module Federation, and performance budgets.

Most options live under the `optimization` key; some (caching) depend on how you
name output files.

## Code Splitting (splitChunks)

`optimization.splitChunks` extracts shared and vendor code into separate chunks
so they can be cached independently and loaded in parallel.

```js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all', // split both async and initial chunks
    },
  },
};
```

Fine-grained control with cache groups:

```js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: -10,
        },
        common: {
          minChunks: 2, // modules used in 2+ chunks
          priority: -20,
          reuseExistingChunk: true,
        },
      },
    },
  },
};
```

Extract the webpack runtime into its own chunk so vendor hashes stay stable:

```js
optimization: {
  runtimeChunk: 'single',
}
```

## Tree Shaking

Tree shaking removes unused exports (dead code). It relies on ES module syntax
(`import` / `export`) and is enabled automatically in `production` mode.

Requirements:

1. Use ESM syntax (not CommonJS `require`).
2. Mark side-effect-free packages in `package.json`:

```json
{
  "sideEffects": false
}
```

Or list files that DO have side effects (e.g. global CSS):

```json
{
  "sideEffects": ["*.css", "./src/polyfills.js"]
}
```

`optimization.usedExports` (on by default in production) flags unused exports;
minification then strips them.

## Lazy Loading (Dynamic import)

Split code on demand with the dynamic `import()` syntax. webpack creates a
separate chunk loaded only when the code runs.

```js
button.addEventListener('click', async () => {
  const { renderChart } = await import(
    /* webpackChunkName: "chart" */ './chart.js'
  );
  renderChart();
});
```

The `webpackChunkName` magic comment names the generated chunk. This is the
foundation of route-based splitting in SPAs.

## Long-Term Caching (contenthash)

Include a content hash in output filenames so browsers only re-download files
that actually changed.

```js
module.exports = {
  output: {
    filename: '[name].[contenthash].js',
    chunkFilename: '[name].[contenthash].js',
    clean: true,
  },
  optimization: {
    moduleIds: 'deterministic', // stable module IDs across builds
    runtimeChunk: 'single',
  },
};
```

`moduleIds: 'deterministic'` (default in production) keeps hashes stable when
unrelated modules change. Pair with `HtmlWebpackPlugin` so the hashed filenames
are injected automatically.

## Minification (TerserPlugin)

In `production` mode webpack minifies JS automatically using
`terser-webpack-plugin` (bundled — no install needed). Customize it via
`optimization.minimizer`:

```js
const TerserPlugin = require('terser-webpack-plugin');

module.exports = {
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: { drop_console: true },
          format: { comments: false },
        },
        extractComments: false,
      }),
    ],
  },
};
```

To also minify extracted CSS, add `css-minimizer-webpack-plugin`. Note: adding a
custom `minimizer` array replaces the defaults, so re-include Terser (via `'...'`
to keep webpack's default minimizers):

```js
optimization: {
  minimizer: [
    '...', // keep the default JS minimizer (Terser)
    new CssMinimizerPlugin(),
  ],
}
```

## Bundle Analysis

Inspect what is in your bundles with `webpack-bundle-analyzer`.

```bash
npm install --save-dev webpack-bundle-analyzer
```

```js
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  plugins: [new BundleAnalyzerPlugin()],
};
```

Alternatively, generate stats and inspect them online:

```bash
webpack --profile --json > stats.json
```

## Module Federation (webpack 5)

Module Federation lets separately built and deployed applications share code at
runtime — the basis of micro-frontends. The `ModuleFederationPlugin` comes from
`webpack.container`.

```js
const { ModuleFederationPlugin } = require('webpack').container;
```

### Remote (exposes modules)

```js
module.exports = {
  output: { uniqueName: 'app1', publicPath: 'auto' },
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/Button',
      },
      shared: { react: { singleton: true } },
    }),
  ],
};
```

### Host (consumes remotes)

```js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        app1: 'app1@http://localhost:3001/remoteEntry.js',
      },
      shared: { react: { singleton: true } },
    }),
  ],
};
```

The host then imports a remote module dynamically:

```js
const Button = await import('app1/Button');
```

| Option | Meaning |
| ------ | ------- |
| `name` | Unique container name for this build. |
| `filename` | Name of the remote entry file. |
| `exposes` | Modules this container makes available. |
| `remotes` | Remote containers this build consumes. |
| `shared` | Dependencies shared across containers (avoid duplication). |

## Performance Budgets

The `performance` option warns when assets exceed size thresholds, helping catch
bundle bloat in CI.

```js
module.exports = {
  performance: {
    hints: 'warning',            // 'error' | 'warning' | false
    maxEntrypointSize: 250000,   // bytes (~244 KB)
    maxAssetSize: 250000,
  },
};
```

Set `hints: 'error'` to fail the build when budgets are exceeded.

## Summary Checklist

- `splitChunks: { chunks: 'all' }` and `runtimeChunk: 'single'` for caching.
- `[contenthash]` filenames + `moduleIds: 'deterministic'` for long-term caching.
- ESM + `sideEffects` for tree shaking.
- Dynamic `import()` for lazy loading.
- Terser (default) for JS minification; add CSS minimizer as needed.
- Analyze with `webpack-bundle-analyzer`; guard with `performance` budgets.
