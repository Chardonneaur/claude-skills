# Security Checklist Reference

Complete security checklist for Matomo plugin development. Based on Matomo's official
Security Best Practices guide (developer.matomo.org/guides/security-in-piwik).

Every check includes a code example showing the CORRECT and INCORRECT approach.

## Table of Contents

1. [CSRF Protection](#1-csrf-protection)
2. [SQL Injection Prevention](#2-sql-injection-prevention)
3. [XSS Prevention](#3-xss-prevention)
4. [Access Control Audit](#4-access-control-audit)
5. [Dependency & Code Audit](#5-dependency--code-audit)
6. [Performance Security](#6-performance-security)
7. [Object Injection Prevention](#7-object-injection-prevention)
8. [File Upload Security](#8-file-upload-security)
9. [Cookie Security](#9-cookie-security)
10. [Error Handling Security](#10-error-handling-security)
11. [Logging Security](#11-logging-security)
12. [Final Checklist](#12-final-checklist)

---

## 1. CSRF Protection

CSRF attacks make a Matomo user perform actions unwillingly via crafted links.

### Controller actions that change data

Every controller method that modifies settings, data, or state MUST call `checkTokenInUrl()`:

```php
// ✅ CORRECT
public function save(): void
{
    Piwik::checkUserHasSuperUserAccess();
    $this->checkTokenInUrl();
    // ... perform the action
}

// ❌ WRONG — missing CSRF check
public function save(): void
{
    Piwik::checkUserHasSuperUserAccess();
    // ... action without CSRF protection
}
```

### Form submissions with Nonce

For HTML forms, use Matomo's Nonce system:

```php
// Controller: generate nonce for form
public function index(): string
{
    $view = new View('@MyPlugin/index');
    $view->nonce = Nonce::getNonce('MyPlugin.save');
    return $view->render();
}

// Controller: validate nonce on submit
public function save(): void
{
    $this->checkTokenInUrl();
    $nonce = Common::getRequestVar('nonce', '', 'string');
    Nonce::checkNonce('MyPlugin.save', $nonce);
    // ... proceed with save
}
```

```twig
{# Template: include nonce in form #}
<form method="post" action="{{ linkTo({'action': 'save'}) }}">
    <input type="hidden" name="nonce" value="{{ nonce }}">
    {# ... form fields ... #}
</form>
```

### API methods

API methods that change data must check permissions (which implicitly validates token_auth):

```php
// ✅ CORRECT
public function updateSetting(int $idSite, string $value): void
{
    Piwik::checkUserHasAdminAccess($idSite);
    // ... update logic
}
```

### token_auth handling

- **NEVER** include token_auth in URLs (browser caches/logs would expose it)
- **ALWAYS** send token_auth via POST requests from JavaScript
- **NEVER** log or display token_auth values

---

## 2. SQL Injection Prevention

### Always use prepared statements

```php
// ✅ CORRECT — prepared statement with bound parameters
$idSite = Common::getRequestVar('idSite', 0, 'int');
$sql = "SELECT * FROM " . Common::prefixTable('site') . " WHERE idsite = ?";
$rows = Db::query($sql, [$idSite]);

// ❌ WRONG — string interpolation in SQL
$idSite = Common::getRequestVar('idSite');
$sql = "SELECT * FROM " . Common::prefixTable('site') . " WHERE idsite = " . $idSite;
$rows = Db::query($sql);
```

### Integer parameters — cast explicitly

```php
// ✅ CORRECT
$idSite = (int) Common::getRequestVar('idSite', 0, 'int');

// ❌ WRONG — trusting user input without casting
$idSite = Common::getRequestVar('idSite');
```

### Table names — always use prefixTable()

```php
// ✅ CORRECT
$table = Common::prefixTable('log_visit');

// ❌ WRONG — hardcoded table name (breaks multi-instance setups)
$table = 'matomo_log_visit';
```

### LogAggregator — prefer over raw SQL

For archiving, use `$this->getLogAggregator()` methods instead of writing raw SQL:

```php
// ✅ CORRECT — uses LogAggregator (handles segments, periods, bindings)
$logAggregator = $this->getLogAggregator();
$query = $logAggregator->queryVisitsByDimension(['log_visit.example_dimension']);

// ❌ WRONG — raw SQL that doesn't handle segments or prepared statements properly
$sql = "SELECT example_dimension, COUNT(*) FROM " . Common::prefixTable('log_visit');
```

---

## 3. XSS Prevention

### Request parameters — always sanitize

```php
// ✅ CORRECT — sanitized automatically
$value = Common::getRequestVar('userInput', '', 'string');

// ❌ WRONG — direct superglobal access
$value = $_GET['userInput'];
```

If you need the original unsanitized value (e.g. for JSON output):
```php
$sanitized = Common::getRequestVar('data', '', 'string');
$original = Common::unsanitizeInputValue($sanitized);
```

### Twig templates — correct escaping strategy

```twig
{# ✅ CORRECT — default escaping (HTML) #}
<p>{{ variableName }}</p>

{# ✅ CORRECT — attribute escaping #}
<div title="{{ variableName|e('html_attr') }}">

{# ✅ CORRECT — JavaScript context #}
<script>var x = {{ variableName|e('js') }};</script>

{# ❌ DANGEROUS — raw output without escaping #}
<p>{{ variableName|raw }}</p>
```

If `|raw` is unavoidable, consider `|rawSafeDecoded` which is more secure.

### JavaScript — text vs html

```javascript
// ✅ CORRECT — inserts as text, not HTML
$('#someLabel').text(ajaxData.label);

// ✅ CORRECT — explicit escaping
var safe = piwikHelper.escape(userInput);

// ❌ DANGEROUS — inserts raw HTML from user input
$('#someLabel').html(ajaxData.label);
```

### Vue 3 — avoid v-html

```vue
<!-- ✅ CORRECT — text interpolation (auto-escaped) -->
<span>{{ userValue }}</span>

<!-- ❌ DANGEROUS — raw HTML rendering -->
<div v-html="userValue"></div>
```

If `v-html` is unavoidable, sanitize with DOMPurify first.

### Content Security Policy

Matomo sets a default CSP since v4.6.0. If your plugin needs additional sources:

```php
// In Controller — per-request CSP modification
$this->securityPolicy->addPolicy('image-src', 'self');

// In config/config.php — global CSP modification
return [
    \Piwik\View\SecurityPolicy::class => Piwik\DI::decorate(function ($previous) {
        if (!\Piwik\SettingsPiwik::isMatomoInstalled()) {
            return $previous;
        }
        $previous->addPolicy('default-src', 'my-cdn.example.com');
        return $previous;
    }),
];
```

---

## 4. Access Control Audit

Read `references/permissions-reference.md` for the full permissions model.

### Every API method needs a permission check

```php
// ✅ CORRECT — explicit permission check
public function getReport(int $idSite, string $period, string $date): DataTable
{
    Piwik::checkUserHasViewAccess($idSite);
    // ...
}

// ❌ WRONG — no permission check
public function getReport(int $idSite, string $period, string $date): DataTable
{
    // ... anyone can call this
}
```

### Controller actions — always check, not just menu visibility

```php
// ✅ CORRECT — permission check in the controller
public function adminPage(): string
{
    Piwik::checkUserHasSuperUserAccess();
    // ...
}

// ❌ WRONG — relying only on menu visibility
// (user can still access the URL directly!)
```

### Admin controllers must extend ControllerAdmin

```php
// ✅ CORRECT
class Controller extends \Piwik\Plugin\ControllerAdmin
{
    // ...
}

// ❌ WRONG — admin functionality in a regular controller
class Controller extends \Piwik\Plugin\Controller
{
    public function adminAction() { /* admin stuff */ }
}
```

---

## 5. Dependency & Code Audit

### Forbidden functions

Never use these in plugin code:
- `eval()` — arbitrary code execution
- `exec()`, `passthru()`, `system()`, `popen()` — shell execution
- `preg_replace()` with `/e` modifier — code execution
- `unserialize()` with user input — use `Common::safe_unserialize()` or JSON instead
- `file_get_contents()` with user-controlled protocol (phar://)
- `require`/`include` with user-controlled paths

### No hardcoded secrets

```php
// ❌ WRONG
$apiKey = 'sk-abc123secret';

// ✅ CORRECT — use Settings framework
$settings = new SystemSettings();
$apiKey = $settings->apiKey->getValue();
```

### External links

All external links must have security attributes:
```html
<a href="https://example.com" rel="noreferrer noopener" target="_blank">
```

### Timing-safe comparisons

For sensitive comparisons (tokens, hashes):
```php
// ✅ CORRECT
Common::hashEquals($expected, $actual);

// ❌ WRONG — vulnerable to timing attacks
$expected === $actual;
```

---

## 6. Performance Security

### Tracker plugins
- Limit how often the tracker sends requests per page view
- Example: limit form tracking to N forms per page view

### Live / raw data features
- Respect `live_query_max_execution_time` config setting
- Limit date ranges for raw data queries (e.g. cap `lastMinutes` parameter)

### Archiver
- Always respect `datatable_archiving_maximum_rows_standard` limits
- Use LogAggregator methods (they handle query timeouts and segmentation)

---

## 7. Object Injection Prevention

### Never unserialize user input

PHP's `unserialize()` can trigger arbitrary object instantiation and method calls:

```php
// ❌ DANGEROUS — arbitrary code execution via crafted payloads
$data = unserialize($userInput);

// ❌ STILL DANGEROUS — even with allowed_classes, exploits exist
$data = unserialize($userInput, ['allowed_classes' => ['MyClass']]);

// ✅ CORRECT — use JSON for user-provided data
$data = json_decode($userInput, true);

// ✅ CORRECT — use Matomo's safe unserialize for internal data
$data = Common::safe_unserialize($trustedInternalData);
```

### Prefer JSON over serialization

When storing structured data in the database:

```php
// ✅ CORRECT — JSON storage
$db->query("INSERT INTO ... VALUES (?)", [json_encode($data)]);
$data = json_decode($row['data'], true);

// ❌ AVOID — PHP serialization in new code
$db->query("INSERT INTO ... VALUES (?)", [serialize($data)]);
```

---

## 8. File Upload Security

### MIME type validation

Never trust the client-provided MIME type:

```php
// ✅ CORRECT — validate MIME type server-side
$finfo = new \finfo(FILEINFO_MIME_TYPE);
$mimeType = $finfo->file($tmpPath);
$allowedTypes = ['image/png', 'image/jpeg', 'image/gif'];

if (!in_array($mimeType, $allowedTypes, true)) {
    throw new \Exception('Invalid file type');
}

// ❌ WRONG — trusting client-provided type
$mimeType = $_FILES['upload']['type']; // can be spoofed
```

### Path traversal prevention

```php
// ✅ CORRECT — sanitize filename, use basename
$filename = basename($uploadedName);
$filename = preg_replace('/[^a-zA-Z0-9._-]/', '', $filename);
$targetPath = PIWIK_DOCUMENT_ROOT . '/tmp/uploads/' . $filename;

// ❌ WRONG — user-controlled path
$targetPath = PIWIK_DOCUMENT_ROOT . '/' . $userProvidedPath;
```

### Temp file handling

```php
// ✅ CORRECT — use Matomo's tmp directory, clean up after processing
$tmpDir = StaticContainer::get('path.tmp') . '/uploads/';
// Process file, then delete
@unlink($tmpPath);
```

---

## 9. Cookie Security

### Secure cookie attributes

When setting cookies, always use secure attributes:

```php
// ✅ CORRECT — secure cookie with all flags
Common::setCookie('my_cookie', $value, $expire, $path, $domain, $secure = true, $httpOnly = true, $sameSite = 'Lax');

// ❌ WRONG — cookie without security flags
setcookie('my_cookie', $value); // no HttpOnly, no Secure, no SameSite
```

### Cookie guidelines

- **HttpOnly**: Always set for cookies not needed by JavaScript (prevents XSS theft)
- **Secure**: Always set in production (prevents transmission over HTTP)
- **SameSite**: Use `Lax` (default) or `Strict` (prevents CSRF via cookies)
- **Never store secrets in cookies** — store only session IDs or non-sensitive preferences

---

## 10. Error Handling Security

### No stack traces in responses

```php
// ✅ CORRECT — generic error message to user, log details
try {
    $this->processData($input);
} catch (\Exception $e) {
    Log::error('Processing failed: ' . $e->getMessage());
    throw new \Exception(Piwik::translate('MyPlugin_ProcessingError'));
}

// ❌ WRONG — exposing stack trace and internal paths
try {
    $this->processData($input);
} catch (\Exception $e) {
    echo $e->getTraceAsString(); // reveals file paths, class names
}
```

### No internal paths in output

```php
// ❌ WRONG — reveals server file structure
echo "Error in " . __FILE__ . " line " . __LINE__;

// ✅ CORRECT — use translation keys for user-facing errors
throw new \Exception(Piwik::translate('MyPlugin_InvalidInput'));
```

---

## 11. Logging Security

### Never log sensitive data

```php
// ❌ WRONG — logging credentials and tokens
Log::debug('API call with token: ' . $tokenAuth);
Log::info('User password: ' . $password);

// ✅ CORRECT — log actions without sensitive values
Log::debug('API call received for method: ' . $method);
Log::info('User authentication attempt for login: ' . $login);
```

### Guidelines

- Never log `token_auth`, passwords, API keys, or session IDs
- Never log personally identifiable information (PII) like email addresses or IP addresses
  unless the Matomo instance is specifically configured to collect them
- Use Matomo's `Log` class, not `error_log()` or `file_put_contents()`
- In production, ensure debug-level logging is disabled

---

## 12. Final Checklist

Run through this checklist for every generated plugin:

- [ ] **Authorization**: Every controller action and API method has a permission check
- [ ] **CSRF**: Every form/action/API that changes data has a CSRF nonce or token check
- [ ] **SQL injection**: All DB parameters use bound parameters or are cast to int
- [ ] **XSS**: All user input is escaped (PHP, Twig, JavaScript, Vue)
- [ ] **Twig escaping**: Correct strategy used (`html`, `html_attr`, `js`, `css`, `url`)
- [ ] **Timing attacks**: `Common::hashEquals()` for sensitive comparisons
- [ ] **Data exposure**: No tokens, passwords, or secrets in HTML/logs/console
- [ ] **Secure storage**: Passwords/sessions stored as secure hashes
- [ ] **External links**: `rel="noreferrer noopener"` on all external links
- [ ] **No forbidden functions**: No eval, exec, passthru, system, popen
- [ ] **No hardcoded secrets**: All credentials in Settings framework
- [ ] **Object injection**: No `unserialize()` on user input, use JSON or `safe_unserialize()`
- [ ] **File uploads**: Server-side MIME validation, no path traversal, temp file cleanup
- [ ] **Cookie security**: HttpOnly, Secure, SameSite flags on all cookies
- [ ] **Error handling**: No stack traces or internal paths in user-facing output
- [ ] **Logging**: No tokens, passwords, PII, or secrets in log output
- [ ] **PHP file format**: Files start with `<?php`, never close the tag
- [ ] **File extensions**: All PHP scripts use `.php` extension
- [ ] **No debug code**: No var_dump, print_r, error_log left in code
