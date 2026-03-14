# Biome

> Official Documentation: https://biomejs.dev/

## Overview

Biome is a fast, modern toolchain for web projects that combines formatting, linting, and import organization into a single tool. Written in Rust, it offers exceptional performance (10-100x faster than ESLint + Prettier) while providing compatibility with existing ecosystems.

Key features:
- **Unified tooling**: Formatter + linter + import organizer in one package
- **Zero configuration**: Works out of the box with sensible defaults
- **Performance**: Near-instant execution even on large codebases
- **TypeScript-first**: Native TypeScript support without additional plugins
- **Editor integration**: First-class VS Code support with real-time feedback

## Installation and Initialization

### Package Installation

```bash
# npm
npm install --save-dev @biomejs/biome

# yarn
yarn add --dev @biomejs/biome

# pnpm
pnpm add --save-dev @biomejs/biome

# bun
bun add --dev @biomejs/biome
```

### Project Initialization

```bash
# Create biome.json with recommended defaults
npx @biomejs/biome init

# Or with specific options
npx @biomejs/biome init --jsonc  # Create biome.jsonc (allows comments)
```

### Global Installation (Optional)

```bash
npm install -g @biomejs/biome
```

## Configuration Structure (biome.json)

### Complete Configuration Example

```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true,
    "defaultBranch": "main"
  },
  "files": {
    "maxSize": 1048576,
    "ignore": ["node_modules", "dist", "build", "coverage", "*.gen.ts"],
    "include": ["src/**/*.ts", "src/**/*.tsx"]
  },
  "organizeImports": {
    "enabled": true
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf",
    "formatWithErrors": false,
    "attributePosition": "auto"
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "jsxQuoteStyle": "double",
      "semicolons": "always",
      "trailingCommas": "all",
      "quoteProperties": "asNeeded",
      "arrowParentheses": "always",
      "bracketSpacing": true,
      "bracketSameLine": false
    },
    "parser": {
      "unsafeParameterDecoratorsEnabled": true
    }
  },
  "json": {
    "formatter": {
      "trailingCommas": "none"
    },
    "parser": {
      "allowComments": true,
      "allowTrailingCommas": false
    }
  },
  "css": {
    "formatter": {
      "enabled": true,
      "indentStyle": "space",
      "indentWidth": 2,
      "lineWidth": 100,
      "quoteStyle": "double"
    },
    "linter": {
      "enabled": true
    }
  },
  "overrides": [
    {
      "include": ["*.test.ts", "*.spec.ts"],
      "linter": {
        "rules": {
          "suspicious": {
            "noExplicitAny": "off"
          }
        }
      }
    }
  ]
}
```

### Schema Versions

Always use the schema for IDE autocompletion:

```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json"
}
```

## Formatter Configuration

### Global Formatter Options

| Option | Values | Default | Description |
|--------|--------|---------|-------------|
| `indentStyle` | `"tab"`, `"space"` | `"tab"` | Indentation character |
| `indentWidth` | 1-24 | 2 | Number of spaces per indent |
| `lineWidth` | 1-320 | 80 | Maximum line length |
| `lineEnding` | `"lf"`, `"crlf"`, `"cr"` | `"lf"` | Line ending style |
| `formatWithErrors` | boolean | false | Format files with syntax errors |
| `attributePosition` | `"auto"`, `"multiline"` | `"auto"` | HTML/JSX attribute positioning |

### JavaScript/TypeScript Formatter Options

```json
{
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "jsxQuoteStyle": "double",
      "semicolons": "always",
      "trailingCommas": "all",
      "quoteProperties": "asNeeded",
      "arrowParentheses": "always",
      "bracketSpacing": true,
      "bracketSameLine": false
    }
  }
}
```

| Option | Values | Default | Description |
|--------|--------|---------|-------------|
| `quoteStyle` | `"single"`, `"double"` | `"double"` | String quote style |
| `jsxQuoteStyle` | `"single"`, `"double"` | `"double"` | JSX attribute quote style |
| `semicolons` | `"always"`, `"asNeeded"` | `"always"` | Semicolon insertion |
| `trailingCommas` | `"all"`, `"es5"`, `"none"` | `"all"` | Trailing comma style |
| `quoteProperties` | `"asNeeded"`, `"preserve"` | `"asNeeded"` | Object property quoting |
| `arrowParentheses` | `"always"`, `"asNeeded"` | `"always"` | Arrow function parentheses |
| `bracketSpacing` | boolean | true | Spaces in object literals |
| `bracketSameLine` | boolean | false | JSX closing bracket position |

