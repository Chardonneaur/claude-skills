# Twig + jQuery Admin Page Patterns

**This is the DEFAULT approach for admin UI pages.** It works on all Matomo installs
(production zip installs, dev checkouts) with zero build step. Vue 3 is optional and
requires `./console vue:build` which only works in dev git checkouts.

---

## Controller Pattern

Load all data **server-side** and pass it to the Twig template. Never rely on
client-side AJAX for initial page data — this avoids authentication issues, UMD
bundle timing issues, and `ajaxHelper` availability problems.

```php
namespace Piwik\Plugins\MyPlugin;

use Piwik\Common;
use Piwik\Nonce;
use Piwik\Notification;
use Piwik\Piwik;
use Piwik\View;

class Controller extends \Piwik\Plugin\ControllerAdmin
{
    const ACTION_NONCE = 'MyPlugin.action';

    public function index(): string
    {
        Piwik::checkUserHasSuperUserAccess();

        // Handle POST actions before rendering
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            $this->handleAction();
        }

        $view = new View('@MyPlugin/index');
        $this->setGeneralVariablesView($view);

        // Load data server-side — pass to template
        $view->items = $this->loadItems();
        $view->actionNonce = Nonce::getNonce(self::ACTION_NONCE);

        return $view->render();
    }

    private function handleAction(): void
    {
        // Validate CSRF nonce
        $nonce = Common::getRequestVar('nonce', '', 'string', $_POST);
        Nonce::checkNonce(self::ACTION_NONCE, $nonce);

        // Read POST parameters safely
        $id = Common::getRequestVar('id', 0, 'int', $_POST);

        // Perform action...

        // Show notification (displayed after page re-renders)
        $notification = new Notification(Piwik::translate('MyPlugin_Success'));
        $notification->context = Notification::CONTEXT_SUCCESS;
        Notification\Manager::notify('MyPlugin_ActionResult', $notification);
    }
}
```

---

## Menu Placement

Choose the correct menu section based on what the plugin does:

```php
use Piwik\Menu\MenuAdmin;

class Menu extends \Piwik\Plugin\Menu
{
    public function configureAdminMenu(MenuAdmin $menu): void
    {
        if (Piwik::hasUserSuperUserAccess()) {
            // Website/measurable features (sites, goals, tracking code, etc.)
            $menu->addMeasurableItem('Menu Title', $this->urlForAction('index'), $order = 35);

            // System/platform features (config, cron, emails, etc.)
            // $menu->addSystemItem('Menu Title', $this->urlForAction('index'), $order = 35);

            // Diagnostic tools (check, audit, debug, etc.)
            // $menu->addDiagnosticItem('Menu Title', $this->urlForAction('index'), $order = 35);
        }
    }
}
```

---

## Twig Template Structure

```twig
{% extends 'admin.twig' %}

{% block content %}
<style>
/* Materialize CSS hides native checkboxes — override is REQUIRED */
.myPlugin-table input[type="checkbox"] {
    opacity: 1;
    position: static;
    pointer-events: auto;
    width: 16px;
    height: 16px;
    cursor: pointer;
}
</style>

<div class="card">
    <div class="card-content">
        <h2 class="card-title">{{ 'MyPlugin_Title'|translate }}</h2>

        {# Data rendered server-side from Controller #}
        {% if items is empty %}
            <div class="alert alert-info">{{ 'MyPlugin_NoItems'|translate }}</div>
        {% else %}
            <form method="post" id="myPluginForm">
                <input type="hidden" name="nonce" value="{{ actionNonce }}" />
                <input type="hidden" name="selectedIds" id="myPluginSelectedIds" value="" />

                <table class="entityTable myPlugin-table">
                    <thead>
                        <tr>
                            <th style="width: 40px;">
                                <input type="checkbox" id="checkAll" />
                            </th>
                            <th>ID</th>
                            <th>{{ 'General_Name'|translate }}</th>
                        </tr>
                    </thead>
                    <tbody>
                        {% for item in items %}
                        <tr data-id="{{ item.id }}">
                            <td><input type="checkbox" class="item-check" data-id="{{ item.id }}" /></td>
                            <td>{{ item.id }}</td>
                            <td>{{ item.name }}</td>
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>

                <button type="submit" class="btn">{{ 'MyPlugin_Submit'|translate }}</button>
            </form>
        {% endif %}
    </div>
</div>

<script>
$(function () {
    // JS for interactivity (filter, select, etc.)
    // Data is already in the DOM — no AJAX needed for initial load
});
</script>
{% endblock %}
```

