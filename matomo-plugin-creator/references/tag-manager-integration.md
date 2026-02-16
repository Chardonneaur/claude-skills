# Tag Manager Integration Reference

How to create custom tags, triggers, and variables for Matomo Tag Manager from external plugins.
Based on real patterns from the TagManager plugin's template system.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Creating a Custom Tag](#creating-a-custom-tag)
3. [Creating a Custom Trigger](#creating-a-custom-trigger)
4. [Creating a Custom Variable](#creating-a-custom-variable)
5. [Parameter Definition](#parameter-definition)
6. [Frontend Implementation](#frontend-implementation)
7. [Event Hooks](#event-hooks)
8. [Category Constants](#category-constants)
9. [Complete Working Example](#complete-working-example)

---

## Architecture Overview

The Tag Manager uses a template system with auto-discovery and event-based registration:

- **Tags** execute code when fired by triggers (analytics, ads, custom HTML)
- **Triggers** define when tags should fire (page view, click, form submit)
- **Variables** provide dynamic data to tags and trigger conditions (cookies, URLs, DOM)

All templates extend `BaseTemplate` which provides parameter management, translation
support, and frontend template loading.

### Plugin structure for TM extensions

```
MyPlugin/
├── MyPlugin.php                          ← Event registration
├── Template/
│   ├── Tag/
│   │   ├── MyCustomTag.php              ← PHP tag definition
│   │   └── MyCustomTag.web.js           ← Frontend implementation
│   ├── Trigger/
│   │   ├── MyCustomTrigger.php
│   │   └── MyCustomTrigger.web.js
│   └── Variable/
│       ├── MyCustomVariable.php
│       └── MyCustomVariable.web.js
└── lang/
    └── en.json                           ← Translation keys
```

---

## Creating a Custom Tag

Extend `BaseTag` and implement the required methods:

```php
<?php

namespace Piwik\Plugins\{PluginName}\Template\Tag;

use Piwik\Piwik;
use Piwik\Plugins\TagManager\Template\Tag\BaseTag;
use Piwik\Settings\FieldConfig;
use Piwik\Validators\NotEmpty;

class MyAnalyticsTag extends BaseTag
{
    public function getCategory(): string
    {
        return self::CATEGORY_ANALYTICS;
    }

    public function getIcon(): string
    {
        return 'plugins/{PluginName}/images/icon.svg';
    }

    public function getParameters(): array
    {
        return [
            $this->makeSetting('trackingId', '', FieldConfig::TYPE_STRING, function (FieldConfig $field) {
                $field->title = Piwik::translate('{PluginName}_TrackingIdTitle');
                $field->description = Piwik::translate('{PluginName}_TrackingIdDescription');
                $field->validators[] = new NotEmpty();
                $field->customFieldComponent = self::FIELD_VARIABLE_COMPONENT;
            }),
            $this->makeSetting('trackPageview', true, FieldConfig::TYPE_BOOL, function (FieldConfig $field) {
                $field->title = Piwik::translate('{PluginName}_TrackPageviewTitle');
                $field->uiControl = FieldConfig::UI_CONTROL_CHECKBOX;
            }),
        ];
    }
}
```

### Key methods

| Method | Required | Description |
|--------|----------|-------------|
| `getCategory()` | Yes | Return a category constant |
| `getParameters()` | Yes | Return array of `Setting` objects |
| `getIcon()` | No | Path to SVG/PNG icon |
| `getId()` | No | Override auto-generated ID |
| `getOrder()` | No | Sort order (default: 9999) |
| `hasAdvancedSettings()` | No | Show advanced tab (default: true) |
| `isCustomTemplate()` | No | Mark as user-editable (default: false) |

---

## Creating a Custom Trigger

Extend `BaseTrigger`:

```php
<?php

namespace Piwik\Plugins\{PluginName}\Template\Trigger;

use Piwik\Piwik;
use Piwik\Plugins\TagManager\Template\Trigger\BaseTrigger;
use Piwik\Settings\FieldConfig;

class MyCustomTrigger extends BaseTrigger
{
    public function getCategory(): string
    {
        return self::CATEGORY_USER_ENGAGEMENT;
    }

    public function getIcon(): string
    {
        return 'plugins/{PluginName}/images/trigger-icon.svg';
    }

    public function getParameters(): array
    {
        return [
            $this->makeSetting('selector', '', FieldConfig::TYPE_STRING, function (FieldConfig $field) {
                $field->title = Piwik::translate('{PluginName}_CssSelectorTitle');
                $field->description = Piwik::translate('{PluginName}_CssSelectorDescription');
                $field->customFieldComponent = self::FIELD_VARIABLE_COMPONENT;
            }),
        ];
    }
}
```

### Simple trigger (no parameters)

```php
class PageViewTrigger extends BaseTrigger
{
    public function getCategory(): string
    {
        return self::CATEGORY_PAGE_VIEW;
    }

    public function getParameters(): array
    {
        return [];
    }
}
```

---

## Creating a Custom Variable

Extend `BaseVariable`:

```php
<?php

namespace Piwik\Plugins\{PluginName}\Template\Variable;

use Piwik\Piwik;
use Piwik\Plugins\TagManager\Template\Variable\BaseVariable;
use Piwik\Settings\FieldConfig;
use Piwik\Validators\NotEmpty;
use Piwik\Validators\CharacterLength;

class MyCustomVariable extends BaseVariable
{
    public function getCategory(): string
    {
        return self::CATEGORY_PAGE_VARIABLES;
    }

    public function getIcon(): string
    {
        return 'plugins/{PluginName}/images/variable-icon.svg';
    }

    public function getParameters(): array
    {
        return [
            $this->makeSetting('attributeName', '', FieldConfig::TYPE_STRING, function (FieldConfig $field) {
                $field->title = Piwik::translate('{PluginName}_AttributeNameTitle');
                $field->validators[] = new NotEmpty();
                $field->validators[] = new CharacterLength(1, 500);
            }),
            $this->makeSetting('urlDecode', false, FieldConfig::TYPE_BOOL, function (FieldConfig $field) {
                $field->title = Piwik::translate('{PluginName}_UrlDecodeTitle');
            }),
        ];
    }
}
```

### Pre-configured variables

Variables that are always available without user configuration:

```php
class ReferrerUrlVariable extends BaseVariable
{
    public function isPreConfigured(): bool
    {
        return true;  // Read-only, always available
    }

    public function getCategory(): string
    {
        return self::CATEGORY_PAGE_VARIABLES;
    }

    public function getParameters(): array
    {
        return [];  // No configuration needed
    }
}
```

---

## Parameter Definition

Parameters use the `makeSetting()` method from `BaseTemplate`:

```php
$this->makeSetting(
    'paramName',           // Name (alphanumeric + underscore only)
    'defaultValue',        // Default value
    FieldConfig::TYPE_*,   // Type constant
    function(FieldConfig $field) {
        // Configure the field
    }
)
```

### Field configuration options

```php
function (FieldConfig $field) {
    // Display
    $field->title = Piwik::translate('TranslationKey');
    $field->description = 'Longer description text';
    $field->inlineHelp = 'Additional help shown inline';

    // UI control type
    $field->uiControl = FieldConfig::UI_CONTROL_TEXT;       // Single-line text
    $field->uiControl = FieldConfig::UI_CONTROL_TEXTAREA;   // Multi-line text
    $field->uiControl = FieldConfig::UI_CONTROL_CHECKBOX;   // Boolean checkbox
    $field->uiControl = FieldConfig::UI_CONTROL_SINGLE_SELECT; // Dropdown
    $field->uiControl = FieldConfig::UI_CONTROL_MULTI_TUPLE;   // Key-value pairs

    // Custom Tag Manager components
    $field->customFieldComponent = self::FIELD_VARIABLE_COMPONENT;          // Variable picker
    $field->customFieldComponent = self::FIELD_TEXTAREA_VARIABLE_COMPONENT; // Textarea with variables
    $field->customFieldComponent = self::FIELD_VARIABLE_TYPE_COMPONENT;     // Variable type selector

    // Dropdown values
    $field->availableValues = [
        'option1' => 'Label 1',
        'option2' => 'Label 2',
    ];

    // Conditional visibility
    $field->condition = 'trackingType == "event"';

    // Validation
    $field->validators[] = new NotEmpty();
    $field->validators[] = new CharacterLength(1, 500);
    $field->validate = function ($value) {
        if ($value && !is_numeric($value)) {
            throw new \Exception('Value must be numeric');
        }
    };

    // Transformation
    $field->transform = function ($value) {
        return trim($value);
    };

    // Advanced settings grouping
    $field->uiControlAttributes['showAdvancedSettings'] = 1;
}
```

### FieldConfig type constants

| Constant | PHP Type |
|----------|----------|
| `FieldConfig::TYPE_STRING` | `string` |
| `FieldConfig::TYPE_INT` | `int` |
| `FieldConfig::TYPE_BOOL` | `bool` |
| `FieldConfig::TYPE_ARRAY` | `array` |

---

## Frontend Implementation

Each template needs a corresponding JavaScript file that implements the actual browser-side
behavior.

### Naming convention

The JS file must match the PHP class name:

| PHP class | JS file |
|-----------|---------|
| `MyCustomTag.php` | `MyCustomTag.web.js` |
| `MyCustomTrigger.php` | `MyCustomTrigger.web.js` |
| `MyCustomVariable.php` | `MyCustomVariable.web.js` |

### Tag frontend (web.js)

```javascript
(function () {
    return function (parameters, TagManager) {
        this.fire = function () {
            var trackingId = parameters.get('trackingId');
            var trackPageview = parameters.get('trackPageview');

            // Your tag implementation
            if (trackPageview) {
                // Send tracking data
            }
        };
    };
})();
```

### Trigger frontend (web.js)

```javascript
(function () {
    return function (parameters, TagManager) {
        this.setUp = function (triggerEvent) {
            var selector = parameters.get('selector');

            // Set up event listener
            TagManager.dom.addEventListener(document, 'click', function (event) {
                if (event.target.matches(selector)) {
                    triggerEvent({ event: 'MyCustomTrigger', element: event.target });
                }
            });
        };
    };
})();
```

### Variable frontend (web.js)

```javascript
(function () {
    return function (parameters, TagManager) {
        this.get = function () {
            var attrName = parameters.get('attributeName');
            var urlDecode = parameters.get('urlDecode');

            var value = document.querySelector('[' + attrName + ']');
            if (value && urlDecode) {
                value = decodeURIComponent(value);
            }
            return value || '';
        };
    };
})();
```

---

## Event Hooks

Register templates from external plugins via events in your main plugin class:

```php
<?php

namespace Piwik\Plugins\{PluginName};

use Piwik\Plugin;

class {PluginName} extends Plugin
{
    public function registerEvents(): array
    {
        return [
            'TagManager.addTags'      => 'addTags',
            'TagManager.addTriggers'  => 'addTriggers',
            'TagManager.addVariables' => 'addVariables',
        ];
    }

    public function addTags(array &$tags): void
    {
        $tags[] = new Template\Tag\MyCustomTag();
    }

    public function addTriggers(array &$triggers): void
    {
        $triggers[] = new Template\Trigger\MyCustomTrigger();
    }

    public function addVariables(array &$variables): void
    {
        $variables[] = new Template\Variable\MyCustomVariable();
    }
}
```

### Filtering (removing/modifying) existing templates

```php
public function registerEvents(): array
{
    return [
        'TagManager.filterTags'      => 'filterTags',
        'TagManager.filterTriggers'  => 'filterTriggers',
        'TagManager.filterVariables' => 'filterVariables',
    ];
}

public function filterTags(array &$tags): void
{
    // Remove a specific tag
    foreach ($tags as $index => $tag) {
        if ($tag->getId() === 'CustomHtml') {
            unset($tags[$index]);
        }
    }
}
```

### Complete event reference

| Event | Parameter | Purpose |
|-------|-----------|---------|
| `TagManager.addTags` | `array &$tags` | Add custom tag templates |
| `TagManager.addTriggers` | `array &$triggers` | Add custom trigger templates |
| `TagManager.addVariables` | `array &$variables` | Add custom variable templates |
| `TagManager.filterTags` | `array &$tags` | Modify/remove tag templates |
| `TagManager.filterTriggers` | `array &$triggers` | Modify/remove trigger templates |
| `TagManager.filterVariables` | `array &$variables` | Modify/remove variable templates |

---

## Category Constants

### Tag categories (`BaseTag`)

| Constant | Description |
|----------|-------------|
| `CATEGORY_ANALYTICS` | Analytics platforms (Matomo, GA) |
| `CATEGORY_CUSTOM` | Custom HTML/JS |
| `CATEGORY_DEVELOPERS` | Developer tools |
| `CATEGORY_ADS` | Ad platforms |
| `CATEGORY_EMAIL` | Email marketing |
| `CATEGORY_AFFILIATES` | Affiliate tracking |
| `CATEGORY_REMARKETING` | Retargeting |
| `CATEGORY_SOCIAL` | Social media |
| `CATEGORY_OTHERS` | Miscellaneous |

### Trigger categories (`BaseTrigger`)

| Constant | Description |
|----------|-------------|
| `CATEGORY_PAGE_VIEW` | Page view events |
| `CATEGORY_CLICK` | Click events |
| `CATEGORY_USER_ENGAGEMENT` | Form, scroll, visibility |
| `CATEGORY_OTHERS` | Miscellaneous |

### Variable categories (`BaseVariable`)

| Constant | Description |
|----------|-------------|
| `CATEGORY_PAGE_VARIABLES` | URL, page data |
| `CATEGORY_CLICKS` | Click data |
| `CATEGORY_CONTAINER_INFO` | Container metadata |
| `CATEGORY_HISTORY` | Browser history |
| `CATEGORY_ERRORS` | JavaScript errors |
| `CATEGORY_SCROLLS` | Scroll tracking |
| `CATEGORY_FORMS` | Form data |
| `CATEGORY_DATE` | Date/time |
| `CATEGORY_PERFORMANCE` | Performance metrics |
| `CATEGORY_UTILITIES` | Utility functions |
| `CATEGORY_DEVICE` | Device info |
| `CATEGORY_SEO` | SEO data |
| `CATEGORY_ANALYTICS` | Analytics configuration |
| `CATEGORY_VISIBILITY` | Visibility events |
| `CATEGORY_OTHERS` | Miscellaneous |

---

## Complete Working Example

### Custom analytics tag with frontend

**PHP definition** (`Template/Tag/SimpleAnalyticsTag.php`):

```php
<?php

namespace Piwik\Plugins\{PluginName}\Template\Tag;

use Piwik\Piwik;
use Piwik\Plugins\TagManager\Template\Tag\BaseTag;
use Piwik\Settings\FieldConfig;
use Piwik\Validators\NotEmpty;

class SimpleAnalyticsTag extends BaseTag
{
    public const ID = 'SimpleAnalytics';

    public function getId(): string
    {
        return self::ID;
    }

    public function getCategory(): string
    {
        return self::CATEGORY_ANALYTICS;
    }

    public function getIcon(): string
    {
        return 'plugins/{PluginName}/images/analytics.svg';
    }

    public function getParameters(): array
    {
        return [
            $this->makeSetting('endpointUrl', '', FieldConfig::TYPE_STRING, function (FieldConfig $field) {
                $field->title = Piwik::translate('{PluginName}_EndpointUrlTitle');
                $field->description = Piwik::translate('{PluginName}_EndpointUrlDescription');
                $field->validators[] = new NotEmpty();
                $field->customFieldComponent = self::FIELD_VARIABLE_COMPONENT;
            }),
            $this->makeSetting('sendPageview', true, FieldConfig::TYPE_BOOL, function (FieldConfig $field) {
                $field->title = Piwik::translate('{PluginName}_SendPageviewTitle');
            }),
        ];
    }
}
```

**Frontend** (`Template/Tag/SimpleAnalyticsTag.web.js`):

```javascript
(function () {
    return function (parameters, TagManager) {
        this.fire = function () {
            var endpointUrl = parameters.get('endpointUrl');
            var sendPageview = parameters.get('sendPageview');

            if (!endpointUrl) {
                return;
            }

            if (sendPageview) {
                var img = new Image(1, 1);
                img.src = endpointUrl + '?url=' + encodeURIComponent(window.location.href)
                    + '&title=' + encodeURIComponent(document.title)
                    + '&referrer=' + encodeURIComponent(document.referrer);
            }
        };
    };
})();
```

**Plugin registration** (`{PluginName}.php`):

```php
public function registerEvents(): array
{
    return [
        'TagManager.addTags' => 'addTags',
    ];
}

public function addTags(array &$tags): void
{
    $tags[] = new Template\Tag\SimpleAnalyticsTag();
}
```

**Translations** (`lang/en.json`):

```json
{
    "{PluginName}_SimpleAnalyticsTagName": "Simple Analytics",
    "{PluginName}_SimpleAnalyticsTagDescription": "Sends page views to a custom analytics endpoint.",
    "{PluginName}_EndpointUrlTitle": "Endpoint URL",
    "{PluginName}_EndpointUrlDescription": "The URL where analytics data will be sent.",
    "{PluginName}_SendPageviewTitle": "Send Pageview"
}
```

### Translation key pattern

Tag Manager auto-generates translation keys from the template ID:

| Template type | Key pattern |
|---------------|-------------|
| Tag | `{PluginName}_{TemplateId}TagName`, `{PluginName}_{TemplateId}TagDescription` |
| Trigger | `{PluginName}_{TemplateId}TriggerName`, `{PluginName}_{TemplateId}TriggerDescription` |
| Variable | `{PluginName}_{TemplateId}VariableName`, `{PluginName}_{TemplateId}VariableDescription` |

### Reserved setting names

These names cannot be used for parameters: `container`, `tag`, `variable`, `trigger`,
`fire_limit`, `fire_delay`, `priority`, `start_date`, `end_date`.