### JSON Formatter Options

```json
{
  "json": {
    "formatter": {
      "trailingCommas": "none"
    },
    "parser": {
      "allowComments": true,
      "allowTrailingCommas": true
    }
  }
}
```

## Linter Configuration

### Rule Categories

Biome organizes lint rules into categories:

| Category | Purpose |
|----------|---------|
| `recommended` | Enable all recommended rules (default) |
| `all` | Enable all available rules |
| `complexity` | Code complexity and simplification |
| `correctness` | Likely bugs and errors |
| `performance` | Performance optimizations |
| `security` | Security vulnerabilities |
| `style` | Code style and conventions |
| `suspicious` | Suspicious patterns that may be bugs |
| `nursery` | Experimental rules (unstable) |
| `a11y` | Accessibility rules for JSX |

### Rule Severity Levels

- `"error"` - Fails the check, blocks CI
- `"warn"` - Reports issue but doesn't fail
- `"off"` - Disables the rule
- `"info"` - Informational only

### Basic Linter Configuration

```json
{
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "correctness": {
        "noUnusedVariables": "error",
        "noUnusedImports": "error"
      },
      "suspicious": {
        "noExplicitAny": "warn",
        "noConsoleLog": "warn"
      },
      "complexity": {
        "noForEach": "off"
      },
      "style": {
        "useConst": "error",
        "noNonNullAssertion": "warn"
      }
    }
  }
}
```

### Complexity Rules

Rules for reducing code complexity:

```json
{
  "complexity": {
    "noBannedTypes": "error",
    "noExcessiveCognitiveComplexity": "warn",
    "noExcessiveNestedTestSuites": "warn",
    "noForEach": "off",
    "noMultipleSpacesInRegularExpressionLiterals": "error",
    "noStaticOnlyClass": "error",
    "noThisInStatic": "error",
    "noUselessCatch": "error",
    "noUselessConstructor": "error",
    "noUselessEmptyExport": "error",
    "noUselessFragments": "error",
    "noUselessLabel": "error",
    "noUselessLoneBlockStatements": "error",
    "noUselessRename": "error",
    "noUselessSwitchCase": "error",
    "noUselessTernary": "error",
    "noUselessThisAlias": "error",
    "noUselessTypeConstraint": "error",
    "noVoid": "off",
    "noWith": "error",
    "useArrowFunction": "error",
    "useFlatMap": "error",
    "useLiteralKeys": "error",
    "useOptionalChain": "error",
    "useRegexLiterals": "error",
    "useSimpleNumberKeys": "error",
    "useSimplifiedLogicExpression": "error"
  }
}
```

### Correctness Rules

Rules for catching bugs:

```json
{
  "correctness": {
    "noChildrenProp": "error",
    "noConstAssign": "error",
    "noConstantCondition": "error",
    "noConstructorReturn": "error",
    "noEmptyCharacterClassInRegex": "error",
    "noEmptyPattern": "error",
    "noGlobalObjectCalls": "error",
    "noInnerDeclarations": "error",
    "noInvalidConstructorSuper": "error",
    "noInvalidNewBuiltin": "error",
    "noInvalidUseBeforeDeclaration": "error",
    "noNewSymbol": "error",
    "noNodejsModules": "off",
    "noNonoctalDecimalEscape": "error",
    "noPrecisionLoss": "error",
    "noRenderReturnValue": "error",
    "noSelfAssign": "error",
    "noSetterReturn": "error",
    "noStringCaseMismatch": "error",
    "noSwitchDeclarations": "error",
    "noUndeclaredVariables": "error",
    "noUnnecessaryContinue": "error",
    "noUnreachable": "error",
    "noUnreachableSuper": "error",
    "noUnsafeFinally": "error",
    "noUnsafeOptionalChaining": "error",
    "noUnusedImports": "warn",
    "noUnusedLabels": "error",
    "noUnusedPrivateClassMembers": "warn",
    "noUnusedVariables": "warn",
    "noVoidElementsWithChildren": "error",
    "noVoidTypeReturn": "error",
    "useArrayLiterals": "error",
    "useExhaustiveDependencies": "warn",
    "useHookAtTopLevel": "error",
    "useIsNan": "error",
    "useJsxKeyInIterable": "error",
    "useValidForDirection": "error",
    "useYield": "error"
  }
}
```

