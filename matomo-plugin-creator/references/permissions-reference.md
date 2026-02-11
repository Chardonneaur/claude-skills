# Permissions Reference

Matomo's role-based access control system for plugin developers.

## Table of Contents

1. [Roles Hierarchy](#roles-hierarchy)
2. [Permission Check Methods](#permission-check-methods)
3. [Where to Check Permissions](#where-to-check-permissions)
4. [Common Patterns](#common-patterns)

---

## Roles Hierarchy

Roles are hierarchical — higher roles inherit all permissions of lower roles.

```
superuser
  └── admin (per site)
        └── write (per site)
              └── view (per site)
                    └── anonymous (no login)
```

| Role | Can do | Typical use |
|------|--------|-------------|
| **anonymous** | View public data only | Unauthenticated visitors |
| **view** | View all reports for assigned sites | Read-only analysts |
| **write** | View + track data, manage goals, funnels | Content editors |
| **admin** | Write + manage users, sites, settings for assigned sites | Site admins |
| **superuser** | Everything, all sites, system-wide settings | System administrators |

---

## Permission Check Methods

All methods are on the `Piwik\Piwik` class.

### Exception-throwing methods (use in API/Controller)

These throw an exception if the check fails, stopping execution and showing an error.

```php
// Super user access (system-wide admin)
Piwik::checkUserHasSuperUserAccess();

// Admin access for specific site(s)
Piwik::checkUserHasAdminAccess($idSite);
Piwik::checkUserHasAdminAccess([$idSite1, $idSite2]);

// At least admin access for ANY site
Piwik::checkUserHasSomeAdminAccess();

// Write access for specific site(s)
Piwik::checkUserHasWriteAccess($idSite);
Piwik::checkUserHasWriteAccess([$idSite1, $idSite2]);

// View access for specific site(s)
Piwik::checkUserHasViewAccess($idSite);
Piwik::checkUserHasViewAccess([$idSite1, $idSite2]);

// At least view access for ANY site
Piwik::checkUserHasSomeViewAccess();

// Super user OR is the specific user
Piwik::checkUserHasSuperUserAccessOrIsTheUser($login);

// Logged in (not anonymous)
Piwik::checkUserIsNotAnonymous();
```

### Boolean methods (use in conditionals)

These return true/false without throwing exceptions. Use for conditional UI elements.

```php
Piwik::hasUserSuperUserAccess();          // bool
Piwik::isUserHasAdminAccess($idSite);     // bool
Piwik::isUserHasWriteAccess($idSite);     // bool
Piwik::isUserHasViewAccess($idSite);      // bool
Piwik::isUserIsAnonymous();               // bool

// Get current user info
Piwik::getCurrentUserLogin();             // string
Piwik::getCurrentUserTokenAuth();         // string (use carefully!)
```

---

## Where to Check Permissions

### API methods — ALWAYS check

Every public API method must have a permission check as its first line:

```php
class API extends \Piwik\Plugin\API
{
    // Read-only report: view access required
    public function getReport(int $idSite, string $period, string $date, ?string $segment = null): DataTable
    {
        Piwik::checkUserHasViewAccess($idSite);
        // ...
    }

    // Modifying data: write or admin access required
    public function saveConfiguration(int $idSite, string $value): void
    {
        Piwik::checkUserHasAdminAccess($idSite);
        // ...
    }

    // System-wide operation: super user required
    public function getGlobalSettings(): array
    {
        Piwik::checkUserHasSuperUserAccess();
        // ...
    }
}
```

### Controller actions — ALWAYS check

**Critical:** Just because a menu item is hidden doesn't mean the URL is inaccessible.
Users can type URLs directly. Always check permissions in the controller.

```php
class Controller extends \Piwik\Plugin\ControllerAdmin
{
    public function index(): string
    {
        // Even if the menu only shows this for super users,
        // check here too — URLs can be accessed directly
        Piwik::checkUserHasSuperUserAccess();

        $view = new View('@MyPlugin/index');
        return $view->render();
    }
}
```

### Menu items — conditional visibility

```php
class Menu extends \Piwik\Plugin\Menu
{
    public function configureAdminMenu(MenuAdmin $menu): void
    {
        // Only show if user has super user access
        if (Piwik::hasUserSuperUserAccess()) {
            $menu->addSystemItem(
                Piwik::translate('MyPlugin_MenuTitle'),
                $this->urlForAction('index'),
                $order = 30
            );
        }
    }

    public function configureReportingMenu(MenuReporting $menu): void
    {
        // Show to anyone with view access to at least one site
        if (!Piwik::isUserIsAnonymous()) {
            $menu->addItem(
                'General_Actions',
                'MyPlugin_SubMenuTitle',
                $this->urlForAction('report'),
                $order = 40
            );
        }
    }
}
```

### Widgets — conditional display

```php
class ExampleWidget extends Widget
{
    public static function configure(WidgetConfig $config): void
    {
        $config->setCategoryId('General_Visitors');
        $config->setName('MyPlugin_WidgetTitle');

        // Only visible for admin users
        if (!Piwik::hasUserSuperUserAccess()) {
            $config->disable();
        }
    }
}
```

### Settings — access control

```php
class Settings extends \Piwik\Plugin\Settings
{
    protected function init(): void
    {
        $this->apiKey = $this->createApiKeySetting();

        // Restrict who can modify this setting
        $this->apiKey->setIsWritableByCurrentUser(
            Piwik::hasUserSuperUserAccess()
        );
    }
}
```

---

## Common Patterns

### Report with site-specific access

```php
// API method
public function getMyReport(int $idSite, string $period, string $date, ?string $segment = null): DataTable
{
    Piwik::checkUserHasViewAccess($idSite);
    $archive = Archive::build($idSite, $period, $date, $segment);
    return $archive->getDataTable(Archiver::RECORD_NAME);
}
```

### Admin action with CSRF protection

```php
// Controller method
public function updateSettings(): void
{
    Piwik::checkUserHasSuperUserAccess();
    $this->checkTokenInUrl();

    $nonce = Common::getRequestVar('nonce', '', 'string');
    Nonce::checkNonce('MyPlugin.updateSettings', $nonce);

    // ... perform the update
    $this->redirectToIndex('MyPlugin', 'index');
}
```

### Super user OR the affected user

Useful for user-specific settings:

```php
public function updateUserPreference(string $userLogin, string $preference, string $value): void
{
    Piwik::checkUserHasSuperUserAccessOrIsTheUser($userLogin);
    // User can change their own preference, super user can change anyone's
}
```

### Checking access in Twig templates

```twig
{% if hasSuperUserAccess %}
    <div class="admin-controls">
        {# Admin-only UI elements #}
    </div>
{% endif %}
```

Set this variable in your Controller:
```php
$view->hasSuperUserAccess = Piwik::hasUserSuperUserAccess();
```
