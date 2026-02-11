---
name: matomo-plugin-creator
description: >
  Create production-ready Matomo plugins through a structured 6-phase process: requirements, architecture,
  modular implementation, QA, security audit, and delivery. Covers tracking (dimensions, events, Tag Manager),
  reporting (reports, widgets, RecordBuilder archiving), and admin plugins (settings, menus, controllers).
  Generates PHP classes, Twig templates, Vue 3 components following Matomo's architecture. Supports lightweight
  and Marketplace-ready packaging. Use this skill whenever the user mentions Matomo plugins, custom dimensions,
  widgets, reports, tracking, or the Matomo Marketplace. Also trigger for "extend Matomo", "Matomo integration",
  "Matomo API", or any reference to Matomo's plugin hooks, events, or archiver.
---

# Matomo Plugin Creator

Create production-ready Matomo plugins through a structured, phase-gated development process.

**Core principle:** Plan first, build modularly, validate security, deliver clean code.
Use only Matomo Core methods, utility classes, and existing implementations.
Never add external libraries/dependencies unless absolutely necessary and explicitly confirmed by the user.

---

## Phase 0: Requirements Gathering (ALWAYS first)

Before planning or writing any code, ask the user ALL of the following questions.
Do not proceed until answers are collected. Group questions logically and present
them in a clear, concise way.

### Technical prerequisites
- **PHP version** (minimum) — default: 8.1+ for Matomo 5.x
- **Matomo version** — default: 5.x (options: 5.x only / 4+5 cross-compatible)
- **Database** — MySQL or MariaDB (affects SQL syntax edge cases)

### Plugin scope
- **What should the plugin do?** (free description from the user)
- **Which plugin types are involved?** Tracking / Reporting / Admin / Combination
- **Does the plugin need its own database tables?**
- **Does it need scheduled tasks (cron)?**
- **Tag Manager integration needed?**

### Packaging and distribution
- **Lightweight or Marketplace-ready?**
- **plugin.json data:** Author name, website, email, license (default: GPL v3+)
- **GitHub repository:** Should the plugin be prepared for GitHub commits?
- **Multilingual:** Just en.json, or additional languages (e.g. de.json)?

### Optional live validation
- **Is a Matomo API token available?** (URL + token_auth)
  - If yes: validate against live instance (version, active plugins, DB schema)
  - If no: generate offline only

### Existing context
- **Is there existing code or a plugin to build upon?**
- **Special performance requirements?** (e.g. high-traffic installations)

---

## Phase 1: Architecture Planning

**No code is written until the user confirms the architecture.**

Create an architecture document (Markdown) containing:

1. **Module plan** — Which PHP classes will be created and where they live
2. **DB schema** — Tables/columns to be created (DDL draft), if applicable
3. **Event hooks** — Which Matomo events will be subscribed to and why
4. **API endpoints** — Method name, parameters, return type, access level
5. **Dependencies** — Which Core classes are used (NO external libs without justification)
6. **Access control** — Which roles need access to which functions

Present the architecture document to the user for review. Wait for confirmation
or refinement before proceeding to Phase 2.

Reference files to read during this phase:
- `references/plugin-types.md` — for class patterns and hook references
- `references/archiver-patterns.md` — for Archiver/RecordBuilder design
- `references/permissions-reference.md` — for access control design

---

## Phase 2: Modular Implementation

Each module is an independent unit, built sequentially. Between modules, give the
user a brief status report with the opportunity to make corrections.

### Module order and dependencies

| Module | Description | Depends on |
|--------|-------------|------------|
| **Core** | Main plugin class, plugin.json, translations, GPL headers | — |
| **Tracking** | Dimensions (Visit/Action), tracker.js, Tag Manager extensions | Core |
| **Archiver** | RecordBuilder or classic Archiver, aggregation logic | Core, Tracking |
| **API** | API.php with all endpoints, access checks | Core, Archiver |
| **Reporting** | Report classes, Widgets, Visualizations | Core, API |
| **Admin UI** | Settings, Controller, Menu, Twig templates, Vue 3 components | Core |
| **Scheduled Tasks** | Tasks.php for cron jobs | Core, API |

Not every plugin needs every module. Skip modules that aren't relevant to the plugin scope.

### Implementation rules

These rules are non-negotiable:

1. **Core methods first** — Always use Matomo Core classes:
   - `Common::getRequestVar()` for all request parameters (NEVER $_GET/$_POST)
   - `Db::query($sql, $params)` with prepared statements (NEVER string interpolation in SQL)
   - `Common::prefixTable()` for all table names
   - `Piwik::translate()` for all user-facing strings
   - `Piwik::check*` methods for all permission checks

2. **No external dependencies** — No Composer packages without explicit justification
   and user confirmation. Matomo ships with jQuery, Vue 3, PHP-DI, Monolog, etc.

3. **GPL v3+ headers** — Every PHP file gets the standard GPL header from `assets/gpl-v3-header.txt`

4. **PSR-4 autoloading** — File names match class names exactly.
   Namespace: `Piwik\Plugins\{PluginName}` (always `Piwik`, not `Matomo`)

5. **Translations** — Every user-facing string uses a translation key in `lang/en.json`.
   PHP: `Piwik::translate('{PluginName}_KeyName')`
   Twig: `{{ '{PluginName}_KeyName'|translate }}`

6. **Archiver pattern** — Prefer RecordBuilder (Matomo 5.x) over classic Archiver.
   Read `references/archiver-patterns.md` before implementing any archiving logic.

7. **Admin UI** — Support both Twig templates and Vue 3 components.
   Read `references/plugin-types.md` for patterns of both approaches.