### Style Rules

Rules for consistent code style:

```json
{
  "style": {
    "noArguments": "error",
    "noCommaOperator": "error",
    "noDefaultExport": "off",
    "noImplicitBoolean": "off",
    "noInferrableTypes": "error",
    "noNamespace": "error",
    "noNamespaceImport": "off",
    "noNegationElse": "off",
    "noNonNullAssertion": "warn",
    "noParameterAssign": "error",
    "noParameterProperties": "off",
    "noRestrictedGlobals": "off",
    "noShoutyConstants": "off",
    "noUnusedTemplateLiteral": "error",
    "noUselessElse": "error",
    "noVar": "error",
    "useBlockStatements": "off",
    "useCollapsedElseIf": "error",
    "useConst": "error",
    "useDefaultParameterLast": "error",
    "useEnumInitializers": "error",
    "useExponentiationOperator": "error",
    "useExportType": "error",
    "useFilenamingConvention": "off",
    "useForOf": "error",
    "useFragmentSyntax": "error",
    "useImportType": "error",
    "useLiteralEnumMembers": "error",
    "useNamingConvention": "off",
    "useNodeAssertStrict": "error",
    "useNodejsImportProtocol": "off",
    "useNumberNamespace": "error",
    "useNumericLiterals": "error",
    "useSelfClosingElements": "error",
    "useShorthandArrayType": "error",
    "useShorthandAssign": "error",
    "useShorthandFunctionType": "error",
    "useSingleCaseStatement": "off",
    "useSingleVarDeclarator": "error",
    "useTemplate": "error"
  }
}
```

### Suspicious Rules

Rules for detecting potentially problematic code:

```json
{
  "suspicious": {
    "noApproximativeNumericConstant": "error",
    "noArrayIndexKey": "error",
    "noAssignInExpressions": "error",
    "noAsyncPromiseExecutor": "error",
    "noCatchAssign": "error",
    "noClassAssign": "error",
    "noCommentText": "error",
    "noCompareNegZero": "error",
    "noConfusingLabels": "error",
    "noConfusingVoidType": "error",
    "noConsoleLog": "warn",
    "noConstEnum": "error",
    "noControlCharactersInRegex": "error",
    "noDebugger": "error",
    "noDoubleEquals": "error",
    "noDuplicateCase": "error",
    "noDuplicateClassMembers": "error",
    "noDuplicateJsxProps": "error",
    "noDuplicateObjectKeys": "error",
    "noDuplicateParameters": "error",
    "noEmptyBlockStatements": "error",
    "noEmptyInterface": "error",
    "noExplicitAny": "warn",
    "noExportsInTest": "error",
    "noExtraNonNullAssertion": "error",
    "noFallthroughSwitchClause": "error",
    "noFocusedTests": "error",
    "noFunctionAssign": "error",
    "noGlobalAssign": "error",
    "noGlobalIsFinite": "error",
    "noGlobalIsNan": "error",
    "noImplicitAnyLet": "error",
    "noImportAssign": "error",
    "noLabelVar": "error",
    "noMisleadingCharacterClass": "error",
    "noMisleadingInstantiation": "error",
    "noMisrefactoredShorthandAssign": "error",
    "noPrototypeBuiltins": "error",
    "noRedeclare": "error",
    "noRedundantUseStrict": "error",
    "noSelfCompare": "error",
    "noShadowRestrictedNames": "error",
    "noSkippedTests": "warn",
    "noSparseArray": "error",
    "noThenProperty": "error",
    "noUnsafeDeclarationMerging": "error",
    "noUnsafeNegation": "error",
    "useAwait": "off",
    "useDefaultSwitchClauseLast": "error",
    "useGetterReturn": "error",
    "useIsArray": "error",
    "useNamespaceKeyword": "error",
    "useValidTypeof": "error"
  }
}
```

### Nursery Rules (Experimental)

