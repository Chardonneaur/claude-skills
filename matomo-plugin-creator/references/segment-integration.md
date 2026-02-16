# Segment Integration Reference

How to define segments, implement custom SQL filters, and hook into Matomo's segment system.
Based on real patterns from Referrers, Actions, UserCountry, and core segment classes.

## Table of Contents

1. [Concepts](#concepts)
2. [Defining Segments via Dimensions](#defining-segments-via-dimensions)
3. [Manual Segment Creation](#manual-segment-creation)
4. [sqlFilterValue Callback](#sqlfiltervalue-callback)
5. [sqlFilter Callback](#sqlfilter-callback)
6. [Match Type Constants](#match-type-constants)
7. [Segment Events](#segment-events)
8. [Union of Segments](#union-of-segments)
9. [Segment Properties Reference](#segment-properties-reference)

---

## Concepts

Segments allow users to filter analytics data by conditions (e.g., `pageUrl==example.com`).
Each segment maps a user-facing name to a database column with optional value transformation.

**Processing order:**
1. `sqlFilterValue` runs first — transforms the user's input value (e.g., `"search"` → `2`)
2. `sqlFilter` runs second — generates SQL or returns a processed value for the WHERE clause

**Segment types:**
- `TYPE_DIMENSION` — text/categorical values (pageUrl, country, etc.)
- `TYPE_METRIC` — numeric values (visits, actions, etc.)

---

## Defining Segments via Dimensions

The simplest way to add a segment is through a Dimension class. Matomo auto-registers segments
from dimension properties.

### Using dimension properties (auto-registration)

```php
<?php

namespace Piwik\Plugins\{PluginName}\Columns;

use Piwik\Plugin\Dimension\VisitDimension;

class ExampleDimension extends VisitDimension
{
    protected $columnName = 'example_dimension';
    protected $columnType = 'VARCHAR(255) NULL DEFAULT NULL';
    protected $nameSingular = '{PluginName}_DimensionName';

    // These properties auto-create a segment:
    protected $segmentName = 'exampleDimension';   // URL parameter name
    protected $sqlSegment = 'log_visit.example_dimension';  // DB column
    protected $acceptValues = 'value1, value2, value3';     // Help text
    protected $category = 'General_Visitors';

    // Optional: value transformation (callable string or array)
    protected $sqlFilterValue = 'Piwik\Plugins\{PluginName}\myValueMapper';

    // Optional: SQL generation callback
    protected $sqlFilter = ['Piwik\Plugins\{PluginName}\MyHelper', 'getSqlFilter'];
}
```

### Using configureSegments() (explicit control)

```php
<?php

namespace Piwik\Plugins\{PluginName}\Columns;

use Piwik\Columns\DimensionSegmentFactory;
use Piwik\Plugin\Dimension\VisitDimension;
use Piwik\Plugin\Segment;
use Piwik\Segment\SegmentsList;

class ExampleDimension extends VisitDimension
{
    protected $columnName = 'example_dimension';
    protected $columnType = 'TINYINT(1) UNSIGNED NULL';
    protected $nameSingular = '{PluginName}_DimensionName';

    public function configureSegments(
        SegmentsList $segmentsList,
        DimensionSegmentFactory $dimensionSegmentFactory
    ): void {
        // Create a segment from the dimension with auto-populated defaults
        $segment = $dimensionSegmentFactory->createSegment();
        $segment->setName('{PluginName}_SegmentName');

        // Override defaults as needed
        $segment->setSqlFilterValue(function ($value) {
            $map = ['active' => 1, 'inactive' => 0];
            return $map[$value] ?? $value;
        });

        $segmentsList->addSegment($segment);
    }
}
```

### Enum dimensions (auto-generated sqlFilterValue)

When a dimension has `getEnumColumnValues()`, the DimensionSegmentFactory automatically
creates a `sqlFilterValue` that maps enum labels to their numeric IDs:

```php
class ReferrerType extends VisitDimension
{
    protected $type = self::TYPE_ENUM;
    protected $segmentName = 'referrerType';
    protected $sqlFilterValue = 'Piwik\Plugins\Referrers\getReferrerTypeFromShortName';

    public function getEnumColumnValues(): array
    {
        return [
            Common::REFERRER_TYPE_DIRECT_ENTRY   => 'direct',
            Common::REFERRER_TYPE_WEBSITE        => 'website',
            Common::REFERRER_TYPE_SEARCH_ENGINE  => 'search',
            Common::REFERRER_TYPE_CAMPAIGN       => 'campaign',
        ];
    }
}
```

---

## Manual Segment Creation

Add segments without a dimension via the `Segment.addSegments` event:

```php
// In your main plugin class
public function registerEvents(): array
{
    return [
        'Segment.addSegments' => 'addSegments',
    ];
}

public function addSegments(SegmentsList $segmentsList): void
{
    $segment = new \Piwik\Plugin\Segment();
    $segment->setSegment('myCustomSegment');
    $segment->setType(\Piwik\Plugin\Segment::TYPE_DIMENSION);
    $segment->setName('{PluginName}_MySegment');
    $segment->setCategory('General_Visitors');
    $segment->setSqlSegment('log_visit.my_column');
    $segment->setAcceptedValues('value1, value2');

    $segmentsList->addSegment($segment);
}
```

---

## sqlFilterValue Callback

Runs **first**. Transforms the user's input value before SQL comparison.
Use this to map human-readable names to database values.

### Signature

```php
function(string $value, string $sqlSegment, string $matchType, string $segmentName): mixed
```

Only the first parameter is required; Matomo passes all four.

### Example: Mapping names to IDs

```php
// From Referrers plugin — maps 'search' → 2, 'direct' → 1, etc.
function getReferrerTypeFromShortName($name)
{
    $map = [
        Common::REFERRER_TYPE_SEARCH_ENGINE  => 'search',
        Common::REFERRER_TYPE_WEBSITE        => 'website',
        Common::REFERRER_TYPE_DIRECT_ENTRY   => 'direct',
        Common::REFERRER_TYPE_CAMPAIGN       => 'campaign',
    ];

    if (isset($map[$name])) {
        return $map[$name];
    }
    if ($found = array_search($name, $map)) {
        return $found;
    }
    throw new \Exception("Referrer type '$name' is not valid.");
}
```

### Setting it on a segment

```php
// As a property (callable string)
protected $sqlFilterValue = 'Piwik\Plugins\Referrers\getReferrerTypeFromShortName';

// As a closure
$segment->setSqlFilterValue(function ($value) {
    return strtolower(trim($value));
});

// As a static method reference
$segment->setSqlFilterValue([MyHelper::class, 'mapValue']);
```

---

## sqlFilter Callback

Runs **second** (after sqlFilterValue). Generates custom SQL for the WHERE clause.
Can return a simple value or a complex subquery.

### Signature

```php
function(string $valueToMatch, string $sqlField, string $matchType, string $segmentName): mixed
```

### Return types

**Option A — Return a scalar value** (used directly in the WHERE clause):

```php
$segment->setSqlFilter(function ($value, $sqlField, $matchType, $segmentName) {
    // Look up the action ID from the database
    $idAction = Db::fetchOne(
        "SELECT idaction FROM " . Common::prefixTable('log_action') . " WHERE name = ?",
        [$value]
    );
    return $idAction ?: 0;
});
```

**Option B — Return an array with SQL subquery** (for complex matching):

```php
$segment->setSqlFilter(function ($value, $sqlField, $matchType, $segmentName) {
    $sql = "SELECT idaction FROM " . Common::prefixTable('log_action')
         . " WHERE name LIKE CONCAT('%', ?, '%') AND type = 1";

    return [
        'SQL'  => $sql,
        'bind' => $value,
    ];
});
```

When returning `['SQL' => ..., 'bind' => ...]`, Matomo uses `MATCH_ACTIONS_CONTAINS` (IN subquery).

### Real example from Actions plugin

```php
// Used for pageUrl, pageTitle, etc. — optimized action ID lookup
protected $sqlFilter = [TableLogAction::class, 'getOptimizedIdActionSqlMatch'];

// The method handles different match types:
// - MATCH_EQUAL/NOT_EQUAL: returns numeric idaction directly
// - MATCH_CONTAINS/STARTS_WITH/etc.: returns ['SQL' => subquery, 'bind' => value]
```

---

## Match Type Constants

From `Piwik\Segment\SegmentExpression`:

### Comparison operators

| Constant | Symbol | Description |
|----------|--------|-------------|
| `MATCH_EQUAL` | `==` | Exact match |
| `MATCH_NOT_EQUAL` | `!=` | Not equal |
| `MATCH_GREATER_OR_EQUAL` | `>=` | Greater or equal |
| `MATCH_LESS_OR_EQUAL` | `<=` | Less or equal |
| `MATCH_GREATER` | `>` | Greater than |
| `MATCH_LESS` | `<` | Less than |

### String operators

| Constant | Symbol | Description |
|----------|--------|-------------|
| `MATCH_CONTAINS` | `=@` | Contains substring |
| `MATCH_DOES_NOT_CONTAIN` | `!@` | Does not contain |
| `MATCH_STARTS_WITH` | `=^` | Starts with |
| `MATCH_ENDS_WITH` | `=$` | Ends with |

### Regex operators

| Constant | Symbol | Description |
|----------|--------|-------------|
| `MATCH_REGEXP` | `=~` | Matches regex |
| `MATCH_NOT_REGEXP` | `!~` | Does not match regex |

### NULL/empty checks

| Constant | Symbol | Description |
|----------|--------|-------------|
| `MATCH_IS_NOT_NULL_NOR_EMPTY` | `::NOT_NULL` | Has a value |
| `MATCH_IS_NULL_OR_EMPTY` | `::NULL` | Is null or empty |

### Internal operators (returned by sqlFilter)

| Constant | Symbol | Description |
|----------|--------|-------------|
| `MATCH_ACTIONS_CONTAINS` | `IN` | ID in subquery result |
| `MATCH_ACTIONS_NOT_CONTAINS` | `NOTIN` | ID not in subquery result |

### Delimiters

| Constant | Symbol | Description |
|----------|--------|-------------|
| `AND_DELIMITER` | `;` | AND between conditions |
| `OR_DELIMITER` | `,` | OR between conditions |

---

## Segment Events

### Segment.addSegments

Add new segments to the global list. Called during segment discovery.

```php
public function addSegments(SegmentsList $segmentsList): void
{
    $segment = new \Piwik\Plugin\Segment();
    $segment->setSegment('mySegment');
    $segment->setSqlSegment('log_visit.my_column');
    $segment->setType(Segment::TYPE_DIMENSION);
    $segment->setName('MyPlugin_SegmentName');
    $segment->setCategory('General_Visitors');
    $segmentsList->addSegment($segment);
}
```

### Segment.filterSegments

Modify, wrap, or remove existing segments. Receives the full `SegmentsList`.

```php
public function filterSegments(SegmentsList &$list): void
{
    $segments = $list->getSegments();

    foreach ($segments as $segment) {
        // Wrap an existing sqlFilter with custom behavior
        $originalFilter = $segment->getSqlFilter();
        $segment->setSqlFilter(function ($value, $sqlField, $matchType, $segmentName) use ($originalFilter) {
            // Custom pre-processing
            $value = strtolower($value);

            // Call original filter if it exists
            if ($originalFilter) {
                return call_user_func($originalFilter, $value, $sqlField, $matchType, $segmentName);
            }
            return $value;
        });
    }
}
```

---

## Union of Segments

Combine multiple segments into a single logical segment using `setUnionOfSegments()`:

```php
$segment = new \Piwik\Plugin\Segment();
$segment->setSegment('visitEcommerceStatus');
$segment->setType(Segment::TYPE_DIMENSION);
$segment->setName('General_EcommerceStatus');
$segment->setCategory('Goals_Ecommerce');

// This segment matches if ANY of these sub-segments match
$segment->setUnionOfSegments([
    'visitEcommerceStatusConverted',
    'visitEcommerceStatusNotConverted',
    'visitEcommerceStatusAbandonedCart',
]);

$segmentsList->addSegment($segment);
```

---

## Segment Properties Reference

### Segment class setter methods

| Method | Purpose |
|--------|---------|
| `setSegment($name)` | URL parameter name (e.g., `pageUrl`) |
| `setSqlSegment($column)` | Database column (e.g., `log_visit.country`) |
| `setType($type)` | `TYPE_DIMENSION` or `TYPE_METRIC` |
| `setName($translationKey)` | Display name translation key |
| `setCategory($translationKey)` | Category for grouping in UI |
| `setAcceptedValues($text)` | Help text for accepted values |
| `setSqlFilterValue($callback)` | Value transformation callback |
| `setSqlFilter($callback)` | SQL generation callback |
| `setUnionOfSegments($segments)` | Combine multiple segment names |
| `setPermission($bool)` | Restrict access (false = hidden) |
| `setSuggestedValuesCallback($cb)` | Callback for autocomplete suggestions |
| `setSuggestedValuesApi($api)` | API method for autocomplete |
| `setIsInternal($bool)` | Hide from UI (still usable in API) |
| `setRequiresRegisteredUser($bool)` | Only show to logged-in users |

### Action type constants (for sqlFilter)

Used in `TableLogAction::guessActionTypeFromSegment()`:

| Constant | Value | Segment names |
|----------|-------|---------------|
| `Action::TYPE_PAGE_URL` | 1 | `pageUrl` |
| `Action::TYPE_OUTLINK` | 2 | `outlinkUrl` |
| `Action::TYPE_DOWNLOAD` | 3 | `downloadUrl` |
| `Action::TYPE_PAGE_TITLE` | 4 | `pageTitle` |
| `Action::TYPE_SITE_SEARCH` | 8 | `siteSearchKeyword` |
| `Action::TYPE_EVENT_CATEGORY` | 9 | `eventCategory` |
| `Action::TYPE_EVENT_ACTION` | 11 | `eventAction` |
| `Action::TYPE_EVENT_NAME` | 12 | `eventName` |
