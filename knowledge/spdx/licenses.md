# SPDX License List

> Source: https://spdx.org/licenses/

## What is SPDX?

SPDX (Software Package Data Exchange) is an open standard for communicating software bill of materials (SBOM), including license information.

## Common License Identifiers

### Permissive Licenses

| Identifier | Name | Key Points |
|------------|------|------------|
| `MIT` | MIT License | Very permissive, minimal restrictions |
| `Apache-2.0` | Apache License 2.0 | Patent grant, attribution required |
| `BSD-2-Clause` | BSD 2-Clause "Simplified" | Attribution only |
| `BSD-3-Clause` | BSD 3-Clause "New" | Attribution + no endorsement |
| `ISC` | ISC License | Simplified MIT |
| `0BSD` | BSD Zero Clause | Public domain equivalent |
| `Unlicense` | The Unlicense | Public domain dedication |

### Copyleft Licenses

| Identifier | Name | Key Points |
|------------|------|------------|
| `GPL-2.0-only` | GNU GPL v2.0 only | Strong copyleft |
| `GPL-2.0-or-later` | GNU GPL v2.0 or later | v2+ choice |
| `GPL-3.0-only` | GNU GPL v3.0 only | Strong copyleft + patent |
| `GPL-3.0-or-later` | GNU GPL v3.0 or later | v3+ choice |
| `LGPL-2.1-only` | GNU LGPL v2.1 only | Weak copyleft |
| `LGPL-3.0-only` | GNU LGPL v3.0 only | Weak copyleft |
| `AGPL-3.0-only` | GNU AGPL v3.0 | Network copyleft |
| `MPL-2.0` | Mozilla Public License 2.0 | File-level copyleft |

### Creative Commons

| Identifier | Name |
|------------|------|
| `CC0-1.0` | CC0 1.0 Universal |
| `CC-BY-4.0` | CC Attribution 4.0 |
| `CC-BY-SA-4.0` | CC Attribution ShareAlike 4.0 |
| `CC-BY-NC-4.0` | CC Attribution NonCommercial 4.0 |

### Other Common

| Identifier | Name |
|------------|------|
| `Artistic-2.0` | Artistic License 2.0 |
| `EPL-2.0` | Eclipse Public License 2.0 |
| `WTFPL` | Do What The F*ck You Want |
| `Zlib` | zlib License |
| `BlueOak-1.0.0` | Blue Oak Model License 1.0.0 |

## Using SPDX in package.json

```json
{
  "name": "my-package",
  "license": "MIT"
}
```

```json
{
  "license": "Apache-2.0"
}
```

```json
{
  "license": "(MIT OR Apache-2.0)"
}
```

## License File Headers

### SPDX-License-Identifier

```javascript
// SPDX-License-Identifier: MIT
```

```javascript
// SPDX-License-Identifier: Apache-2.0
```

```javascript
/*
 * SPDX-FileCopyrightText: 2024 My Company
 * SPDX-License-Identifier: MIT
 */
```

### Multi-file Project

```
// SPDX-License-Identifier: GPL-3.0-or-later
// SPDX-FileCopyrightText: 2024 Author Name <email@example.com>
```

## License Categories by Use Case

### For Libraries (Publish to npm)

| License | Pros | Cons |
|---------|------|------|
| `MIT` | Maximum adoption | No patent grant |
| `Apache-2.0` | Patent grant | Slightly more complex |
| `ISC` | Simple, npm default | Less recognized |

### For Applications

| License | Use Case |
|---------|----------|
| `MIT` / `Apache-2.0` | Open source SaaS |
| `AGPL-3.0-only` | Prevent proprietary forks |
| `Proprietary` | Commercial software |

### For Documentation

| License | Use Case |
|---------|----------|
| `CC-BY-4.0` | Attribution required |
| `CC0-1.0` | No restrictions |
| `MIT` | Code documentation |

## License Compatibility

### Can Combine With

```
MIT → Apache-2.0 → LGPL → GPL
     (more permissive → more restrictive)
```

### Common Combinations

| Your Code | Dependency | Result |
|-----------|------------|--------|
| MIT | MIT | MIT |
| MIT | Apache-2.0 | Apache-2.0 |
| MIT | GPL-3.0 | GPL-3.0 |
| Apache-2.0 | MIT | Apache-2.0 |
| GPL-3.0 | MIT | GPL-3.0 |
| GPL-2.0-only | GPL-3.0-only | Incompatible |

## Deprecated Identifiers

| Deprecated | Use Instead |
|------------|-------------|
| `GPL-2.0` | `GPL-2.0-only` or `GPL-2.0-or-later` |
| `GPL-3.0` | `GPL-3.0-only` or `GPL-3.0-or-later` |
| `LGPL-2.1` | `LGPL-2.1-only` or `LGPL-2.1-or-later` |
| `LGPL-3.0` | `LGPL-3.0-only` or `LGPL-3.0-or-later` |

## OSI Approved Licenses

Common OSI-approved identifiers:
- `MIT`, `Apache-2.0`, `BSD-2-Clause`, `BSD-3-Clause`
- `GPL-2.0-only`, `GPL-3.0-only`, `LGPL-2.1-only`, `LGPL-3.0-only`
- `MPL-2.0`, `EPL-2.0`, `ISC`
- `CDDL-1.0`, `Artistic-2.0`

## License Text Locations

Standard file names:
- `LICENSE`
- `LICENSE.txt`
- `LICENSE.md`
- `COPYING`
- `NOTICE` (for Apache-2.0)

## Dual Licensing

```json
{
  "license": "(MIT OR Apache-2.0)"
}
```

```json
{
  "license": "SEE LICENSE IN LICENSE.md"
}
```

For commercial + open source:
```
// SPDX-License-Identifier: MIT AND Proprietary
// See LICENSE.md for details
```