---

## Phase 3: QA & Testing

After ALL modules are implemented, run through these checks:

### Code quality
- All PHP files pass syntax check (`php -l`)
- Namespace consistency (PSR-4)
- All classes correctly extend Matomo base classes
- GPL header present in every PHP file
- No `var_dump()`, `print_r()`, `error_log()`, or debug code
- All translation keys in code exist in `lang/en.json`

### Performance review
Read `references/archiver-patterns.md` for performance guidance, then verify:
- DB queries in Archiver use proper indexes and LIMIT
- Archiver respects `datatable_archiving_maximum_rows_standard` config
- No N+1 query patterns
- LogAggregator methods preferred over raw SQL
- `live_query_max_execution_time` respected for live/raw data features
- Tracker plugins limit request frequency per page view

---

## Phase 4: Security Audit

**This phase is mandatory.** Read `references/security-checklist.md` for the complete
checklist with code examples, then verify every point.

Summary of checks:

1. **CSRF protection** — checkTokenInUrl(), Nonce system, token_auth via POST only
2. **SQL injection** — Prepared statements everywhere, int casting, prefixTable()
3. **XSS prevention** — Common::getRequestVar(), proper Twig escaping, no unsafe v-html
4. **Access control** — Every API method and controller action has permission checks
5. **Dependency audit** — No external packages, no eval/exec/system/passthru

### Mandatory user notice

After the security audit, always display this notice:

> **⚠️ Important notice:** This plugin was generated with AI assistance.
> Before deploying to production, we strongly recommend:
> - A code review by an experienced PHP/Matomo developer
> - A security check against Matomo's Security Best Practices
> - Testing in a staging environment
> - For plugins with database changes: backup before activation

---

## Phase 5: Packaging & Delivery

### Lightweight delivery
- Plugin folder with all files
- Installation instructions (copy to `plugins/`, activate via console or UI)

### Marketplace-ready delivery
Read `references/marketplace-scaffolding.md` for full details, then generate:
- `plugin.json` (complete metadata)
- `README.md` (Marketplace format)
- `CHANGELOG.md`
- `LICENSE` (GPL v3+)
- `screenshots/` (placeholder note for user to add)
- `docs/index.md` + `docs/faq.md`
- `.gitignore` for IDE files, node_modules, vendor/

### Optional GitHub setup
If the user requested GitHub integration:
- Generate `.gitignore`, `README.md`
- Provide instructions for Marketplace webhook setup
- Explain tag-based versioning (`git tag 1.0.0 && git push --tags`)

---

## Optional: Live API Validation

If the user provides a Matomo API token (URL + token_auth), validate:

1. **Matomo version**: `GET ?module=API&method=API.getMatomoVersion&token_auth=...&format=json`
   → Confirm version targeting is correct
2. **Active plugins**: `GET ?module=API&method=API.getSettings&token_auth=...&format=json`
   → Detect potential dependency conflicts
3. **Plugin activation**: After generation, test if plugin activates cleanly

Implementation: Simple HTTP GET requests to Matomo's Reporting API. No SDK needed.
Always use HTTPS. Never log or display the token_auth back to the user.

---

## Plugin Types Quick Reference

| Type | Key classes | Reference |
|------|-----------|-----------|
| Tracking | Columns/*, tracker.js, TrackingDimensions | `references/plugin-types.md` § Tracking |
| Reporting | Reports/*, API.php, Widgets/*, Archiver.php | `references/plugin-types.md` § Reporting |
| Admin | Settings.php, Controller.php, Menu.php | `references/plugin-types.md` § Admin |
| Archiver | Archiver.php, RecordBuilders/ | `references/archiver-patterns.md` |
| Security | (all types) | `references/security-checklist.md` |
| Permissions | (all types) | `references/permissions-reference.md` |
| Marketplace | plugin.json, README.md, tests/ | `references/marketplace-scaffolding.md` |

---

## Directory Structure Templates

### Lightweight
```
PluginName/
├── plugin.json
├── PluginName.php
├── API.php
├── Columns/
├── Reports/
├── Settings.php
├── templates/
└── lang/
    └── en.json
```

### Marketplace-Ready
```
PluginName/
├── plugin.json
├── README.md
├── CHANGELOG.md
├── LICENSE
├── PluginName.php
├── API.php
├── Archiver.php
├── RecordBuilders/
├── Controller.php
├── Menu.php
├── Settings.php
├── Tasks.php
├── Columns/
├── Reports/
├── Widgets/
├── templates/
├── vue/src/
├── lang/en.json
├── config/config.php
├── screenshots/
├── tests/Unit/
├── tests/Integration/
└── docs/
```

---

## What This Skill Does NOT Do

- Deploy to live servers
- Publish to the Marketplace automatically
- Write UI screenshot tests (but points to Matomo's UI testing framework)
- Guarantee bug-free code — user is always advised to do code review

---

## Companion Files

- **Claude.md** — Project instructions: code standards, phase rules, behavioral constraints.
  Read this first to understand the non-negotiable rules for this skill.
- **memory.md** — Decision log: tracks user choices, reference sources, per-plugin context.
  Update this whenever the user makes an architectural decision.
- **README.md** — Human-facing documentation, skill structure, disclaimer.

## Recommended Companion Skills

| Skill | When to suggest |
|-------|----------------|
| **doc-coauthoring** | User wants structured plugin docs, user guides, or technical specs |
| **frontend-design** | User needs polished Vue 3 admin UIs or widget visual design |
| **skill-creator** | User wants to iterate on this skill's performance or add features |