```json
{
  "nursery": {
    "noBarrelFile": "off",
    "noConsole": "off",
    "noDoneCallback": "warn",
    "noDuplicateElseIf": "error",
    "noEvolvingTypes": "off",
    "noExportedImports": "off",
    "noMisplacedAssertion": "warn",
    "noReactSpecificProps": "off",
    "noRestrictedImports": "off",
    "noUndeclaredDependencies": "off",
    "noUselessStringConcat": "error",
    "noUselessUndefinedInitialization": "error",
    "useConsistentBuiltinInstantiation": "error",
    "useDateNow": "error",
    "useDefaultSwitchClause": "off",
    "useErrorMessage": "warn",
    "useExplicitLengthCheck": "error",
    "useImportExtensions": "off",
    "useImportRestrictions": "off",
    "useSortedClasses": "off",
    "useThrowNewError": "error",
    "useThrowOnlyError": "error",
    "useTopLevelRegex": "off"
  }
}
```

## Important Lint Rules with Examples

### noUnusedVariables / noUnusedImports

```typescript
// Bad - triggers error
import { unused } from 'module';  // noUnusedImports
const x = 5;  // noUnusedVariables (x is never used)

// Good
import { used } from 'module';
const x = 5;
console.log(x, used);
```

### noExplicitAny

```typescript
// Bad - triggers warning
function process(data: any): any {
  return data;
}

// Good
function process<T>(data: T): T {
  return data;
}
```

### useConst

```typescript
// Bad
let name = 'John';  // never reassigned

// Good
const name = 'John';
```

### noConsoleLog

```typescript
// Bad - triggers warning in production code
console.log('debug info');

// Good - use proper logging
logger.debug('debug info');
```

### useOptionalChain

```typescript
// Bad
const value = obj && obj.prop && obj.prop.nested;

// Good
const value = obj?.prop?.nested;
```

### noDoubleEquals

```typescript
// Bad
if (value == null) { }

// Good
if (value === null || value === undefined) { }
// Or
if (value == null) { }  // Exception: == null is often allowed
```

### useExhaustiveDependencies (React)

```typescript
// Bad - missing dependency
useEffect(() => {
  fetchData(userId);
}, []);  // userId should be in deps

// Good
useEffect(() => {
  fetchData(userId);
}, [userId]);
```

### noArrayIndexKey (React)

```typescript
// Bad
{items.map((item, index) => (
  <Item key={index} data={item} />
))}

// Good
{items.map((item) => (
  <Item key={item.id} data={item} />
))}
```

## Import Organization

### Configuration

```json
{
  "organizeImports": {
    "enabled": true
  }
}
```

### Import Order (Default)

Biome automatically organizes imports in this order:

1. Side-effect imports (`import './styles.css'`)
2. Node.js built-in modules (`import fs from 'node:fs'`)
3. External packages (`import React from 'react'`)
4. Internal/aliased imports (`import { util } from '@/utils'`)
5. Relative imports (`import { Component } from './Component'`)

### Example

```typescript
// Before
import { useState } from 'react';
import './styles.css';
import { helper } from '../utils';
import axios from 'axios';
import { Component } from './Component';
import path from 'node:path';

// After (organized)
import './styles.css';

import path from 'node:path';

import axios from 'axios';
import { useState } from 'react';

import { helper } from '../utils';
import { Component } from './Component';
```

## CLI Commands

### biome format

Format source files:

```bash
# Format and show diff
biome format .

# Format and write changes
biome format --write .

# Format specific files
biome format --write src/index.ts src/utils.ts

# Format with specific config
biome format --config-path ./config/biome.json --write .

# Check if files are formatted (CI)
biome format --check .
```

### biome lint

Run linter:

```bash
# Lint all files
biome lint .

# Lint with auto-fix
biome lint --write .

# Lint specific directory
biome lint src/

# Apply only safe fixes
biome lint --write --safe .

# Apply unsafe fixes too
biome lint --write --unsafe .
```

### biome check

Run all checks (format + lint + organize imports):

```bash
# Check all
biome check .

# Check and fix all issues
biome check --write .

# Check specific files
biome check src/**/*.ts

# Only run specific checks
biome check --formatter-enabled=true --linter-enabled=false .
```

### biome ci

CI-optimized check (exits with error on issues):

