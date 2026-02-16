# Marketplace Scaffolding Reference

Everything needed to package a Matomo plugin for the Marketplace.

## Table of Contents

1. [plugin.json Schema](#pluginjson-schema)
2. [README.md Format](#readmemd-format)
3. [CHANGELOG.md Format](#changelogmd-format)
4. [License](#license)
5. [Screenshots](#screenshots)
6. [Documentation](#documentation)
7. [Tests](#tests)
8. [GitHub & Publishing](#github--publishing)
9. [CI/CD with GitHub Actions](#cicd-with-github-actions)
10. [Vue Build Step](#vue-build-step)
11. [Marketplace Checklist](#marketplace-checklist)

---

## plugin.json Schema

### Minimal (lightweight)

```json
{
    "name": "PluginName",
    "description": "One-line description of the plugin.",
    "version": "1.0.0",
    "require": {
        "matomo": ">=5.0.0-b1,<6.0.0-b1",
        "php": ">=8.1.0"
    }
}
```

### Full (Marketplace-ready)

```json
{
    "name": "PluginName",
    "description": "A concise one-line description.",
    "version": "1.0.0",
    "theme": false,
    "require": {
        "matomo": ">=5.0.0-b1,<6.0.0-b1",
        "php": ">=8.1.0"
    },
    "authors": [
        {
            "name": "Author Name",
            "email": "author@example.com",
            "homepage": "https://example.com"
        }
    ],
    "homepage": "https://example.com/matomo-plugin",
    "license": "GPL v3+",
    "keywords": [
        "matomo",
        "analytics",
        "specific-keyword"
    ],
    "support": {
        "email": "support@example.com",
        "issues": "https://github.com/author/plugin-PluginName/issues",
        "forum": "https://forum.matomo.org",
        "docs": "https://example.com/docs",
        "source": "https://github.com/author/plugin-PluginName"
    },
    "archive": {
        "exclude": [
            "tests",
            "node_modules",
            ".github",
            ".gitignore",
            "vue/src"
        ]
    }
}
```

### Version constraints

| Target | Constraint |
|--------|-----------|
| Matomo 5 only | `"matomo": ">=5.0.0-b1,<6.0.0-b1"` |
| Matomo 4+5 | `"matomo": ">=4.0.0-b1,<6.0.0-b1"` |
| PHP 8.1+ | `"php": ">=8.1.0"` |
| PHP 7.2+ (Matomo 4) | `"php": ">=7.2.0"` |

### Plugin dependencies

```json
{
    "require": {
        "matomo": ">=5.0.0-b1,<6.0.0-b1",
        "piwik": {
            "TagManager": ">=5.0.0"
        }
    }
}
```

### Naming rules

- Plugin name must start with a letter
- Only letters and numbers allowed (no hyphens, underscores)
- Must NOT contain: "Matomo", "Piwik", "Analytics", "Core", "Plugin"

### License compatibility

Supported license values: `"GPL-3.0+"`, `"GPL-3.0"`, `"BSD AND GPL-3.0+"`,
`"GPL-2.0-only"`, `"GPL-2.0+"`, `"MIT"`. Contact Matomo for other licenses.

---

## README.md Format

Displayed on the Marketplace listing page.

```markdown
# Plugin Name

## Description

Clear, compelling description of what the plugin does.
This is the first thing visitors see on the Marketplace.

## Features

- Feature one with brief explanation
- Feature two with brief explanation

## Requirements

- Matomo 5.0 or later
- PHP 8.1 or later

## Installation

### Via Marketplace (recommended)
1. Go to Administration > Marketplace
2. Search for "Plugin Name"
3. Click Install

### Manual installation
1. Download the latest release
2. Extract to your Matomo `plugins/` directory
3. Activate: `./console plugin:activate PluginName`

## Configuration

Describe settings available under Administration > General Settings > Plugin Name.

## FAQ

**Q: Common question?**
A: Clear answer.

## Support

- Issues: https://github.com/author/plugin-PluginName/issues
- Forum: https://forum.matomo.org

## License

GPL v3 or later
```

---

## CHANGELOG.md Format

Newest version first:

```markdown
## Changelog

### 1.0.0

- Initial release
- Feature: Main feature description
- Feature: Secondary feature description
```

---

## License

Marketplace requires GPL v3+. Include a `LICENSE` file with the full GPL v3 text.

Every PHP file should include the GPL header (see `assets/gpl-v3-header.txt`).

---

## Screenshots

Place in `screenshots/` directory. File naming:

- `1.png`, `2.png`, ... — numbered screenshots shown in order
- `_cover.png` — 880×480px cover image for Marketplace listing
- Descriptive names also work: `dashboard_widget.png`

Only `.png`, `.jpg`, `.jpeg` allowed. File names must be alphanumeric with
underscores and dashes only.

Remind the user that screenshots must be actual screenshots from their Matomo
instance — they cannot be programmatically generated.

---

## Documentation

### docs/index.md

Developer/user documentation. Displayed in a "Documentation" tab on the
Marketplace page.

### docs/faq.md

FAQ section. Displayed in a "FAQ" tab.

---

## Tests

### Test structure

```
tests/
├── Unit/
│   └── PluginNameTest.php
├── Integration/
│   └── ApiTest.php
└── UI/
    └── PluginName_spec.js
```

### Running tests

```bash
# All tests for this plugin
./console tests:run PluginName

# Only unit tests
./console tests:run PluginName --testsuite unit

# Only integration tests
./console tests:run PluginName --testsuite integration
```

---

## GitHub & Publishing

### Repository naming convention

`plugin-{PluginName}` (e.g. `plugin-CustomDimensions`)

### .gitignore

```gitignore
.idea/
*.iml
node_modules/
vendor/
*.DS_Store
```

**Important:** Do NOT add `vue/dist/` to `.gitignore`. The built Vue files must be
committed so the Marketplace can include them in the downloadable archive.

### Publishing workflow

1. Set version in `plugin.json` (start with `0.1.0`)
2. Build Vue components if the plugin has them (see [Vue Build Step](#vue-build-step))
3. Add Matomo Marketplace webhook to your GitHub repo:
   - Repository Settings > Webhooks > Add webhook
   - URL: `https://plugins.matomo.org/webhooks/github`
4. Commit all files including `vue/dist/` build output
5. Create a git tag and push:
   ```bash
   git tag 0.1.0
   git push origin main --tags
   ```
6. Every new tag pushed to GitHub creates a new Marketplace version

### Version numbering

Follow semantic versioning: `MAJOR.MINOR.PATCH`
- MAJOR: Breaking changes
- MINOR: New features, backward compatible
- PATCH: Bug fixes

---

## CI/CD with GitHub Actions

Matomo provides an official GitHub Action for running plugin tests automatically.
See the [Matomo developer docs on GitHub Actions](https://developer.matomo.org/guides/tests-github)
for the latest details.

### Generating the workflow file

Matomo can auto-generate a workflow file based on your plugin's test structure:

```bash
./console generate:test-action --plugin="PluginName" --php-versions="8.1,8.3"
```

This creates `.github/workflows/matomo-tests.yml`. The command detects which test types
(PHP, Client, UI) exist in your plugin and configures jobs accordingly.

### Command options

| Option | Description |
|--------|-------------|
| `--plugin` | Plugin name |
| `--php-versions` | Comma-separated PHP versions (e.g. `"8.1,8.3"`) |
| `--dependent-plugins` | Other plugins needed (e.g. `"matomo-org/plugin-CustomVariables"`) |
| `--force-php-tests` | Force PHP test job even if not auto-detected |
| `--force-ui-tests` | Force UI test job even if not auto-detected |
| `--force-client-tests` | Force client/JS test job even if not auto-detected |
| `--setup-script` | Custom bash script to run before tests |
| `--enable-redis` | Provision Redis for tests |
| `--protect-artifacts` | Require login to view test artifacts (premium plugins) |

### Manual workflow file

If the `generate:test-action` command is not available, create the workflow manually:

```yaml
# .github/workflows/matomo-tests.yml
name: Matomo Tests

on:
  pull_request:
    types: [opened, synchronize]
  push:
    branches:
      - main
      - develop

permissions:
  contents: read

jobs:
  PluginTests:
    runs-on: ubuntu-22.04
    strategy:
      fail-fast: false
      matrix:
        php: ['8.1', '8.3']
        target: ['minimum_required_matomo', 'maximum_supported_matomo']
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - name: Run tests
        uses: matomo-org/github-action-tests@main
        with:
          plugin-name: 'PluginName'
          php-version: ${{ matrix.php }}
          test-type: 'PluginTests'
          matomo-test-branch: ${{ matrix.target }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Workflow configuration reference

**test-type values:**

| Value | Description |
|-------|-------------|
| `PluginTests` | Run all PHP tests (unit + integration) for the plugin |
| `UnitTests` | Run only unit tests |
| `UI` | Run UI screenshot tests |
| `Client` | Run JavaScript/Vue client tests |
| `JS` | Run legacy JavaScript tests |

**matomo-test-branch values:**

| Value | Description |
|-------|-------------|
| `minimum_required_matomo` | Auto-detects minimum version from `plugin.json` |
| `maximum_supported_matomo` | Auto-detects maximum supported version |
| `5.x-dev` | Latest Matomo 5.x development branch |
| `5.2.1` | Specific Matomo version tag |

### Full workflow with multiple jobs

```yaml
name: Matomo Tests

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  PHPTests:
    runs-on: ubuntu-22.04
    strategy:
      fail-fast: false
      matrix:
        php: ['8.1', '8.3']
        target: ['minimum_required_matomo', 'maximum_supported_matomo']
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - name: Run tests
        uses: matomo-org/github-action-tests@main
        with:
          plugin-name: 'PluginName'
          php-version: ${{ matrix.php }}
          test-type: 'PluginTests'
          matomo-test-branch: ${{ matrix.target }}
          github-token: ${{ secrets.GITHUB_TOKEN }}

  ClientTests:
    runs-on: ubuntu-22.04
    strategy:
      fail-fast: false
      matrix:
        target: ['minimum_required_matomo', 'maximum_supported_matomo']
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - name: Run tests
        uses: matomo-org/github-action-tests@main
        with:
          plugin-name: 'PluginName'
          php-version: '8.1'
          test-type: 'Client'
          matomo-test-branch: ${{ matrix.target }}
          node-version: '16'
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Dependent plugins

If your plugin depends on other plugins not bundled with Matomo:

```yaml
      - name: Run tests
        uses: matomo-org/github-action-tests@main
        with:
          plugin-name: 'PluginName'
          php-version: ${{ matrix.php }}
          test-type: 'PluginTests'
          matomo-test-branch: ${{ matrix.target }}
          dependent-plugins: 'matomo-org/plugin-CustomVariables,author/plugin-OtherPlugin'
          github-token: ${{ secrets.TESTS_ACCESS_TOKEN || secrets.GITHUB_TOKEN }}
```

For private plugin dependencies, add a `TESTS_ACCESS_TOKEN` repository secret with
read access to those repositories.

### Keeping the workflow updated

The `generate:test-action` command may change over time. If your CI builds start
failing, re-run the command to regenerate an updated workflow file.

---

## Vue Build Step

Plugins with Vue 3 components **must** build their frontend code before publishing.
The built files in `vue/dist/` must be committed to the repository and included in
tagged releases.

### Building Vue components

```bash
# Build for production (from Matomo root directory)
./console vue:build PluginName

# Build with watch mode for development
./console vue:build PluginName --watch

# Clear webpack cache if builds behave unexpectedly
./console vue:build PluginName --clear-webpack-cache
```

### Requirements

- Node.js 16.0.0+ and npm 7.0.0+
- Run `npm install` from Matomo root first (installs shared build dependencies)

### What the build produces

```
plugins/PluginName/vue/
├── src/                    ← Source files (excluded from Marketplace archive)
│   ├── index.ts
│   ├── MyComponent/
│   │   └── MyComponent.vue
│   └── MyStore/
│       └── MyStore.store.ts
└── dist/                   ← Built output (MUST be committed)
    ├── PluginName.umd.js
    └── PluginName.umd.min.js
```

### .gitignore considerations

The default `.gitignore` should NOT exclude `vue/dist/`. The Marketplace needs the
built files. Only exclude source files via `plugin.json`:

```json
{
    "archive": {
        "exclude": [
            "vue/src",
            "node_modules",
            "tests",
            ".github"
        ]
    }
}
```

### Release workflow with Vue build

```bash
# 1. Make your code changes
# 2. Build Vue components
./console vue:build PluginName

# 3. Bump version in plugin.json
# 4. Update CHANGELOG.md

# 5. Commit everything including vue/dist/
git add plugin.json CHANGELOG.md vue/dist/
git commit -m "Release 1.1.0"

# 6. Tag and push
git tag 1.1.0
git push origin main --tags
```

### Automating Vue builds in CI

You can add a build check to your GitHub Actions workflow to verify the Vue build
is up to date:

```yaml
  VueBuildCheck:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '16'

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'

      - name: Checkout Matomo
        run: |
          git clone --depth 1 --branch 5.x-dev https://github.com/matomo-org/matomo.git matomo
          cp -r $GITHUB_WORKSPACE matomo/plugins/PluginName

      - name: Install dependencies
        working-directory: matomo
        run: |
          composer install --prefer-dist --no-progress
          npm install

      - name: Build Vue
        working-directory: matomo
        run: ./console vue:build PluginName

      - name: Check for uncommitted changes
        working-directory: matomo/plugins/PluginName
        run: |
          if [[ -n $(git diff --name-only vue/dist/) ]]; then
            echo "ERROR: vue/dist/ is out of date. Run './console vue:build PluginName' and commit."
            git diff --stat vue/dist/
            exit 1
          fi
```

---

## Marketplace Checklist

Before publishing:

- [ ] `plugin.json` has: name, description, version, require, license, authors
- [ ] `README.md` with description and installation instructions
- [ ] `CHANGELOG.md` with at least one version entry
- [ ] `LICENSE` file present (GPL v3+)
- [ ] All PHP files have GPL license header
- [ ] Plugin name is PascalCase, no special characters
- [ ] Plugin activates without errors on fresh Matomo install
- [ ] No hardcoded URLs, paths, or credentials
- [ ] All user-facing strings use translation keys
- [ ] Settings use the Settings framework
- [ ] API methods have proper access checks
- [ ] No `var_dump()`, `print_r()`, or debug code
- [ ] `archive.exclude` in plugin.json excludes tests, node_modules, .git, vue/src
- [ ] Vue components built (`./console vue:build PluginName`) and `vue/dist/` committed
- [ ] GitHub Actions workflow in `.github/workflows/matomo-tests.yml` (if tests exist)
- [ ] No phoning home without informed user consent
- [ ] All images/scripts loaded locally (no external CDN without disclosure)
