# Updates & Migrations Reference

How to create version updates, database migrations, and manage plugin lifecycle in Matomo.
Based on real patterns from CustomAlerts, TagManager, FormAnalytics, Funnels, and core updates.

## Table of Contents

1. [Concepts](#concepts)
2. [File Naming & Class Structure](#file-naming--class-structure)
3. [Migration Factory Methods](#migration-factory-methods)
4. [Update Examples](#update-examples)
5. [Conditional Migrations](#conditional-migrations)
6. [Non-Database Updates](#non-database-updates)
7. [Plugin Lifecycle Methods](#plugin-lifecycle-methods)
8. [Version Bumping](#version-bumping)
9. [Best Practices](#best-practices)

---

## Concepts

Matomo's update system handles schema changes and data migrations when a plugin version
changes. The system:

1. Compares the plugin version in `plugin.json` against the last installed version
2. Discovers update files in `Updates/` that match version numbers between old and new
3. Executes each update in version order

Updates have two key methods:
- `getMigrations()` — declares what database changes to make (previewable)
- `doUpdate()` — executes the migrations plus any PHP logic

---

## File Naming & Class Structure

### File location and naming

```
PluginName/
├── Updates/
│   ├── 1.0.0.php
│   ├── 1.1.0.php
│   ├── 2.0.0-b1.php      ← beta version
│   └── 2.0.0-rc1.php     ← release candidate
```

### Naming conventions

| Element | Format | Example |
|---------|--------|---------|
| Filename | `X.Y.Z.php` or `X.Y.Z-bN.php` | `5.2.0.php`, `5.2.0-b1.php` |
| Class name | `Updates_X_Y_Z` (dots → underscores) | `Updates_5_2_0`, `Updates_5_2_0_b1` |
| Namespace | `Piwik\Plugins\{PluginName}` | `Piwik\Plugins\CustomAlerts` |

### Basic structure

```php
<?php

namespace Piwik\Plugins\{PluginName};

use Piwik\Updater;
use Piwik\Updates as PiwikUpdates;
use Piwik\Updater\Migration\Factory as MigrationFactory;

class Updates_1_0_0 extends PiwikUpdates
{
    /** @var MigrationFactory */
    private $migration;

    public function __construct(MigrationFactory $factory)
    {
        $this->migration = $factory;
    }

    /**
     * Declare database migrations (previewable without execution).
     */
    public function getMigrations(Updater $updater): array
    {
        return [
            $this->migration->db->addColumn('my_table', 'new_column', 'VARCHAR(255) NULL'),
        ];
    }

    /**
     * Execute the update. Must call executeMigrations() if getMigrations() is defined.
     */
    public function doUpdate(Updater $updater): void
    {
        $updater->executeMigrations(__FILE__, $this->getMigrations($updater));
    }
}
```

---

## Migration Factory Methods

Access via `$this->migration->db->methodName()` in your update class.

**Important:** Table names passed to factory methods are automatically prefixed.
Pass unprefixed names (e.g., `'alert'` not `'matomo_alert'`).

### Column operations

```php
// Add a single column
$this->migration->db->addColumn(
    'table_name',           // unprefixed table name
    'new_column',           // column name
    'VARCHAR(255) NOT NULL',// column type with constraints
    'after_column'          // optional: position after this column
)

// Add multiple columns at once (better performance — single ALTER)
$this->migration->db->addColumns(
    'table_name',
    [
        'column_a' => 'VARCHAR(100) NOT NULL',
        'column_b' => 'INT UNSIGNED DEFAULT 0',
    ],
    'after_column'          // optional
)

// Drop a column
$this->migration->db->dropColumn('table_name', 'old_column')

// Drop multiple columns
$this->migration->db->dropColumns('table_name', ['col_a', 'col_b'])

// Change column type only
$this->migration->db->changeColumnType('table_name', 'column_name', 'BIGINT NOT NULL')

// Change multiple column types
$this->migration->db->changeColumnTypes('table_name', [
    'col_a' => 'VARCHAR(500) NOT NULL',
    'col_b' => 'INT UNSIGNED DEFAULT 0',
])

// Rename and change type
$this->migration->db->changeColumn('table_name', 'old_name', 'new_name', 'VARCHAR(255) NOT NULL')
```

### Table operations

```php
// Create a table
$this->migration->db->createTable(
    'new_table',
    [
        'id'    => 'INT UNSIGNED NOT NULL AUTO_INCREMENT',
        'name'  => 'VARCHAR(100) NOT NULL',
        'data'  => 'TEXT NULL',
    ],
    ['id']  // primary key column(s)
)

// Drop a table
$this->migration->db->dropTable('old_table')
```

### Index operations

```php
// Add an index
$this->migration->db->addIndex('table_name', ['column_a', 'column_b'], 'index_name')

// Add an index with prefix length
$this->migration->db->addIndex('table_name', ['column_a(10)', 'column_b'], 'index_name')

// Add unique key
$this->migration->db->addUniqueKey('table_name', ['column_a'], 'unique_name')

// Drop an index
$this->migration->db->dropIndex('table_name', 'index_name')

// Add/drop primary key
$this->migration->db->addPrimaryKey('table_name', ['id', 'date'])
$this->migration->db->dropPrimaryKey('table_name')
```

### Data operations

```php
// Insert a row (ignores duplicate entry errors)
$this->migration->db->insert('table_name', [
    'column1' => 'value1',
    'column2' => 'value2',
])

// Batch insert (uses LOAD DATA INFILE when available)
$this->migration->db->batchInsert(
    'table_name',
    ['col_a', 'col_b'],        // column names
    [['val1', 'val2'], ...],   // rows of values
    false,                      // throwException
    'utf8'                      // charset
)
```

### Raw SQL

```php
// Custom SQL (use Common::prefixTable() yourself)
$this->migration->db->sql(
    'ALTER TABLE `' . \Piwik\Common::prefixTable('my_table') . '` ADD created_time DATETIME NOT NULL',
    []  // error codes to ignore (optional)
)

// Parameterized SQL
$this->migration->db->boundSql(
    'UPDATE `' . \Piwik\Common::prefixTable('my_table') . '` SET status = ? WHERE type = ?',
    ['active', 'default'],  // bind parameters
    []                       // error codes to ignore
)
```

### Error codes to ignore

The `Db` migration base class defines constants for common MySQL error codes:

```php
use Piwik\Updater\Migration\Db;

Db::ERROR_CODE_TABLE_EXISTS          // 1050
Db::ERROR_CODE_UNKNOWN_COLUMN        // 1054
Db::ERROR_CODE_DUPLICATE_COLUMN      // 1060
Db::ERROR_CODE_DUPLICATE_KEY         // 1061
Db::ERROR_CODE_DUPLICATE_ENTRY       // 1062
Db::ERROR_CODE_COLUMN_NOT_EXISTS     // 1091
Db::ERROR_CODE_TABLE_NOT_EXISTS      // 1146
```

Pass these to `sql()` or `boundSql()` to make migrations idempotent:

```php
$this->migration->db->sql(
    "ALTER TABLE `{$table}` ADD `new_col` INT",
    [Db::ERROR_CODE_DUPLICATE_COLUMN]  // ignore "column already exists"
)
```

---

## Update Examples

### Simple column addition

```php
<?php
// plugins/CustomAlerts/Updates/5.2.0.php

namespace Piwik\Plugins\CustomAlerts;

use Piwik\Updater;
use Piwik\Updates as PiwikUpdates;
use Piwik\Updater\Migration\Factory as MigrationFactory;

class Updates_5_2_0 extends PiwikUpdates
{
    private $migration;

    public function __construct(MigrationFactory $factory)
    {
        $this->migration = $factory;
    }

    public function getMigrations(Updater $updater): array
    {
        return [
            $this->migration->db->addColumn('alert', 'ms_teams_webhook_url', 'VARCHAR(500) NULL', 'slack_channel_id'),
            $this->migration->db->addColumn('alert_triggered', 'ms_teams_webhook_url', 'VARCHAR(500) NULL', 'slack_channel_id'),
        ];
    }

    public function doUpdate(Updater $updater): void
    {
        $updater->executeMigrations(__FILE__, $this->getMigrations($updater));
    }
}
```

### Mixed migrations with post-migration logic

```php
<?php
// plugins/TagManager/Updates/5.2.0-b1.php

namespace Piwik\Plugins\TagManager;

use Piwik\Updater;
use Piwik\Updates as PiwikUpdates;
use Piwik\Updater\Migration\Factory as MigrationFactory;

class Updates_5_2_0_b1 extends PiwikUpdates
{
    private $migration;

    public function __construct(MigrationFactory $factory)
    {
        $this->migration = $factory;
    }

    public function getMigrations(Updater $updater): array
    {
        return [
            $this->migration->db->addColumn(
                'tagmanager_container',
                'isTagFireLimitAllowedInPreviewMode',
                'TINYINT(1) UNSIGNED NOT NULL DEFAULT 0',
                'ignoreGtmDataLayer'
            ),
            $this->migration->db->changeColumn(
                'tagmanager_container_version', 'name', 'name',
                "VARCHAR(255) NOT NULL DEFAULT ''"
            ),
            $this->migration->db->changeColumn(
                'tagmanager_tag', 'name', 'name',
                'VARCHAR(255) NOT NULL'
            ),
        ];
    }

    public function doUpdate(Updater $updater): void
    {
        // Execute DB migrations first
        $updater->executeMigrations(__FILE__, $this->getMigrations($updater));

        // Then run additional PHP logic
        $migrator = new UpdateHelper\NewVariableParameterMigrator(
            Template\Variable\MatomoConfigurationVariable::ID,
            'trackBots',
            false
        );
        $migrator->migrate();
    }
}
```

---

## Conditional Migrations

Check if a migration is needed before adding it. This makes updates safe to re-run.

```php
<?php
// plugins/FormAnalytics/Updates/5.1.0.php

namespace Piwik\Plugins\FormAnalytics;

use Piwik\Common;
use Piwik\DbHelper;
use Piwik\Updater;
use Piwik\Updates as PiwikUpdates;
use Piwik\Updater\Migration\Factory as MigrationFactory;

class Updates_5_1_0 extends PiwikUpdates
{
    private $migration;

    public function __construct(MigrationFactory $factory)
    {
        $this->migration = $factory;
    }

    public function getMigrations(Updater $updater): array
    {
        $migrations = [];

        // Check if column already exists before adding
        $columns = DbHelper::getTableColumns(Common::prefixTable('site_form'));

        if (!array_key_exists('idgoal', $columns)) {
            $migrations[] = $this->migration->db->addColumn('site_form', 'idgoal', 'INT NULL', 'idsite');
        }

        return $migrations;
    }

    public function doUpdate(Updater $updater): void
    {
        $updater->executeMigrations(__FILE__, $this->getMigrations($updater));
    }
}
```

---

## Non-Database Updates

Updates that only modify configuration or perform PHP logic don't need `getMigrations()`.

```php
<?php
// plugins/CustomReports/Updates/5.4.0.php

namespace Piwik\Plugins\CustomReports;

use Piwik\Config;
use Piwik\Updater;
use Piwik\Updates as PiwikUpdates;

class Updates_5_4_0 extends PiwikUpdates
{
    public function doUpdate(Updater $updater): void
    {
        // Migrate configuration values
        $config = Config::getInstance();
        $pluginConfig = $config->CustomReports;

        $pluginConfig['new_setting'] = $pluginConfig['old_setting'] ?? 'default';
        unset($pluginConfig['old_setting']);

        $config->CustomReports = $pluginConfig;
        $config->forceSave();
    }
}
```

---

## Plugin Lifecycle Methods

Beyond versioned updates, plugins have lifecycle hooks in the main plugin class.

```php
<?php

namespace Piwik\Plugins\{PluginName};

use Piwik\Common;
use Piwik\Db;
use Piwik\Plugin;

class {PluginName} extends Plugin
{
    /**
     * Called when the plugin is first installed.
     * Must be idempotent — may be called multiple times.
     */
    public function install(): void
    {
        $table = Common::prefixTable('my_table');

        $sql = "CREATE TABLE IF NOT EXISTS `{$table}` (
            `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
            `idsite` INT UNSIGNED NOT NULL,
            `name` VARCHAR(255) NOT NULL,
            `created_date` DATETIME NOT NULL,
            PRIMARY KEY (`id`),
            INDEX `idx_idsite` (`idsite`)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;";

        Db::exec($sql);
    }

    /**
     * Called when the plugin is uninstalled (removed entirely).
     * Drop all plugin tables here.
     */
    public function uninstall(): void
    {
        Db::dropTables([Common::prefixTable('my_table')]);
    }

    /**
     * Called when the plugin is activated.
     * Runs after install(). Good for seeding initial data.
     */
    public function activate(): void
    {
        // Optional: seed default data, register capabilities, etc.
    }

    /**
     * Called when the plugin is deactivated.
     * Do NOT drop tables here — that's for uninstall().
     */
    public function deactivate(): void
    {
        // Optional: cleanup temporary data, revoke capabilities, etc.
    }
}
```

### DAO pattern for table management

For plugins with multiple tables, use a DAO class:

```php
<?php

namespace Piwik\Plugins\{PluginName}\Dao;

use Piwik\Common;
use Piwik\Db;
use Piwik\DbHelper;

class MyEntityDao
{
    private string $table;
    private string $tablePrefixed;

    public function __construct()
    {
        $this->table = 'my_entity';
        $this->tablePrefixed = Common::prefixTable($this->table);
    }

    public function install(): void
    {
        DbHelper::createTable($this->table, "
            `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
            `idsite` INT UNSIGNED NOT NULL,
            `name` VARCHAR(255) NOT NULL,
            `status` TINYINT(1) NOT NULL DEFAULT 0,
            `created_date` DATETIME NOT NULL,
            PRIMARY KEY (`id`),
            INDEX `idx_idsite` (`idsite`)
        ");
    }

    public function uninstall(): void
    {
        Db::dropTables([$this->tablePrefixed]);
    }
}
```

Then in your main plugin class:

```php
public function install(): void
{
    $dao = new Dao\MyEntityDao();
    $dao->install();
}

public function uninstall(): void
{
    $dao = new Dao\MyEntityDao();
    $dao->uninstall();
}
```

---

## Version Bumping

When creating an update, bump the version in `plugin.json`:

```json
{
    "name": "MyPlugin",
    "version": "1.1.0",
    "require": {
        "matomo": ">=5.0.0-b1,<6.0.0-b1"
    }
}
```

Then create the corresponding update file: `Updates/1.1.0.php`

Matomo detects the version change and runs any update files between the old and new versions.

---

## Best Practices

1. **Always define `getMigrations()`** — allows preview of SQL changes without execution
2. **Always call `executeMigrations()` in `doUpdate()`** — required to actually run migrations
3. **Use factory methods** over raw SQL — they handle table prefixing and are more readable
4. **Make migrations idempotent** — check column existence with `DbHelper::getTableColumns()` or pass error codes to ignore
5. **Use `CREATE TABLE IF NOT EXISTS`** in `install()` — safe for repeated calls
6. **Use `addColumns()` over multiple `addColumn()`** — single ALTER TABLE is faster
7. **Order migrations carefully** — they execute sequentially in array order
8. **Separate concerns** — DB migrations in `getMigrations()`, PHP logic in `doUpdate()`
9. **Never drop tables in `deactivate()`** — only in `uninstall()`
10. **Test updates** — verify they work on fresh installs AND on upgrades from previous versions