```bash
# Standard CI check
biome ci .

# With specific reporter
biome ci --reporter=github .

# Check changed files only (with git)
biome ci --changed --since=main .
```

### Additional Commands

```bash
# Initialize project
biome init

# Migrate from ESLint/Prettier
biome migrate eslint --write
biome migrate prettier --write

# Explain a rule
biome explain noExplicitAny

# Show version
biome --version

# Update schema
biome migrate --write
```

## VS Code Integration

### Installation

1. Install the "Biome" extension from VS Code marketplace
2. Extension ID: `biomejs.biome`

### Settings (settings.json)

```json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },
  "[javascript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[typescript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[json]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[jsonc]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "biome.enabled": true,
  "biome.lspBin": "./node_modules/@biomejs/biome/bin/biome"
}
```

### Workspace Settings (.vscode/settings.json)

```json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports.biome": "explicit"
  }
}
```

### Recommended Extensions (.vscode/extensions.json)

```json
{
  "recommendations": ["biomejs.biome"]
}
```

## Git Hooks with lint-staged

### Installation

```bash
npm install --save-dev husky lint-staged
npx husky init
```

### Configuration (package.json)

```json
{
  "lint-staged": {
    "*.{js,ts,jsx,tsx,json,css}": ["biome check --write --no-errors-on-unmatched"]
  }
}
```

### Alternative: lint-staged.config.js

```javascript
export default {
  '*.{js,ts,jsx,tsx}': ['biome check --write --no-errors-on-unmatched'],
  '*.{json,jsonc}': ['biome format --write --no-errors-on-unmatched'],
  '*.css': ['biome format --write --no-errors-on-unmatched'],
};
```

### Husky Hook (.husky/pre-commit)

```bash
#!/bin/sh
npx lint-staged
```

### Using lefthook (Alternative)

```yaml
# lefthook.yml
pre-commit:
  commands:
    biome:
      glob: "*.{js,ts,jsx,tsx,json,css}"
      run: npx biome check --write --staged --no-errors-on-unmatched {staged_files}
      stage_fixed: true
```

## Migration from ESLint/Prettier

### Automatic Migration

```bash
# Migrate ESLint configuration
npx @biomejs/biome migrate eslint --write

# Migrate Prettier configuration
npx @biomejs/biome migrate prettier --write

# Migrate both
npx @biomejs/biome migrate eslint --write && npx @biomejs/biome migrate prettier --write
```

### Manual Migration Steps

1. **Install Biome**
   ```bash
   npm install --save-dev @biomejs/biome
   npx @biomejs/biome init
   ```

2. **Update package.json scripts**
   ```json
   {
     "scripts": {
       "lint": "biome check .",
       "lint:fix": "biome check --write .",
       "format": "biome format --write ."
     }
   }
   ```

3. **Remove ESLint/Prettier packages**
   ```bash
   npm uninstall eslint prettier eslint-config-prettier eslint-plugin-*
   ```

4. **Remove old config files**
   ```bash
   rm .eslintrc* .prettierrc* .eslintignore .prettierignore
   ```

5. **Update VS Code settings**
   ```json
   {
     "editor.defaultFormatter": "biomejs.biome"
   }
   ```

### ESLint Rule Equivalents

| ESLint Rule | Biome Rule |
|-------------|------------|
| `no-unused-vars` | `correctness/noUnusedVariables` |
| `no-console` | `suspicious/noConsoleLog` |
| `eqeqeq` | `suspicious/noDoubleEquals` |
| `prefer-const` | `style/useConst` |
| `no-var` | `style/noVar` |
| `@typescript-eslint/no-explicit-any` | `suspicious/noExplicitAny` |
| `react-hooks/exhaustive-deps` | `correctness/useExhaustiveDependencies` |
| `react/jsx-key` | `correctness/useJsxKeyInIterable` |

## Ignore Patterns

### Global Ignores (biome.json)

```json
{
  "files": {
    "ignore": [
      "node_modules",
      "dist",
      "build",
      ".next",
      "coverage",
      "*.min.js",
      "*.gen.ts",
      "**/*.d.ts"
    ]
  }
}
```

### Inline Ignores

