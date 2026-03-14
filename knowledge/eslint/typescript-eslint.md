# TypeScript ESLint

> Source: https://typescript-eslint.io/getting-started/

## Installation

```bash
npm install --save-dev eslint @eslint/js typescript typescript-eslint
```

## Basic Configuration

```javascript
// eslint.config.mjs
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  eslint.configs.recommended,
  tseslint.configs.recommended,
);
```

## Configuration Presets

| Preset | Description |
|--------|-------------|
| `recommended` | Core rules without type checking |
| `recommendedTypeChecked` | Recommended + type-aware rules |
| `strict` | All recommended + stricter rules |
| `strictTypeChecked` | Strict + type-aware |
| `stylistic` | Style/convention rules |
| `stylisticTypeChecked` | Stylistic + type-aware |

### Type-Checked Configuration

```javascript
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },
);
```

### Strict Configuration

```javascript
export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
);
```

## Important Type-Checked Rules

### Async/Promise Safety

```javascript
rules: {
  // Require awaiting thenables
  '@typescript-eslint/await-thenable': 'error',

  // Disallow floating Promises
  '@typescript-eslint/no-floating-promises': 'error',

  // Disallow misused Promises
  '@typescript-eslint/no-misused-promises': 'error',

  // Require await in async functions
  '@typescript-eslint/require-await': 'error',
}
```

### Type Safety

```javascript
rules: {
  // No any
  '@typescript-eslint/no-explicit-any': 'error',

  // No unsafe operations with any
  '@typescript-eslint/no-unsafe-argument': 'error',
  '@typescript-eslint/no-unsafe-assignment': 'error',
  '@typescript-eslint/no-unsafe-call': 'error',
  '@typescript-eslint/no-unsafe-member-access': 'error',
  '@typescript-eslint/no-unsafe-return': 'error',

  // Prefer nullish coalescing
  '@typescript-eslint/prefer-nullish-coalescing': 'error',

  // Prefer optional chaining
  '@typescript-eslint/prefer-optional-chain': 'error',
}
```

### Best Practices

```javascript
rules: {
  // Consistent type imports
  '@typescript-eslint/consistent-type-imports': 'error',

  // Explicit return types
  '@typescript-eslint/explicit-function-return-type': 'warn',

  // No unused variables
  '@typescript-eslint/no-unused-vars': ['error', {
    argsIgnorePattern: '^_',
    varsIgnorePattern: '^_'
  }],

  // No unnecessary conditions
  '@typescript-eslint/no-unnecessary-condition': 'error',

  // Strict boolean expressions
  '@typescript-eslint/strict-boolean-expressions': 'error',
}
```

## Handling JavaScript Files

```javascript
export default tseslint.config(
  // TypeScript files
  {
    files: ['**/*.ts', '**/*.tsx'],
    extends: [
      eslint.configs.recommended,
      ...tseslint.configs.recommendedTypeChecked,
    ],
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },

  // JavaScript files (no type checking)
  {
    files: ['**/*.js', '**/*.mjs'],
    extends: [
      eslint.configs.recommended,
      tseslint.configs.disableTypeChecked,
    ],
  },
);
```

## Common Configurations

### React + TypeScript

```javascript
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactPlugin from 'eslint-plugin-react';
import reactHooksPlugin from 'eslint-plugin-react-hooks';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  {
    files: ['**/*.tsx'],
    plugins: {
      react: reactPlugin,
      'react-hooks': reactHooksPlugin,
    },
    rules: {
      ...reactPlugin.configs.recommended.rules,
      ...reactHooksPlugin.configs.recommended.rules,
      'react/react-in-jsx-scope': 'off',
    },
    settings: {
      react: { version: 'detect' },
    },
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
        ecmaFeatures: { jsx: true },
      },
    },
  },
);
```

### Node.js + TypeScript

```javascript
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import nodePlugin from 'eslint-plugin-n';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  nodePlugin.configs['flat/recommended'],
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },
);
```

## Ignoring Type Errors

```javascript
// @ts-expect-error - Known issue, will fix later
const result = problematicFunction();

// eslint-disable-next-line @typescript-eslint/no-explicit-any
function legacy(data: any): void { }
```

## Performance Tips

1. **Use projectService** - More efficient than `project` option
2. **Narrow file patterns** - Only lint files that need it
3. **Disable type-checking for JS** - Use `disableTypeChecked` config
4. **Cache results** - Use `--cache` flag
