# Plugin Types Reference

Class templates and hook patterns for each Matomo plugin type.
Read the section(s) relevant to the plugin being built.

## Table of Contents

1. [Main Plugin Class](#main-plugin-class)
2. [Tracking Plugins](#tracking-plugins)
3. [Reporting Plugins](#reporting-plugins)
4. [Admin/Settings Plugins](#adminsettings-plugins)
5. [Vue 3 Components](#vue-3-components)
6. [Event Hook Reference](#event-hook-reference)

---

## Main Plugin Class

Every plugin needs `{PluginName}.php` — the entry point and event subscriber.

```php
<?php

/**
 * See assets/gpl-v3-header.txt for the full GPL header to insert here.
 */

namespace Piwik\Plugins\{PluginName};

use Piwik\Plugin;

class {PluginName} extends Plugin
{
    public function registerEvents(): array
    {
        return [
            // Register event handlers here
            // 'EventName' => 'methodName',
        ];
    }

    public function install(): void
    {
        // DB schema creation — must be idempotent
    }

    public function uninstall(): void
    {
        // DB schema removal
    }

    public function activate(): void
    {
        // Called when the plugin is activated (after install)
    }

    public function deactivate(): void
    {
        // Called when the plugin is deactivated (before uninstall)
    }
}
```

### Registering assets and translations

```php
public function registerEvents(): array
{
    return [
        'AssetManager.getJavaScriptFiles'       => 'getJsFiles',
        'AssetManager.getStylesheetFiles'        => 'getCssFiles',
        'Translate.getClientSideTranslationKeys' => 'getClientSideTranslationKeys',
    ];
}

public function getJsFiles(array &$jsFiles): void
{
    $jsFiles[] = 'plugins/{PluginName}/javascripts/plugin.js';
}

public function getCssFiles(array &$cssFiles): void
{
    $cssFiles[] = 'plugins/{PluginName}/stylesheets/plugin.less';
}

public function getClientSideTranslationKeys(array &$translationKeys): void
{
    $translationKeys[] = '{PluginName}_SomeKey';
}
```

---

## Tracking Plugins

### Visit Dimension

Stores one value per visit. Computed on first action, optionally updated on subsequent actions.

```php
<?php

namespace Piwik\Plugins\{PluginName}\Columns;

use Piwik\Columns\DimensionSegmentFactory;
use Piwik\Plugin\Dimension\VisitDimension;
use Piwik\Segment\SegmentsList;
use Piwik\Tracker\Action;
use Piwik\Tracker\Request;
use Piwik\Tracker\Visitor;

class ExampleVisitDimension extends VisitDimension
{
    protected $columnName = 'example_visit_dimension';
    protected $columnType = 'VARCHAR(255) NULL DEFAULT NULL';
    protected $nameSingular = '{PluginName}_DimensionName';
    protected $segmentName = 'exampleDimension';

    public function onNewVisit(Request $request, Visitor $visitor, ?Action $action): mixed
    {
        return $request->getParam('example_param') ?: false;
    }

    public function onExistingVisit(Request $request, Visitor $visitor, ?Action $action): mixed
    {
        return false; // don't update
    }

    public function onAnyGoalConversion(Request $request, Visitor $visitor, ?Action $action): mixed
    {
        return $visitor->getVisitorColumn($this->columnName);
    }

    public function configureSegments(SegmentsList $segmentsList, DimensionSegmentFactory $dimensionSegmentFactory): void
    {
        $segment = $dimensionSegmentFactory->createSegment();
        $segment->setName('{PluginName}_SegmentName');
        $segmentsList->addSegment($segment);
    }
}
```

### Action Dimension

Stores one value per page view / event / download.

```php
<?php

namespace Piwik\Plugins\{PluginName}\Columns;

use Piwik\Plugin\Dimension\ActionDimension;
use Piwik\Tracker\Action;
use Piwik\Tracker\Request;
use Piwik\Tracker\Visitor;

class ExampleActionDimension extends ActionDimension
{
    protected $columnName = 'example_action_dimension';
    protected $columnType = 'VARCHAR(255) NULL DEFAULT NULL';
    protected $nameSingular = '{PluginName}_ActionDimensionName';
    protected $segmentName = 'exampleActionDimension';

    public function onNewAction(Request $request, Visitor $visitor, Action $action): mixed
    {
        $url = $action->getActionUrl();
        // Extract value from URL, custom parameter, etc.
        return false;
    }
}
```

### Custom Tracker JavaScript (tracker.js)

Extends the browser-side tracking code:

```javascript
(function () {
    Matomo.addPlugin('pluginName', {
        log: function () {
            // Called before each tracking request is sent
            // 'this' is the tracker instance
        },
        unload: function () {
            // Called on page unload
        }
    });
})();
```

Register in the main plugin class:
```php
public function registerEvents(): array
{
    return [
        'Tracker.getJavascriptCode' => 'getJavascriptCode',
    ];
}
```

---

## Reporting Plugins

### API Class

Public methods become API endpoints. Each needs explicit permission checks.

```php
<?php

namespace Piwik\Plugins\{PluginName};

use Piwik\Archive;
use Piwik\DataTable;
use Piwik\Piwik;
use Piwik\Plugin\API as PluginAPI;

class API extends PluginAPI
{
    /**
     * API URL: index.php?module=API&method={PluginName}.getExampleReport
     *          &idSite=1&period=day&date=today&format=json
     */
    public function getExampleReport(
        int $idSite,
        string $period,
        string $date,
        ?string $segment = null
    ): DataTable {
        Piwik::checkUserHasViewAccess($idSite);

        $archive = Archive::build($idSite, $period, $date, $segment);
        $dataTable = $archive->getDataTable(Archiver::RECORD_NAME);
        $dataTable->queueFilter('ReplaceColumnNames');
        $dataTable->queueFilter('AddColumnsProcessedMetrics');

        return $dataTable;
    }
}
```

### Report Class

Defines how a report appears in the Matomo UI.

```php
<?php

namespace Piwik\Plugins\{PluginName}\Reports;

use Piwik\Piwik;
use Piwik\Plugin\Report;
use Piwik\Plugin\ViewDataTable;
use Piwik\Report\ReportWidgetFactory;
use Piwik\Widget\WidgetsList;

class GetExampleReport extends Report
{
    protected function init(): void
    {
        parent::init();

        $this->categoryId = 'General_Actions';
        $this->subcategoryId = '{PluginName}_ReportCategory';
        $this->name = Piwik::translate('{PluginName}_ReportName');
        $this->documentation = Piwik::translate('{PluginName}_ReportDocumentation');
        $this->dimension = new \Piwik\Plugins\{PluginName}\Columns\ExampleDimension();
        $this->metrics = ['nb_visits', 'nb_uniq_visitors', 'nb_actions'];
        $this->order = 10;
    }

    public function configureWidgets(WidgetsList $widgetsList, ReportWidgetFactory $factory): void
    {
        $widgetsList->addWidgetConfig(
            $factory->createWidget()
                ->setName('{PluginName}_WidgetTitle')
                ->setOrder(10)
        );
    }

    public function configureView(ViewDataTable $view): void
    {
        $view->config->show_search = true;
        $view->config->show_limit_control = true;
        $view->config->columns_to_display = ['label', 'nb_visits', 'nb_actions'];
    }
}
```

### Widget Class (standalone)

For widgets not tied to a specific report:

```php
<?php

namespace Piwik\Plugins\{PluginName}\Widgets;

use Piwik\Common;
use Piwik\Piwik;
use Piwik\Widget\Widget;
use Piwik\Widget\WidgetConfig;

class ExampleWidget extends Widget
{
    public static function configure(WidgetConfig $config): void
    {
        $config->setCategoryId('General_Visitors');
        $config->setSubcategoryId('{PluginName}_Widgets');
        $config->setName('{PluginName}_WidgetTitle');
        $config->setOrder(99);
        $config->setIsEnabled(!Piwik::isUserIsAnonymous());
    }

    public function render(): string
    {
        Piwik::checkUserHasSomeViewAccess();

        return $this->renderTemplate('widget', [
            'data' => $this->fetchData(),
        ]);
    }

    private function fetchData(): array
    {
        return \Piwik\API\Request::processRequest('{PluginName}.getExampleReport', [
            'idSite' => Common::getRequestVar('idSite', 0, 'int'),
            'period' => Common::getRequestVar('period', '', 'string'),
            'date'   => Common::getRequestVar('date', '', 'string'),
        ]);
    }
}
```

---

## Admin/Settings Plugins

### Settings Class

```php
<?php

namespace Piwik\Plugins\{PluginName};

use Piwik\Piwik;
use Piwik\Settings\FieldConfig;
use Piwik\Settings\Plugin\SystemSetting;
use Piwik\Settings\Plugin\UserSetting;

class Settings extends \Piwik\Plugin\Settings
{
    public SystemSetting $apiEndpoint;
    public SystemSetting $enableFeature;
    public UserSetting $refreshInterval;

    protected function init(): void
    {
        $this->apiEndpoint = $this->makeSetting('apiEndpoint', '', FieldConfig::TYPE_STRING, function (FieldConfig $field) {
            $field->title = Piwik::translate('{PluginName}_ApiEndpoint');
            $field->uiControl = FieldConfig::UI_CONTROL_TEXT;
            $field->description = Piwik::translate('{PluginName}_ApiEndpointDesc');
        });

        $this->enableFeature = $this->makeSetting('enableFeature', false, FieldConfig::TYPE_BOOL, function (FieldConfig $field) {
            $field->title = Piwik::translate('{PluginName}_EnableFeature');
            $field->uiControl = FieldConfig::UI_CONTROL_CHECKBOX;
        });

        $this->refreshInterval = $this->makeSetting('refreshInterval', 30, FieldConfig::TYPE_INT, function (FieldConfig $field) {
            $field->title = Piwik::translate('{PluginName}_RefreshInterval');
            $field->uiControl = FieldConfig::UI_CONTROL_TEXT;
            $field->validators[] = new \Piwik\Validators\NumberRange(1, 3600);
        });
    }
}
```

Settings can also be configured via `config/config.ini.php`:
```ini
[MyPlugin]
refreshInterval = 60
```
When set in config, the setting becomes read-only in the UI.

### Controller Class

```php
<?php

namespace Piwik\Plugins\{PluginName};

use Piwik\Nonce;
use Piwik\Piwik;
use Piwik\View;

class Controller extends \Piwik\Plugin\ControllerAdmin
{
    public function index(): string
    {
        Piwik::checkUserHasSuperUserAccess();

        $view = new View('@{PluginName}/index');
        $view->settings = new Settings();
        $view->nonce = Nonce::getNonce('{PluginName}.save');
        $this->setGeneralVariablesView($view);

        return $view->render();
    }

    public function save(): void
    {
        Piwik::checkUserHasSuperUserAccess();
        $this->checkTokenInUrl();

        $nonce = \Piwik\Common::getRequestVar('nonce', '', 'string');
        Nonce::checkNonce('{PluginName}.save', $nonce);

        // ... save logic ...

        $this->redirectToIndex('{PluginName}', 'index');
    }
}
```

### Menu Class

```php
<?php

namespace Piwik\Plugins\{PluginName};

use Piwik\Menu\MenuAdmin;
use Piwik\Piwik;

class Menu extends \Piwik\Plugin\Menu
{
    public function configureAdminMenu(MenuAdmin $menu): void
    {
        if (Piwik::hasUserSuperUserAccess()) {
            $menu->addSystemItem(
                Piwik::translate('{PluginName}_MenuTitle'),
                $this->urlForAction('index'),
                $order = 30
            );
        }
    }
}
```

### Twig Template

```twig
{# templates/index.twig #}
{% extends 'admin.twig' %}

{% block content %}
<div class="card">
    <div class="card-content">
        <h2 class="card-title">{{ '{PluginName}_PageTitle'|translate }}</h2>
        <p>{{ '{PluginName}_PageDescription'|translate }}</p>
    </div>
</div>
{% endblock %}
```

### Scheduled Tasks

```php
<?php

namespace Piwik\Plugins\{PluginName};

use Piwik\Plugin\Tasks as PluginTasks;

class Tasks extends PluginTasks
{
    public function schedule(): void
    {
        $this->hourly('runHourlyTask');
        $this->daily('runDailyTask');
        // Also: $this->weekly(), $this->monthly()
    }

    public function runHourlyTask(): void
    {
        // ... hourly logic
    }

    public function runDailyTask(): void
    {
        // ... daily logic
    }
}
```

---

## Vue 3 Components

Matomo 5.x uses Vue 3 for frontend components. Vue source lives in `vue/src/`
and is compiled to `vue/dist/` via `./console vue:build {PluginName}`.

### File structure

```
PluginName/
├── vue/
│   ├── src/
│   │   ├── index.ts              # Entry point — registers all exports
│   │   ├── MyComponent/
│   │   │   ├── MyComponent.vue   # Component definition
│   │   │   └── MyComponent.adapter.ts  # Optional: bridge to PHP/Twig
│   │   └── types.ts              # TypeScript interfaces
│   └── dist/
│       └── {PluginName}.umd.js   # Compiled bundle (auto-generated)
```

### Vue component example

```vue
<!-- vue/src/ExampleWidget/ExampleWidget.vue -->
<template>
  <div class="exampleWidget">
    <h3>{{ title }}</h3>
    <table v-if="data.length">
      <thead>
        <tr>
          <th>{{ translate('General_Label') }}</th>
          <th>{{ translate('General_ColumnNbVisits') }}</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="row in data" :key="row.label">
          <td>{{ row.label }}</td>
          <td>{{ row.nb_visits }}</td>
        </tr>
      </tbody>
    </table>
    <p v-else>{{ translate('CoreHome_NoData') }}</p>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { translate } from 'CoreHome';

export default defineComponent({
  props: {
    title: { type: String, default: '' },
    data: { type: Array, default: () => [] },
  },
  methods: {
    translate,
  },
});
</script>
```

### Entry point (index.ts)

```typescript
// vue/src/index.ts
export { default as ExampleWidget } from './ExampleWidget/ExampleWidget.vue';
```

### Using Vue components in Twig

```twig
{# Load the component via Matomo's Vue bridge #}
<div vue-entry="{PluginName}.ExampleWidget"
     title="{{ title|e('html_attr') }}"
     data="{{ dataJson|e('html_attr') }}">
</div>
```

Set `dataJson` in your Controller:
```php
$view->dataJson = json_encode($data);
```

### Building Vue components

```bash
# Development (watch mode)
./console vue:build {PluginName} --watch

# Production build
./console vue:build {PluginName}
```

---

## Event Hook Reference

Key events organized by category. Subscribe in `registerEvents()`.

### Tracking
| Event | When fired | Common use |
|-------|-----------|-----------|
| `Tracker.newConversionInformation` | Goal conversion recorded | Enrich conversion data |
| `Tracker.existingVisitInformation` | Existing visit updated | Modify visit attributes |
| `Tracker.end` | Tracking request completes | Cleanup, notifications |

### Reporting & Archiving
| Event | When fired | Common use |
|-------|-----------|-----------|
| `API.Request.dispatch` | Before any API call | Intercept, modify, block |
| `API.Request.dispatch.end` | After any API call | Post-process results |
| `Archiver.addRecordBuilders` | RecordBuilders collected | Register parameterized builders |
| `ViewDataTable.configure` | Report visualization init | Customize display |

### UI & Assets
| Event | When fired | Common use |
|-------|-----------|-----------|
| `AssetManager.getJavaScriptFiles` | JS assets collected | Register JS files |
| `AssetManager.getStylesheetFiles` | CSS assets collected | Register LESS/CSS |
| `Translate.getClientSideTranslationKeys` | JS translations | Expose keys to frontend |
| `Template.beforeTopBar` | Before top nav | Inject UI elements |
| `Template.jsGlobalVariables` | JS globals set | Inject config to JS |

### System & Lifecycle
| Event | When fired | Common use |
|-------|-----------|-----------|
| `Console.filterCommands` | CLI commands loaded | Register commands |
| `CronArchive.archiveSingleSite.finish` | Site archiving done | Post-archive tasks |
| `PluginManager.pluginActivated` | Plugin activated | Init, migrate |
| `Access.Capability.addCapabilities` | Capabilities loaded | Custom capabilities |