```typescript
// Ignore next line
// biome-ignore lint/suspicious/noExplicitAny: needed for external API
const data: any = externalApi.response;

// Ignore with explanation (recommended)
// biome-ignore lint/complexity/noForEach: performance not critical here
items.forEach(process);

// Ignore formatting
// biome-ignore format: keep manual alignment
const matrix = [
  [1,  2,  3],
  [4,  5,  6],
  [7,  8,  9],
];

// Ignore multiple rules
// biome-ignore lint/suspicious/noExplicitAny lint/style/noNonNullAssertion: legacy code
const value: any = obj!.prop;
```

### Range Ignores

```typescript
// biome-ignore-start lint/suspicious/noConsoleLog: debugging section
console.log('step 1');
console.log('step 2');
console.log('step 3');
// biome-ignore-end lint/suspicious/noConsoleLog
```

### Per-file Overrides

```json
{
  "overrides": [
    {
      "include": ["*.test.ts", "*.spec.ts", "**/__tests__/**"],
      "linter": {
        "rules": {
          "suspicious": {
            "noExplicitAny": "off",
            "noFocusedTests": "error"
          }
        }
      }
    },
    {
      "include": ["scripts/**"],
      "linter": {
        "rules": {
          "suspicious": {
            "noConsoleLog": "off"
          }
        }
      }
    }
  ]
}
```

## Best Practices

### 1. Start with Recommended Rules

```json
{
  "linter": {
    "rules": {
      "recommended": true
    }
  }
}
```

### 2. Use Consistent Configuration Across Projects

Create a shared configuration:

```json
// biome.shared.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "formatter": {
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "semicolons": "always"
    }
  }
}
```

Extend in projects:

```json
{
  "extends": ["./biome.shared.json"]
}
```

### 3. Integrate with CI/CD

```yaml
# GitHub Actions
- name: Lint and Format Check
  run: npx biome ci .
```

### 4. Use --write for Local Development

```json
{
  "scripts": {
    "lint": "biome check .",
    "lint:fix": "biome check --write ."
  }
}
```

### 5. Configure Editor for Format on Save

Ensure VS Code formats on save for immediate feedback.

### 6. Use Specific Ignore Comments

Always provide a reason:

```typescript
// biome-ignore lint/suspicious/noExplicitAny: API response type unknown
```

### 7. Regularly Update Biome

```bash
npm update @biomejs/biome
npx biome migrate --write  # Update config schema
```

### 8. Use Overrides for Test Files

Test files often need relaxed rules:

```json
{
  "overrides": [
    {
      "include": ["*.test.ts"],
      "linter": {
        "rules": {
          "suspicious": { "noExplicitAny": "off" }
        }
      }
    }
  ]
}
```

## Common Pitfalls

### 1. Forgetting to Run with --write

```bash
# This only checks, doesn't fix
biome check .

# This actually fixes issues
biome check --write .
```

### 2. Not Updating the Schema Version

Keep schema updated for IDE support:

```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json"
}
```

### 3. Ignoring VCS Integration

Enable VCS for better ignore handling:

```json
{
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  }
}
```

### 4. Conflicts with Other Formatters

Disable Prettier extension when using Biome:

```json
{
  "prettier.enable": false,
  "editor.defaultFormatter": "biomejs.biome"
}
```

### 5. Missing Files in lint-staged

Use `--no-errors-on-unmatched` to prevent errors:

```json
{
  "lint-staged": {
    "*.ts": ["biome check --write --no-errors-on-unmatched"]
  }
}
```

### 6. Not Using --staged Flag

For pre-commit hooks, only check staged files:

```bash
biome check --write --staged
```

### 7. Overly Strict Initial Configuration

Start permissive, then tighten:

```json
{
  "linter": {
    "rules": {
      "recommended": true,
      "suspicious": {
        "noExplicitAny": "warn"  // Start with warn, move to error later
      }
    }
  }
}
```

### 8. Ignoring noUnusedImports in Large Codebases

This rule is essential for clean code:

```json
{
  "correctness": {
    "noUnusedImports": "error"
  }
}
```

### 9. Not Using Type-Aware Rules

Biome has type-aware rules that require TypeScript:

```json
{
  "javascript": {
    "jsxRuntime": "reactClassic"
  }
}
```

### 10. Forgetting CSS Support

CSS linting and formatting is available:

```json
{
  "css": {
    "formatter": { "enabled": true },
    "linter": { "enabled": true }
  }
}
```
