# webpack Plugins

> Official Documentation: https://webpack.js.org/concepts/plugins/

## Overview

Where loaders transform individual files, **plugins** operate on the whole build.
They tap into webpack's lifecycle hooks to perform tasks loaders cannot: emitting
HTML, extracting CSS into files, injecting environment variables, copying static
assets, and shaping optimization.

Plugins are added as instances to the `plugins` array:

```js
const HtmlWebpackPlugin = require('html-webpack-plugin');
const webpack = require('webpack');

module.exports = {
  plugins: [
    new HtmlWebpackPlugin({ template: './src/index.html' }),
    new webpack.DefinePlugin({ /* ... */ }),
  ],
};
```

Some plugins ship inside webpack itself (e.g. `webpack.DefinePlugin`); others are
installed as separate packages.

## Common Plugins

| Plugin | Package | Purpose |
| ------ | ------- | ------- |
| `HtmlWebpackPlugin` | `html-webpack-plugin` | Generate an HTML file and inject bundles. |
| `MiniCssExtractPlugin` | `mini-css-extract-plugin` | Extract CSS into separate files. |
| `DefinePlugin` | built-in (`webpack`) | Replace globals at compile time. |
| `CopyWebpackPlugin` | `copy-webpack-plugin` | Copy static files to the output dir. |
| `ProvidePlugin` | built-in (`webpack`) | Auto-import modules without explicit imports. |

## HtmlWebpackPlugin

Generates an `index.html` and automatically injects `<script>` / `<link>` tags
for emitted bundles — essential once filenames include content hashes.

```bash
npm install --save-dev html-webpack-plugin
```

```js
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html', // use your own HTML as the template
      filename: 'index.html',
      title: 'My App',
      minify: true, // enabled by default in production mode
    }),
  ],
};
```

## MiniCssExtractPlugin

Extracts CSS into standalone `.css` files (instead of injecting via
`style-loader`). Use its loader in place of `style-loader`, and add the plugin.

```bash
npm install --save-dev mini-css-extract-plugin
```

```js
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/i,
        use: [MiniCssExtractPlugin.loader, 'css-loader'],
      },
    ],
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css',
    }),
  ],
};
```

Common pattern: use `style-loader` in development (faster, supports HMR) and
`MiniCssExtractPlugin.loader` in production.

## DefinePlugin

Replaces expressions with constants at build time — useful for feature flags and
environment values. Values must be stringified (they are inlined as raw code).

```js
const webpack = require('webpack');

module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify('production'),
      __API_URL__: JSON.stringify('https://api.example.com'),
      __DEV__: JSON.stringify(false),
    }),
  ],
};
```

Note: setting `mode` already defines `process.env.NODE_ENV`; use DefinePlugin for
your own custom constants.

## CopyWebpackPlugin

Copies files/folders (favicons, robots.txt, static assets) into the output
directory without importing them.

```bash
npm install --save-dev copy-webpack-plugin
```

```js
const CopyPlugin = require('copy-webpack-plugin');

module.exports = {
  plugins: [
    new CopyPlugin({
      patterns: [
        { from: 'public', to: '.' },
        { from: 'src/assets/favicon.ico', to: 'favicon.ico' },
      ],
    }),
  ],
};
```

## Plugin Ordering

Plugins in the `plugins` array are applied in order. For most plugins the order
is irrelevant because they attach to specific hooks. However, when one plugin
consumes the output of another, order (and correct loader placement) matters —
for example, `MiniCssExtractPlugin.loader` must be present in the rule *before*
`MiniCssExtractPlugin` can extract the CSS it produces.

## Writing a Basic Plugin

A plugin is a class (or object) with an `apply(compiler)` method. Inside, tap
webpack's hooks (powered by the **tapable** library) to run custom logic.

```js
// plugins/FileListPlugin.js
class FileListPlugin {
  constructor(options = {}) {
    this.outputFile = options.outputFile || 'FILELIST.md';
  }

  apply(compiler) {
    const pluginName = FileListPlugin.name;
    const { webpack } = compiler;
    const { Compilation } = webpack;
    const { RawSource } = webpack.sources;

    compiler.hooks.thisCompilation.tap(pluginName, (compilation) => {
      compilation.hooks.processAssets.tap(
        {
          name: pluginName,
          stage: Compilation.PROCESS_ASSETS_STAGE_SUMMARIZE,
        },
        (assets) => {
          const list = Object.keys(assets)
            .map((name) => `- ${name}`)
            .join('\n');
          compilation.emitAsset(
            this.outputFile,
            new RawSource(`# Assets\n\n${list}\n`)
          );
        }
      );
    });
  }
}

module.exports = FileListPlugin;
```

Register it like any plugin:

```js
const FileListPlugin = require('./plugins/FileListPlugin');

module.exports = {
  plugins: [new FileListPlugin({ outputFile: 'assets.md' })],
};
```

### Hook Types (tapable)

| Tap method | Hook kind |
| ---------- | --------- |
| `.tap()` | Synchronous hooks. |
| `.tapAsync(name, (arg, callback) => ...)` | Async hooks using a callback. |
| `.tapPromise(name, async (arg) => ...)` | Async hooks returning a Promise. |

Key compiler hooks include `compile`, `thisCompilation`, `compilation`, `emit`,
and `done`. In webpack 5, prefer `compilation.hooks.processAssets` (with a
`stage`) over the older `emit` hook for asset manipulation.
