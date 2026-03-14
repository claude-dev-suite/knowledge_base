# SPDX License Expressions

> Source: https://spdx.github.io/spdx-spec/v2.3/SPDX-license-expressions/

## Expression Syntax

SPDX expressions allow you to describe complex licensing scenarios.

### Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `AND` | Both licenses apply | `MIT AND Apache-2.0` |
| `OR` | Choice between licenses | `MIT OR GPL-3.0-only` |
| `WITH` | License with exception | `GPL-2.0-only WITH Classpath-exception-2.0` |
| `+` | "Or later" version | `GPL-2.0+` (deprecated, use `-or-later`) |

### Precedence

1. `+` (highest)
2. `WITH`
3. `AND`
4. `OR` (lowest)

Use parentheses for clarity:
```
(MIT AND BSD-3-Clause) OR Apache-2.0
```

## Common Expressions

### Single License

```
MIT
Apache-2.0
GPL-3.0-only
```

### Dual License (Choice)

```
MIT OR Apache-2.0
GPL-2.0-only OR MIT
(MIT OR Apache-2.0)
```

### Conjunctive (All Apply)

```
MIT AND BSD-3-Clause
Apache-2.0 AND MIT AND BSD-2-Clause
```

### With Exception

```
GPL-2.0-only WITH Classpath-exception-2.0
GPL-2.0-only WITH GCC-exception-3.1
Apache-2.0 WITH LLVM-exception
```

### Complex Expressions

```
(MIT OR Apache-2.0) AND BSD-3-Clause
GPL-3.0-only OR (MIT AND BSD-2-Clause)
(Apache-2.0 WITH LLVM-exception) OR MIT
```

## Common Exceptions

| Exception ID | Used With | Purpose |
|--------------|-----------|---------|
| `Classpath-exception-2.0` | GPL-2.0 | Java classpath linking |
| `GCC-exception-3.1` | GPL-3.0 | GCC runtime library |
| `LLVM-exception` | Apache-2.0 | LLVM compiler |
| `Bootloader-exception` | GPL | Bootloader linking |
| `Font-exception-2.0` | GPL | Font embedding |
| `GPL-CC-1.0` | GPL | Combine with CC licenses |

## Expression Examples in package.json

### Simple

```json
{ "license": "MIT" }
```

### Dual License

```json
{ "license": "(MIT OR Apache-2.0)" }
```

### Multiple Required

```json
{ "license": "MIT AND BSD-3-Clause" }
```

### With Exception

```json
{ "license": "GPL-2.0-only WITH Classpath-exception-2.0" }
```

### Complex

```json
{ "license": "(Apache-2.0 AND MIT) OR GPL-3.0-only" }
```

## Expression Validation

### Using spdx-expression-parse

```javascript
const parse = require('spdx-expression-parse');

// Valid expressions
parse('MIT');
parse('MIT OR Apache-2.0');
parse('(MIT AND BSD-2-Clause) OR Apache-2.0');

// Invalid (throws error)
parse('MIT or Apache'); // lowercase 'or'
parse('MIT/Apache');    // wrong operator
```

### Using spdx-satisfies

```javascript
const satisfies = require('spdx-satisfies');

// Does this expression allow MIT-only usage?
satisfies('MIT', 'MIT OR Apache-2.0');        // true
satisfies('GPL-3.0-only', 'MIT OR Apache-2.0'); // false

// Can I use this with my MIT project?
satisfies('MIT AND BSD-2-Clause', 'MIT');     // false (needs BSD too)
```

## License References

### External Documents

```
DocumentRef-external:LicenseRef-CustomLicense
```

### Custom License

```
LicenseRef-Proprietary
LicenseRef-CompanyInternal
```

In package.json:
```json
{
  "license": "SEE LICENSE IN LICENSE.md"
}
```

## Or-Later Versions

### Preferred Syntax

```
GPL-2.0-or-later    (v2 or any later)
GPL-3.0-or-later    (v3 or any later)
LGPL-2.1-or-later   (v2.1 or any later)
```

### Deprecated Syntax

```
GPL-2.0+            (use GPL-2.0-or-later)
GPL-3.0+            (use GPL-3.0-or-later)
```

## Expression Parsing

### AST Structure

```javascript
const parse = require('spdx-expression-parse');

parse('MIT OR Apache-2.0');
// Returns:
{
  "left": { "license": "MIT" },
  "conjunction": "or",
  "right": { "license": "Apache-2.0" }
}

parse('MIT AND BSD-2-Clause');
// Returns:
{
  "left": { "license": "MIT" },
  "conjunction": "and",
  "right": { "license": "BSD-2-Clause" }
}

parse('GPL-2.0-only WITH Classpath-exception-2.0');
// Returns:
{
  "license": "GPL-2.0-only",
  "exception": "Classpath-exception-2.0"
}
```

## Practical Examples

### Node.js Runtime

```
MIT AND ISC AND BSD-3-Clause
```

### Rust Crates (Common)

```
MIT OR Apache-2.0
```

### Java Libraries (GPL + Classpath)

```
GPL-2.0-only WITH Classpath-exception-2.0
```

### LLVM/Clang

```
Apache-2.0 WITH LLVM-exception
```

## Best Practices

1. **Always use SPDX identifiers** - Standardized and machine-readable
2. **Be explicit about versions** - Use `-only` or `-or-later`
3. **Parenthesize for clarity** - `(MIT OR Apache-2.0) AND BSD-3-Clause`
4. **Validate expressions** - Use spdx-expression-parse
5. **Document custom licenses** - Use LicenseRef- with explanation
