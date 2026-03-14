# ESLint Flat Config

> Source: https://eslint.org/docs/latest/use/configure/configuration-files

## Overview

ESLint 9.0.0 uses flat config (`eslint.config.js`) as the default configuration format. It replaces the legacy `.eslintrc.*` format.

## Basic Setup

```bash
npm install --save-dev eslint @eslint/js
```

```javascript
// eslint.config.js
import eslint from '@eslint/js';

export default [
  eslint.configs.recommended,
];
```

## Configuration Structure

Flat config is an array of configuration objects. Each object can specify:

```javascript
export default [
  {
    // Files to apply this config to
    files: ['**/*.js', '**/*.mjs'],

    // Files to ignore
    ignores: ['**/node_modules/**', '**/dist/**'],

    // Language options
    languageOptions: {
      ecmaVersion: 2024,
      sourceType: 'module',
      globals: {
        window: 'readonly',
        document: 'readonly'
      },
      parser: customParser,
      parserOptions: {
        ecmaFeatures: { jsx: true }
      }
    },

    // Linter options
    linterOptions: {
      noInlineConfig: false,
      reportUnusedDisableDirectives: 'warn'
    },

    // Processor
    processor: customProcessor,

    // Plugins
    plugins: {
      react: reactPlugin
    },

    // Rules
    rules: {
      'no-unused-vars': 'error',
      'react/jsx-uses-react': 'error'
    },

    // Settings shared with plugins
    settings: {
      react: { version: 'detect' }
    }
  }
];
```

## Global Ignores

```javascript
export default [
  {
    // Global ignores (no other keys)
    ignores: [
      '**/node_modules/**',
      '**/dist/**',
      '**/build/**',
      '**/*.min.js'
    ]
  },
  // Other configs...
];
```

## TypeScript Configuration

```javascript
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  {
    files: ['**/*.ts', '**/*.tsx'],
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      }
    },
    rules: {
      '@typescript-eslint/no-unused-vars': 'error'
    }
  }
);
```

## Multiple File Types

```javascript
export default [
  // JavaScript files
  {
    files: ['**/*.js'],
    rules: {
      'no-var': 'error'
    }
  },

  // TypeScript files
  {
    files: ['**/*.ts', '**/*.tsx'],
    plugins: { '@typescript-eslint': tseslint.plugin },
    rules: {
      '@typescript-eslint/explicit-function-return-type': 'warn'
    }
  },

  // Test files
  {
    files: ['**/*.test.js', '**/*.spec.js'],
    rules: {
      'no-unused-expressions': 'off'
    }
  }
];
```

## Using Plugins

```javascript
import reactPlugin from 'eslint-plugin-react';
import reactHooksPlugin from 'eslint-plugin-react-hooks';

export default [
  {
    files: ['**/*.jsx', '**/*.tsx'],
    plugins: {
      react: reactPlugin,
      'react-hooks': reactHooksPlugin
    },
    rules: {
      ...reactPlugin.configs.recommended.rules,
      ...reactHooksPlugin.configs.recommended.rules,
      'react/react-in-jsx-scope': 'off'
    },
    settings: {
      react: { version: 'detect' }
    }
  }
];
```

## Extending Configs

```javascript
import eslint from '@eslint/js';
import prettier from 'eslint-config-prettier';

export default [
  eslint.configs.recommended,
  prettier,  // Disable formatting rules that conflict with Prettier
  {
    rules: {
      // Your custom rules
    }
  }
];
```

## Migration from .eslintrc

```bash
# Automatic migration
npx @eslint/migrate-config .eslintrc.json
```

### Manual Migration

| Legacy | Flat Config |
|--------|-------------|
| `env` | `languageOptions.globals` |
| `extends` | Spread configs in array |
| `parserOptions` | `languageOptions.parserOptions` |
| `parser` | `languageOptions.parser` |
| `plugins` | Object with plugin instances |
| `root` | Not needed (auto-detected) |

## CommonJS Support

```javascript
// eslint.config.cjs
const eslint = require('@eslint/js');

module.exports = [
  eslint.configs.recommended,
];
```

## Best Practices

1. **Use ESM** - Prefer `eslint.config.mjs` for ESM syntax
2. **Order matters** - Later configs override earlier ones
3. **Be specific** - Use `files` to target specific file types
4. **Global ignores** - Put them first with no other keys
5. **Type checking** - Use `typescript-eslint` projectService for type-aware rules
