# webpack Loaders

> Official Documentation: https://webpack.js.org/concepts/loaders/

## Overview

webpack only understands JavaScript and JSON out of the box. **Loaders** let it
process other file types — transpiling TypeScript, compiling CSS, or importing
images — by transforming the source into a valid module that gets added to the
dependency graph.

Loaders are configured under `module.rules`. Each rule matches files with `test`
and applies one or more loaders via `use`.

## Rule Structure: test / use

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/i,          // which files this rule applies to
        use: ['style-loader', 'css-loader'],
        exclude: /node_modules/,  // optional: skip these files
        include: path.resolve(__dirname, 'src'), // optional: only these
      },
    ],
  },
};
```

- `test` — RegExp matching filenames to transform.
- `use` — a loader string, an array of loaders, or objects with `options`.
- `exclude` / `include` — narrow which files the rule targets.

## Loader Chaining and Order

When `use` is an array, loaders execute **right to left** (bottom to top). The
last loader receives the raw file; its output is passed leftward.

```js
{
  test: /\.scss$/i,
  use: [
    'style-loader',   // 3. injects CSS into the DOM
    'css-loader',     // 2. resolves @import / url() into modules
    'sass-loader',    // 1. compiles Sass -> CSS (runs first)
  ],
}
```

## Loaders with Options

Use the object form to pass options to a loader:

```js
{
  test: /\.css$/i,
  use: [
    'style-loader',
    {
      loader: 'css-loader',
      options: {
        modules: true,     // enable CSS Modules
        importLoaders: 1,
      },
    },
  ],
}
```

## Common Loaders

| Loader | Purpose | Install |
| ------ | ------- | ------- |
| `babel-loader` | Transpile modern/JSX JS via Babel | `babel-loader @babel/core @babel/preset-env` |
| `ts-loader` | Compile TypeScript using `tsc` | `ts-loader typescript` |
| `css-loader` | Resolve `@import` and `url()` in CSS | `css-loader` |
| `style-loader` | Inject CSS into the DOM via `<style>` | `style-loader` |
| `sass-loader` | Compile Sass/SCSS to CSS | `sass-loader sass` |
| `postcss-loader` | Run PostCSS (autoprefixer, etc.) | `postcss-loader postcss` |

### babel-loader (JavaScript / JSX)

```bash
npm install --save-dev babel-loader @babel/core @babel/preset-env
```

```js
{
  test: /\.jsx?$/,
  exclude: /node_modules/,
  use: {
    loader: 'babel-loader',
    options: {
      presets: ['@babel/preset-env'],
    },
  },
}
```

### ts-loader (TypeScript)

```bash
npm install --save-dev ts-loader typescript
```

```js
{
  test: /\.tsx?$/,
  use: 'ts-loader',
  exclude: /node_modules/,
}
```

Pair with `resolve.extensions: ['.ts', '.tsx', '.js']` so imports omit
extensions. A `tsconfig.json` is required.

### CSS (css-loader + style-loader)

```bash
npm install --save-dev css-loader style-loader
```

```js
{
  test: /\.css$/i,
  use: ['style-loader', 'css-loader'],
}
```

For production, replace `style-loader` with `MiniCssExtractPlugin.loader` to emit
separate `.css` files — see **plugins.md**.

## Asset Modules (webpack 5)

webpack 5 handles images, fonts, and other files natively via **Asset Modules**,
replacing `file-loader`, `url-loader`, and `raw-loader`. Set `type` on the rule
instead of using a loader.

| `type` | Behaviour | Replaces |
| ------ | --------- | -------- |
| `asset/resource` | Emits a separate file, exports its URL | `file-loader` |
| `asset/inline` | Inlines as a Base64 data URI | `url-loader` |
| `asset/source` | Exports raw source as a string | `raw-loader` |
| `asset` | Auto: inline if small, else emit a file | `url-loader` with `limit` |

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.(png|jpe?g|gif|svg)$/i,
        type: 'asset/resource',
      },
      {
        test: /\.(woff2?|eot|ttf)$/i,
        type: 'asset/resource',
      },
      {
        test: /\.txt$/i,
        type: 'asset/source',
      },
    ],
  },
};
```

### Inline threshold and output name

For `type: 'asset'`, control the inline-vs-emit threshold (default 8 KB):

```js
{
  test: /\.svg$/i,
  type: 'asset',
  parser: {
    dataUrlCondition: {
      maxSize: 4 * 1024, // inline files under 4 KB, emit larger ones
    },
  },
  generator: {
    filename: 'static/[hash][ext][query]',
  },
}
```

A global emitted-asset name can be set via `output.assetModuleFilename`.

## Writing a Simple Loader

A loader is a function that receives source content and returns transformed
content. Keep it synchronous and pure when possible.

```js
// loaders/upper-loader.js
module.exports = function (source) {
  // `this` is the loader context; options via this.getOptions()
  const options = this.getOptions() || {};
  return options.uppercase ? source.toUpperCase() : source;
};
```

Reference a local loader by path with `resolveLoader` or directly:

```js
module.exports = {
  resolveLoader: {
    modules: ['node_modules', path.resolve(__dirname, 'loaders')],
  },
  module: {
    rules: [
      {
        test: /\.txt$/,
        use: {
          loader: 'upper-loader',
          options: { uppercase: true },
        },
      },
    ],
  },
};
```

For asynchronous work, call `const callback = this.async()` and invoke
`callback(err, content, sourceMap)` when done.
