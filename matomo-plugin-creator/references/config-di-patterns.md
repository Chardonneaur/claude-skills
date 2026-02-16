# DI Container & Config Patterns Reference

How to register services, configure dependencies, and use the DI container in Matomo plugins.
Based on real patterns from Monolog, TagManager, Login, CoreHome, and other core plugins.

## Table of Contents

1. [File Location & Structure](#file-location--structure)
2. [DI Helper Methods](#di-helper-methods)
3. [Service Registration with Constructor Params](#service-registration-with-constructor-params)
4. [Autowiring with Parameter Overrides](#autowiring-with-parameter-overrides)
5. [Factory Pattern](#factory-pattern)
6. [Array Merging with DI::add()](#array-merging-with-diadd)
7. [Decorator Pattern](#decorator-pattern)
8. [Interface Binding & Aliases](#interface-binding--aliases)
9. [Container Access in Closures](#container-access-in-closures)
10. [When to Use config.php vs StaticContainer](#when-to-use-configphp-vs-staticcontainer)

---

## File Location & Structure

DI configuration lives in `config/config.php` within your plugin directory. It returns a PHP
array mapping service names to their definitions.

```php
<?php
// plugins/MyPlugin/config/config.php

use Piwik\DI;

return [
    // Service definitions go here
];
```

Matomo auto-discovers this file when the plugin is loaded. No registration needed.

### Minimal example (empty config)

```php
<?php

return [];
```

### Single service

```php
<?php

return [
    'Piwik\Plugins\MyPlugin\MyService' => Piwik\DI::create(),
];
```

---

## DI Helper Methods

All methods are available as static methods on `Piwik\DI`. They wrap the PHP-DI library.

| Method | Return Type | Purpose |
|--------|-------------|---------|
| `DI::create(?string $className)` | `CreateDefinitionHelper` | Explicit object instantiation |
| `DI::autowire(?string $className)` | `AutowireDefinitionHelper` | Auto-discover & inject dependencies |
| `DI::factory($callable)` | `FactoryDefinitionHelper` | Lazy computation via callable |
| `DI::decorate($callable)` | `FactoryDefinitionHelper` | Wrap/decorate an existing service |
| `DI::get(string $entryName)` | `Reference` | Reference another container entry |
| `DI::value($value)` | `ValueDefinition` | Wrap a raw value |
| `DI::add($values)` | `ArrayDefinitionExtension` | Extend an existing array definition |
| `DI::string(string $expression)` | `StringDefinition` | Template string with `{}` placeholders |
| `DI::env(string $varName, $default)` | `EnvironmentVariableDefinition` | Access environment variables |

### Method chaining on definition helpers

Both `create()` and `autowire()` support chaining:

```php
Piwik\DI::create(MyClass::class)
    ->constructor($arg1, $arg2)          // Set constructor arguments
    ->method('setOption', $option)        // Call setter after construction
    ->method('setLogger', Piwik\DI::get('logger'))
```

```php
Piwik\DI::autowire(MyClass::class)
    ->constructorParameter('paramName', $value)  // Override specific param
    ->method('init')                              // Call method after construction
```

---

## Service Registration with Constructor Params

Use `DI::create()` with `->constructor()` to pass explicit constructor arguments.

```php
<?php
// From Monolog plugin

use Piwik\Log\Logger;

return [
    // Pass named channel, handlers array, and processors array to Logger constructor
    Logger::class => Piwik\DI::create(Logger::class)
        ->constructor('piwik', Piwik\DI::get('log.handlers'), Piwik\DI::get('log.processors')),

    // Constructor + setter method chaining
    'Piwik\Plugins\Monolog\Handler\FileHandler' => Piwik\DI::create()
        ->constructor(Piwik\DI::get('log.file.filename'), Piwik\DI::get('log.level.file'))
        ->method('setFormatter', Piwik\DI::get('log.lineMessageFormatter.file')),
];
```

**Key points:**
- When `create()` has no className argument, the container entry key is used as the class name
- `DI::get()` injects other container entries as constructor arguments
- `->method()` calls setter methods after instantiation; can be chained multiple times

---

## Autowiring with Parameter Overrides

Use `DI::autowire()` when a class has many dependencies but you only need to override
specific constructor parameters. PHP-DI resolves the rest automatically.

```php
<?php
// From Monolog plugin — override specific constructor params

return [
    'Piwik\Plugins\Monolog\Handler\ErrorLogHandler' => Piwik\DI::autowire()
        ->constructorParameter('level', Piwik\DI::get('log.level.errorlog'))
        ->method('setFormatter', Piwik\DI::get('log.lineMessageFormatter.file')),

    'Piwik\Plugins\Monolog\Handler\SyslogHandler' => Piwik\DI::autowire()
        ->constructorParameter('ident', Piwik\DI::get('log.syslog.ident'))
        ->constructorParameter('level', Piwik\DI::get('log.level.syslog'))
        ->method('setFormatter', Piwik\DI::get('log.lineMessageFormatter.file')),
];
```

```php
<?php
// From CoreHome plugin — override config-driven params

return [
    'Piwik\Plugins\CoreHome\Tracker\VisitRequestProcessor' => Piwik\DI::autowire()
        ->constructorParameter('visitStandardLength', Piwik\DI::get('ini.Tracker.visit_standard_length'))
        ->constructorParameter('trackerAlwaysNewVisitor', Piwik\DI::get('ini.Debug.tracker_always_new_visitor')),
];
```

**When to use `autowire()` vs `create()`:**
- Use `autowire()` when the class has many type-hinted dependencies and you only override a few
- Use `create()` when you want full control over every constructor argument
- `autowire()` is more resilient to constructor signature changes

---

## Factory Pattern

Use `DI::factory()` for complex initialization logic, conditional service creation, or
computed values.

```php
<?php
// From Monolog plugin — conditional handler selection

use Piwik\Container\Container;
use Piwik\Log;

return [
    // Factory with conditional logic based on config
    'log.handlers' => Piwik\DI::factory(function (Container $c) {
        if ($c->has('ini.log.log_writers')) {
            $writerNames = $c->get('ini.log.log_writers');
        } else {
            return [];
        }

        $classes = $c->get('log.handler.classes');
        $writers = [];

        foreach ($writerNames as $writerName) {
            if (isset($classes[$writerName])) {
                $writers[$writerName] = $c->get($classes[$writerName]);
            }
        }

        return array_values($writers);
    }),

    // Factory for computed config values
    'log.level' => Piwik\DI::factory(function (Container $c) {
        if ($c->has('ini.log.log_level')) {
            $level = strtoupper($c->get('ini.log.log_level'));
            if (!empty($level) && defined('Piwik\Log::' . $level)) {
                return Log::getMonologLevel(constant('Piwik\Log::' . $level));
            }
        }
        return \Monolog\Logger::WARNING;
    }),

    // Cascading factory — falls back to another factory
    'log.level.file' => Piwik\DI::factory(function (Container $c) {
        if ($c->has('ini.log.log_level_file')) {
            $level = Log::getMonologLevelIfValid($c->get('ini.log.log_level_file'));
            if ($level !== null) {
                return $level;
            }
        }
        return $c->get('log.level'); // fallback to parent
    }),
];
```

### Simple callable (no DI::factory wrapper needed)

For straightforward values, you can use plain closures:

```php
<?php
// From TagManager plugin

return [
    'TagManagerContainerStorageDir' => function () {
        return '/js';
    },

    'TagManagerContainerWebDir' => function (\Piwik\Container\Container $c) {
        return $c->get('TagManagerContainerStorageDir');
    },

    // Scalar values need no wrapper at all
    'TagManagerJSMinificationEnabled' => true,
];
```

**Key points:**
- Factories execute lazily — only when the service is first requested
- Container (`$c`) is injected if the closure type-hints `Container` as first parameter
- Use `$c->has()` to safely check if an entry exists before `$c->get()`
- Factories can reference other factories via `$c->get()`

---

## Array Merging with DI::add()

Use `DI::add()` to extend array definitions from other plugins without replacing them.
This is the standard way for plugins to contribute items to shared arrays.

```php
<?php
// From TagManager plugin — extend diagnostics and file integrity arrays

return [
    'diagnostics.required' => Piwik\DI::add([
        Piwik\DI::get('Piwik\Plugins\TagManager\Diagnostic\ContainerWriteAccess'),
    ]),

    'fileintegrity.ignore' => Piwik\DI::add([
        Piwik\DI::get('fileintegrityIgnoreTagManager'),
    ]),
];
```

**Key points:**
- `DI::add()` EXTENDS the existing array — does NOT replace it
- Items can be `DI::get()` references, values, or factories
- Essential for plugin contributions to shared registries (diagnostics, file integrity, etc.)

---

## Decorator Pattern

Use `DI::decorate()` to wrap an existing service with additional behavior.

```php
<?php
// Decorate SecurityPolicy to add CSP headers for your plugin

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

**Signature:** The decorator callable receives:
1. `$previous` — the existing service instance being decorated
2. `$container` (optional) — the DI container

```php
// With container access
SomeService::class => Piwik\DI::decorate(function ($previous, \Piwik\Container\Container $c) {
    $config = $c->get('Piwik\Config');
    // Wrap $previous with additional behavior
    return new DecoratedService($previous, $config);
}),
```

---

## Interface Binding & Aliases

Bind interfaces to implementations, or create backward-compatibility aliases:

```php
<?php
// From Monolog and Login plugins

return [
    // Interface binding
    'Piwik\Auth' => Piwik\DI::create('Piwik\Plugins\Login\Auth'),

    // Interface to concrete class reference
    \Piwik\Log\LoggerInterface::class => Piwik\DI::get(\Monolog\Logger::class),

    // Backward-compatibility alias
    'Psr\Log\LoggerInterface' => Piwik\DI::get(\Piwik\Log\LoggerInterface::class),

    // Bind interface to autowired implementation
    'Piwik\Plugins\TagManager\Model\Container\ContainerIdGenerator' =>
        Piwik\DI::autowire('Piwik\Plugins\TagManager\Model\Container\RandomContainerIdGenerator'),
];
```

### Arrays of service references

```php
<?php
// From Monolog plugin — array of processor references

return [
    'log.processors' => [
        Piwik\DI::get('Piwik\Plugins\Monolog\Processor\SprintfProcessor'),
        Piwik\DI::get('Piwik\Plugins\Monolog\Processor\ClassNameProcessor'),
        Piwik\DI::get('Piwik\Plugins\Monolog\Processor\RequestIdProcessor'),
        Piwik\DI::get('Piwik\Plugins\Monolog\Processor\ExceptionToTextProcessor'),
    ],

    // Configuration map
    'log.handler.classes' => [
        'file'     => 'Piwik\Plugins\Monolog\Handler\FileHandler',
        'screen'   => 'Piwik\Plugins\Monolog\Handler\WebNotificationHandler',
        'database' => 'Piwik\Plugins\Monolog\Handler\DatabaseHandler',
    ],
];
```

---

## Container Access in Closures

Factories and decorators can access the DI container to resolve other services at runtime.

### INI configuration access pattern

Matomo exposes `config.ini.php` values via the `ini.SectionName.keyName` convention:

```php
// Check and retrieve INI config values
$c->has('ini.log.log_writers')           // Check if entry exists
$c->get('ini.log.log_writers')           // Get array from [log] log_writers[]
$c->get('ini.log.log_level')             // Get scalar from [log] log_level
$c->get('ini.Tracker.visit_standard_length')  // Get from [Tracker] section
$c->get('ini.Debug.tracker_always_new_visitor')
```

### Accessing Config service directly

```php
'my.service' => Piwik\DI::factory(function (\Piwik\Container\Container $c) {
    $logConfig = $c->get(\Piwik\Config::class)->log;
    $enabled = isset($logConfig['enable_feature']) && $logConfig['enable_feature'] == 1;
    return new MyService($enabled);
}),
```

---

## When to Use config.php vs StaticContainer

### Use `config/config.php` when:
- Registering services that should be available globally
- Binding interfaces to implementations
- Extending shared arrays (diagnostics, file integrity)
- Decorating core services (SecurityPolicy, Auth, etc.)
- Defining configuration values that other services depend on

### Use `StaticContainer::get()` when:
- Accessing services at runtime inside plugin code
- Services are needed in static contexts (event handlers, callbacks)
- You need a one-off service resolution

```php
// Runtime access in plugin code
use Piwik\Container\StaticContainer;

$myService = StaticContainer::get('Piwik\Plugins\MyPlugin\MyService');

// In factories that need other services during configuration
'my.service' => Piwik\DI::factory(function () {
    $settings = StaticContainer::get('Piwik\Plugins\MyPlugin\SystemSettings');
    return new MyService($settings->apiKey->getValue());
}),
```

**Prefer constructor injection** (via `config/config.php`) over `StaticContainer::get()` when
possible. `StaticContainer` should be a last resort for cases where DI is not available.
