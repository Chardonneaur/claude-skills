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
9. [Marketplace Checklist](#marketplace-checklist)

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
vue/dist/
*.DS_Store
```

### Publishing workflow

1. Set version in `plugin.json` (start with `0.1.0`)
2. Add Matomo Marketplace webhook to your GitHub repo:
   - Repository Settings > Webhooks > Add webhook
   - URL: `https://plugins.matomo.org/webhooks/github`
3. Create a git tag and push:
   ```bash
   git tag 0.1.0
   git push --tags
   ```
4. Every new tag pushed to GitHub creates a new Marketplace version

### Version numbering

Follow semantic versioning: `MAJOR.MINOR.PATCH`
- MAJOR: Breaking changes
- MINOR: New features, backward compatible
- PATCH: Bug fixes

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
- [ ] `archive.exclude` in plugin.json excludes tests, node_modules, .git
- [ ] No phoning home without informed user consent
- [ ] All images/scripts loaded locally (no external CDN without disclosure)
