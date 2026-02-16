# Testing Patterns Reference

How to write unit, integration, system, and frontend tests for Matomo plugins.
Based on real patterns from core plugins and the Matomo contributor coding guidelines.

## Table of Contents

1. [Directory Structure](#directory-structure)
2. [Unit Tests](#unit-tests)
3. [Integration Tests](#integration-tests)
4. [System Tests](#system-tests)
5. [Vue/TypeScript Tests](#vuetypescript-tests)
6. [Running Tests](#running-tests)
7. [Test Naming Conventions](#test-naming-conventions)
8. [Best Practices](#best-practices)

---

## Directory Structure

```
PluginName/
├── tests/
│   ├── Unit/
│   │   └── ModelTest.php
│   ├── Integration/
│   │   └── APITest.php
│   ├── System/
│   │   └── ApiTest.php
│   └── Fixtures/
│       └── SomeFixture.php
├── vue/src/
│   └── **/*.spec.ts           ← Vue/TS tests live alongside source
```

---

## Unit Tests

Unit tests verify isolated logic without database or framework dependencies.

**Base class:** `PHPUnit\Framework\TestCase`

```php
<?php

namespace Piwik\Plugins\{PluginName}\Tests\Unit;

use PHPUnit\Framework\TestCase;
use Piwik\Plugins\{PluginName}\Model;

class ModelTest extends TestCase
{
    private Model $model;

    protected function setUp(): void
    {
        parent::setUp();
        $this->model = new Model();
    }

    public function test_validateName_throwsOnEmptyName(): void
    {
        $this->expectException(\Exception::class);
        $this->model->validateName('');
    }

    public function test_validateName_acceptsValidName(): void
    {
        $result = $this->model->validateName('My Name');
        $this->assertEquals('My Name', $result);
    }

    /**
     * @dataProvider provideValidNames
     */
    public function test_validateName_handlesEdgeCases(string $input, string $expected): void
    {
        $result = $this->model->validateName($input);
        $this->assertEquals($expected, $result);
    }

    public function provideValidNames(): array
    {
        return [
            ['Goal Name', 'Goal Name'],
            ['  trimmed  ', 'trimmed'],
            ['special%20chars', 'special chars'],
        ];
    }
}
```

### Key patterns

- Extend `PHPUnit\Framework\TestCase`
- Use `setUp()` to initialize test objects
- Use `@dataProvider` for parameterized tests
- Use `expectException()` for error cases
- No database, no framework initialization needed

---

## Integration Tests

Integration tests verify behavior with the full Matomo framework and database.

**Base class:** `Piwik\Tests\Framework\TestCase\IntegrationTestCase`

```php
<?php

namespace Piwik\Plugins\{PluginName}\Tests\Integration;

use Piwik\Tests\Framework\TestCase\IntegrationTestCase;
use Piwik\Tests\Framework\Fixture;
use Piwik\Plugins\{PluginName}\API;

class APITest extends IntegrationTestCase
{
    private API $api;

    public function setUp(): void
    {
        parent::setUp();

        // Create test site (required for most API methods)
        Fixture::createWebsite('2020-01-01');

        $this->api = API::getInstance();
    }

    public function test_addItem_createsSuccessfully(): void
    {
        $id = $this->api->addItem(1, 'Test Item');

        $this->assertIsInt($id);
        $this->assertGreaterThan(0, $id);

        // Verify it was saved
        $item = $this->api->getItem(1, $id);
        $this->assertEquals('Test Item', $item['name']);
    }

    public function test_getItems_returnsEmptyArrayForNewSite(): void
    {
        $items = $this->api->getItems(1);

        $this->assertIsArray($items);
        $this->assertEmpty($items);
    }

    public function test_deleteItem_removesFromDatabase(): void
    {
        $id = $this->api->addItem(1, 'To Delete');
        $this->api->deleteItem(1, $id);

        $item = $this->api->getItem(1, $id);
        $this->assertNull($item);
    }
}
```

### Key patterns

- Extend `IntegrationTestCase` (sets up full framework + test DB)
- Use `Fixture::createWebsite()` to create test sites
- Access API via `API::getInstance()`
- Database is reset between tests automatically
- Call `parent::setUp()` first

---

## System Tests

System tests verify full request/response cycles including API output format.

**Base class:** `Piwik\Tests\Framework\TestCase\SystemTestCase`

```php
<?php

namespace Piwik\Plugins\{PluginName}\Tests\System;

use Piwik\Tests\Framework\TestCase\SystemTestCase;

class ApiTest extends SystemTestCase
{
    /**
     * @var \Piwik\Plugins\{PluginName}\Tests\Fixtures\SimpleFixture
     */
    public static $fixture = null;

    /**
     * @dataProvider getApiForTesting
     */
    public function testApi($api, $params): void
    {
        $this->runApiTests($api, $params);
    }

    public function getApiForTesting(): array
    {
        return [
            ['{PluginName}.getReport', [
                'idSite'  => self::$fixture->idSite,
                'date'    => self::$fixture->dateTime,
                'periods' => ['day'],
            ]],
        ];
    }
}

ApiTest::$fixture = new \Piwik\Plugins\{PluginName}\Tests\Fixtures\SimpleFixture();
```

### Fixture class

```php
<?php

namespace Piwik\Plugins\{PluginName}\Tests\Fixtures;

use Piwik\Tests\Framework\Fixture;

class SimpleFixture extends Fixture
{
    public int $idSite = 1;
    public string $dateTime = '2023-01-01 00:00:00';

    public function setUp(): void
    {
        $this->setUpWebsitesAndGoals();
        $this->trackVisits();
    }

    private function setUpWebsitesAndGoals(): void
    {
        if (!self::siteCreated($this->idSite)) {
            self::createWebsite($this->dateTime);
        }
    }

    private function trackVisits(): void
    {
        // Use the tracking API to create test data
        $tracker = self::getTracker($this->idSite, $this->dateTime, $defaultInit = true);
        $tracker->setUrl('http://example.com/page1');
        self::checkResponse($tracker->doTrackPageView('Page 1'));
    }
}
```

---

## Vue/TypeScript Tests

Vue component and store tests use Jest with `.spec.ts` files.

### Store test pattern

```typescript
// vue/src/MyStore/MyStore.spec.ts

import { reactive } from 'vue';
import MyStore from './MyStore.store';
import { AjaxHelper } from 'CoreHome';

describe('PluginName/MyStore', () => {
    const mockItems = [
        { id: 1, name: 'Item 1' },
        { id: 2, name: 'Item 2' },
    ];

    function resetStoreState(): void {
        // Reset the singleton store state between tests
        (MyStore as any).privateState = reactive({
            items: [],
            isInitialized: false,
        });
    }

    beforeEach(() => {
        jest.spyOn(AjaxHelper, 'fetch').mockImplementation((params: any) => {
            if (params.method === 'PluginName.getItems') {
                return Promise.resolve(mockItems);
            }
            return Promise.resolve([]);
        });
    });

    afterEach(() => {
        resetStoreState();
        jest.restoreAllMocks();
    });

    it('should load items on first call', async () => {
        const items = await MyStore.loadItems(1);
        expect(items).toEqual(mockItems);
        expect(MyStore.isInitialized.value).toBe(true);
    });

    it('should return cached items on subsequent calls', async () => {
        await MyStore.loadItems(1);
        await MyStore.loadItems(1);
        expect(AjaxHelper.fetch).toHaveBeenCalledTimes(1);
    });
});
```

### Component test pattern

```typescript
// vue/src/Periods/Periods.spec.ts

import Periods from './Periods';
import './Day';
import './Week';

describe('CoreHome/Periods', () => {
    let originalDateNow: (() => number) | null;

    beforeEach(() => {
        originalDateNow = null;
        window.piwik.timezoneOffset = 0;
    });

    afterEach(() => {
        if (originalDateNow) {
            Date.now = originalDateNow;
        }
    });

    it('should get daterange for day', () => {
        const day = '2021-03-10';
        const result = Periods.parse('day', day).getDateRange();
        expect(result).toEqual([new Date(day), new Date(day)]);
    });

    it('should handle date mocking', () => {
        originalDateNow = Date.now;
        Date.now = () => new Date('2021-03-31').getTime();

        const result = Periods.parseDate('last month');
        expect(result.getMonth()).toEqual(1); // February
    });
});
```

### HTTP mocking with nock

```typescript
import nock from 'nock';

describe('API Integration', () => {
    let nockScope: nock.Scope;

    beforeAll(() => {
        nockScope = nock('http://localhost')
            .persist()
            .post('/')
            .query((query) => query.method === 'API.getSomeData')
            .reply(200, JSON.stringify({ data: 'test' }));
    });

    afterAll(() => {
        nockScope.done();
    });

    it('should fetch data', async () => {
        // Test implementation
    });
});
```

---

## Running Tests

### PHP tests

```bash
# Run all plugin tests
./console tests:run {PluginName}

# Run specific test type
./console tests:run {PluginName} --testsuite unit
./console tests:run {PluginName} --testsuite integration

# Run specific test file
./console tests:run plugins/{PluginName}/tests/Unit/ModelTest.php

# Run specific test method
./console tests:run plugins/{PluginName}/tests/Unit/ModelTest.php --filter test_validateName
```

### Vue/TypeScript tests

```bash
# Run all frontend tests
npm test

# Run with coverage
npm test -- --coverage

# Watch mode
npm test -- --watch

# Run specific test file
npm test -- --testPathPattern=MyStore.spec.ts
```

### Building Vue components before testing

```bash
# Build for development
./console vue:build {PluginName}

# Build with watch mode
./console vue:build {PluginName} --watch
```

---

## Test Naming Conventions

### PHP

| Pattern | Example |
|---------|---------|
| `test_methodName_expectedBehavior` | `test_addGoal_throwsOnEmptyName` |
| `test_methodName_whenCondition` | `test_getGoals_whenNoGoalsExist` |
| `test_methodName_returnsExpected` | `test_validateName_returnsTrimmed` |

### TypeScript/JavaScript

| Pattern | Example |
|---------|---------|
| Descriptive `it()` string | `'should load items on first call'` |
| Behavior-focused | `'should return cached items on subsequent calls'` |
| Error case | `'should throw when id is invalid'` |

---

## Best Practices

1. **Test one thing per test** — each test should verify a single behavior
2. **Use descriptive names** — test names should document expected behavior
3. **Reset state** — use `setUp()`/`tearDown()` to clean up between tests
4. **Mock external dependencies** — use `jest.spyOn()` for API calls, avoid real HTTP
5. **Use data providers** — parametrize tests for multiple input/output combinations
6. **Test edge cases** — empty strings, null values, boundary conditions
7. **Don't test framework behavior** — focus on your plugin's logic
8. **Keep tests fast** — unit tests should be instant, integration tests should be quick
9. **Test error paths** — verify exceptions and error handling, not just happy paths
10. **Use fixtures for complex data** — create reusable fixture classes for system tests
