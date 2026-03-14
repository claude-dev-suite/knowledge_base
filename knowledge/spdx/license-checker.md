# License Checker Tools

> Source: https://github.com/davglass/license-checker

## license-checker (Node.js)

### Installation

```bash
npm install -g license-checker
# or
npm install --save-dev license-checker
```

### Basic Usage

```bash
# List all licenses
license-checker

# JSON output
license-checker --json

# CSV output
license-checker --csv

# Only production dependencies
license-checker --production
```

### Filtering

```bash
# Only show these licenses
license-checker --onlyAllow "MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC"

# Exclude packages
license-checker --excludePackages "package1;package2"

# Exclude private packages
license-checker --excludePrivatePackages

# Only direct dependencies
license-checker --direct
```

### Output Formats

```bash
# Summary only
license-checker --summary

# Include license text
license-checker --json --customPath customFormat.json

# Markdown table
license-checker --markdown

# Plain text report
license-checker --out licenses.txt
```

### CI Integration

```bash
# Fail if disallowed license found
license-checker --failOn "GPL-3.0-only;AGPL-3.0-only"

# Fail if unknown license
license-checker --failOn "UNKNOWN"

# Whitelist only
license-checker --onlyAllow "MIT;Apache-2.0;ISC;BSD-2-Clause;BSD-3-Clause" || exit 1
```

### Custom Format

```json
// customFormat.json
{
  "name": "",
  "version": "",
  "license": "",
  "repository": "",
  "licenseText": ""
}
```

```bash
license-checker --customPath customFormat.json --json > licenses.json
```

## npm-license-crawler

### Installation

```bash
npm install -g npm-license-crawler
```

### Usage

```bash
# Generate CSV
npm-license-crawler --csv licenses.csv

# JSON output
npm-license-crawler --json licenses.json

# Only production
npm-license-crawler --onlyDirectDependencies

# Exclude dev
npm-license-crawler --production
```

## legally (Alternative)

```bash
npm install -g legally

# Run in project directory
legally
```

## license-report

```bash
npm install -g license-report

# HTML report
license-report --output=html > report.html

# JSON report
license-report --output=json > report.json
```

## Programmatic Usage

```javascript
const checker = require('license-checker');

checker.init({
  start: process.cwd(),
  production: true,
  json: true
}, (err, packages) => {
  if (err) {
    console.error(err);
    return;
  }

  for (const [name, info] of Object.entries(packages)) {
    console.log(`${name}: ${info.licenses}`);
  }
});
```

### With Options

```javascript
checker.init({
  start: '/path/to/project',
  production: true,           // Only production deps
  development: false,
  direct: true,               // Only direct deps
  onlyAllow: 'MIT;Apache-2.0;ISC',
  excludePackages: 'package1;package2',
  customPath: {
    name: '',
    version: '',
    license: '',
    repository: ''
  }
}, callback);
```

### Validation Script

```javascript
// scripts/check-licenses.js
const checker = require('license-checker');

const ALLOWED = [
  'MIT',
  'Apache-2.0',
  'BSD-2-Clause',
  'BSD-3-Clause',
  'ISC',
  '0BSD',
  'CC0-1.0',
  'Unlicense'
];

const FORBIDDEN = [
  'GPL-2.0',
  'GPL-3.0',
  'AGPL-3.0',
  'GPL-2.0-only',
  'GPL-3.0-only',
  'AGPL-3.0-only'
];

checker.init({
  start: process.cwd(),
  production: true
}, (err, packages) => {
  if (err) {
    console.error(err);
    process.exit(1);
  }

  const violations = [];

  for (const [name, info] of Object.entries(packages)) {
    const license = info.licenses;

    if (FORBIDDEN.some(f => license.includes(f))) {
      violations.push({ name, license, reason: 'forbidden' });
    } else if (!ALLOWED.some(a => license.includes(a))) {
      violations.push({ name, license, reason: 'unknown' });
    }
  }

  if (violations.length > 0) {
    console.error('License violations found:');
    violations.forEach(v => {
      console.error(`  ${v.name}: ${v.license} (${v.reason})`);
    });
    process.exit(1);
  }

  console.log('All licenses OK');
});
```

## GitHub Actions

```yaml
name: License Check

on: [push, pull_request]

jobs:
  license-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci

      - name: Check licenses
        run: |
          npm install -g license-checker
          license-checker --production --onlyAllow "MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;0BSD;Unlicense;CC0-1.0"
```

## SBOM Generation

### Generate SPDX SBOM

```bash
# Using cdxgen
npx @cyclonedx/cdxgen -o sbom.json

# Using spdx-sbom-generator
npx spdx-sbom-generator
```

### CycloneDX Format

```bash
npx @cyclonedx/cdxgen -o bom.json --format json
npx @cyclonedx/cdxgen -o bom.xml --format xml
```

## Configuration File

### .licensecheckrc

```json
{
  "production": true,
  "onlyAllow": [
    "MIT",
    "Apache-2.0",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC"
  ],
  "excludePackages": [
    "internal-package"
  ]
}
```

### package.json Script

```json
{
  "scripts": {
    "license:check": "license-checker --production --onlyAllow 'MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC'",
    "license:report": "license-checker --production --json > licenses.json",
    "license:csv": "license-checker --production --csv > licenses.csv"
  }
}
```

## Best Practices

1. **Run in CI** - Block PRs with incompatible licenses
2. **Production only** - Dev dependencies usually don't affect distribution
3. **Maintain allowlist** - Explicit list of approved licenses
4. **Review unknowns** - Manually check packages with unknown licenses
5. **Generate reports** - Keep license documentation for compliance
6. **Update regularly** - New dependencies may introduce new licenses