---

## Twig Translations in JavaScript

**Correct syntax** — quotes OUTSIDE the Twig tags:

```javascript
var label = '{{ "MyPlugin_Label"|translate|e("js") }}';
var msg = '{{ "MyPlugin_Message"|translate("%1$s")|e("js") }}';
$('#el').text(msg.replace('%1$s', count));
```

**WRONG syntax** — this outputs raw text, filters are NOT applied:

```javascript
// BROKEN: {{ "..." }} makes it a Twig string literal, filters ignored
var label = {{ "'MyPlugin_Label'|translate|e('js')" }};
```

Why: `{{ "some string" }}` outputs the string literally. The `|translate` and `|e()`
inside the quotes are NOT treated as filters — they become part of the raw text output.

---

## CSRF Protection with Nonce

For form-based actions, use Matomo's `Nonce` system (NOT `token_auth` in JavaScript):

**Controller** — generate and validate:
```php
use Piwik\Nonce;

// Generate (in index action, passed to template)
$view->deleteNonce = Nonce::getNonce('MyPlugin.delete');

// Validate (in POST handler)
$nonce = Common::getRequestVar('nonce', '', 'string', $_POST);
Nonce::checkNonce('MyPlugin.delete', $nonce);
```

**Template** — include in form:
```html
<form method="post">
    <input type="hidden" name="nonce" value="{{ deleteNonce }}" />
    <!-- form fields -->
</form>
```

---

## Notifications (Server-Side)

Use Matomo's `Notification\Manager` in the Controller — notifications display automatically
after the page re-renders:

```php
use Piwik\Notification;

// Success
$notification = new Notification(Piwik::translate('MyPlugin_DeletedSuccess', $count));
$notification->context = Notification::CONTEXT_SUCCESS;
Notification\Manager::notify('MyPlugin_Success', $notification);

// Error
$notification = new Notification($errorMessage);
$notification->context = Notification::CONTEXT_ERROR;
Notification\Manager::notify('MyPlugin_Error', $notification);
```

Available contexts: `CONTEXT_SUCCESS`, `CONTEXT_ERROR`, `CONTEXT_WARNING`, `CONTEXT_INFO`.

---

## Client-Side Search/Filter (No AJAX)

Filter table rows by showing/hiding — data is already in the DOM:

```javascript
$('#searchInput').on('input', function () {
    var term = ($(this).val() || '').toLowerCase();
    $('#tableBody tr').each(function () {
        var $row = $(this);
        var name = ($row.data('name') || '').toString().toLowerCase();
        var match = !term || name.indexOf(term) !== -1;
        $row.toggle(match);
    });
});
```

---

## Why NOT ajaxHelper in Inline Scripts

`window.ajaxHelper` is defined in `CoreHome.umd.js` (a Vue UMD bundle). UMD bundles
may load AFTER inline `<script>` blocks in `{% block content %}` execute, even inside
`$(function() {...})` (DOM ready fires before deferred scripts complete).

**If you absolutely need AJAX** (e.g., for a non-reload action), use plain `$.ajax`:

```javascript
$.ajax({
    url: 'index.php',
    method: 'POST',
    data: {
        module: 'API',
        method: 'MyPlugin.myAction',
        format: 'json',
        token_auth: piwik.token_auth,
        param1: value1
    },
    dataType: 'json',
    success: function (response) { /* handle */ },
    error: function () { /* handle */ }
});
```

**But prefer form POST** to the Controller for write actions — it's simpler, more
reliable, and uses proper Nonce CSRF protection.

---

## Materialize CSS Gotchas

Matomo uses Materialize CSS which aggressively styles form elements:

1. **Checkboxes are hidden** — native `<input type="checkbox">` gets `opacity: 0;
   position: absolute`. MUST override with CSS (see template example above).

2. **Inputs need `browser-default` class** — without it, Materialize adds custom
   styling that may conflict. Always add `class="browser-default"` on text inputs.

3. **Buttons** — use `class="btn"` for primary, `class="btn-flat"` for secondary.
   For danger actions, add inline style: `style="background-color: #c62828; color: #fff;"`

---

## Complete Example: Bulk Action Page

See the [BulkSiteDelete plugin](https://github.com/Chardonneaur/BulkSiteDelete) for
a complete working example of:
- Server-side data loading in Controller
- Twig table with checkboxes and search filter
- Form POST with Nonce CSRF protection
- Confirmation modal dialog
- Server-side notifications
- Materialize CSS checkbox fix
