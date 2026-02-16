# Archiver Patterns Reference

How to aggregate and persist analytics data in Matomo plugins.
**Prefer the RecordBuilder pattern** (Matomo 5.x) for new plugins.

## Table of Contents

1. [Concepts](#concepts)
2. [RecordBuilder Pattern (Preferred)](#recordbuilder-pattern-preferred)
3. [Classic Archiver Pattern](#classic-archiver-pattern)
4. [LogAggregator Methods](#logaggregator-methods)
5. [ArchiveProcessor Methods](#archiveprocessor-methods)
6. [Performance Best Practices](#performance-best-practices)

---

## Concepts

Matomo stores analytics data in two forms:

- **Numeric records** — single values (e.g. total visits, conversion count)
- **Blob records** — serialized DataTables (e.g. pages report, referrers report)

Records are named: `{PluginName}_{recordName}` (e.g. `MyPlugin_visitorsByCountry`).

The archiving process runs in two modes:

- **Day archiving** (`aggregateDayReport`/`buildFromLogs`): Query raw log tables
  (`log_visit`, `log_link_visit_action`, `log_conversion`) and aggregate into records
- **Period archiving** (`aggregateMultipleReports`/`buildForNonDayPeriod`): Combine
  day-level records into week/month/year/range records

---

## RecordBuilder Pattern (Preferred)

Introduced in Matomo 5.x (PR #20394). Splits large Archiver classes into smaller,
focused units that each handle a specific set of related records.

**Benefits:**
- Better query origin hints in SQL (helps debugging)
- Granular archiving — only needed records are computed
- Cleaner separation of concerns
- Better performance for partial archives

### Creating a RecordBuilder

RecordBuilders live in a `RecordBuilders/` directory within your plugin:

```php
<?php

namespace Piwik\Plugins\{PluginName}\RecordBuilders;

use Piwik\ArchiveProcessor;
use Piwik\ArchiveProcessor\Record;
use Piwik\ArchiveProcessor\RecordBuilder;
use Piwik\Config;
use Piwik\DataTable;
use Piwik\Metrics;

class ExampleRecordBuilder extends RecordBuilder
{
    public function __construct()
    {
        parent::__construct();

        $this->maxRowsInTable = Config::getInstance()->General['datatable_archiving_maximum_rows_standard'];
        $this->maxRowsInSubTable = Config::getInstance()->General['datatable_archiving_maximum_rows_subtable'];
        $this->columnToSortByBeforeTruncation = Metrics::INDEX_NB_VISITS;
    }

    /**
     * Declare which records this builder produces.
     * This lets Matomo know when to invoke this builder.
     */
    public function getRecordMetadata(ArchiveProcessor $archiveProcessor): array
    {
        return [
            Record::make(Record::TYPE_BLOB, '{PluginName}_exampleReport'),
            // Add more records if this builder produces multiple related ones:
            // Record::make(Record::TYPE_NUMERIC, '{PluginName}_totalExampleCount'),
        ];
    }

    /**
     * Aggregate raw log data for a single day.
     * Called during day archiving.
     */
    public function buildFromLogs(ArchiveProcessor $archiveProcessor): void
    {
        $logAggregator = $archiveProcessor->getLogAggregator();

        // Use LogAggregator methods to query log tables
        $query = $logAggregator->queryVisitsByDimension(
            dimensions: ['log_visit.example_dimension'],
        );

        $dataTable = new DataTable();

        while ($row = $query->fetch()) {
            $dataTable->addRowFromSimpleArray([
                'label'                      => $row['label'] ?? '',
                Metrics::INDEX_NB_VISITS     => (int) $row[Metrics::INDEX_NB_VISITS],
                Metrics::INDEX_NB_UNIQ_VISITORS => (int) $row[Metrics::INDEX_NB_UNIQ_VISITORS],
            ]);
        }

        $archiveProcessor->insertBlobRecord(
            '{PluginName}_exampleReport',
            $dataTable->getSerialized(
                $this->maxRowsInTable,
                $this->maxRowsInSubTable,
                $this->columnToSortByBeforeTruncation
            )
        );
    }

    /**
     * Aggregate sub-period records for non-day periods (week, month, year, range).
     * Typically just sums up day records.
     */
    public function buildForNonDayPeriod(ArchiveProcessor $archiveProcessor): void
    {
        $archiveProcessor->aggregateDataTableRecords(
            ['{PluginName}_exampleReport'],
            $this->maxRowsInTable,
            $this->maxRowsInSubTable,
            $this->columnToSortByBeforeTruncation
        );
    }
}
```

### Auto-discovery

RecordBuilders are auto-discovered if they:
1. Are in the `RecordBuilders/` directory of your plugin
2. Extend `ArchiveProcessor\RecordBuilder`
3. Have a no-argument constructor (or optional arguments)

If your RecordBuilder needs parameters (e.g. an entity ID), register it via event:

```php
// In your main plugin class
public function registerEvents(): array
{
    return [
        'Archiver.addRecordBuilders' => 'addRecordBuilders',
    ];
}

public function addRecordBuilders(array &$recordBuilders): void
{
    $recordBuilders[] = new RecordBuilders\ParameterizedBuilder($someId);
}
```

### Empty Archiver with RecordBuilders only

If all your archiving logic is in RecordBuilders, you still need an Archiver class,
but it can be minimal:

```php
<?php

namespace Piwik\Plugins\{PluginName};

class Archiver extends \Piwik\Plugin\Archiver
{
    public function aggregateDayReport(): void
    {
        // All logic handled by RecordBuilders
    }

    public function aggregateMultipleReports(): void
    {
        // All logic handled by RecordBuilders
    }
}
```

---

## Classic Archiver Pattern

For simpler plugins or when backward compatibility with Matomo 4.x is needed.

```php
<?php

namespace Piwik\Plugins\{PluginName};

use Piwik\Config;
use Piwik\DataTable;
use Piwik\Metrics;
use Piwik\Plugin\Archiver as PluginArchiver;

class Archiver extends PluginArchiver
{
    public const RECORD_NAME = '{PluginName}_report';

    private int $maxRows;

    public function __construct(\Piwik\ArchiveProcessor $processor)
    {
        parent::__construct($processor);
        $this->maxRows = Config::getInstance()->General['datatable_archiving_maximum_rows_standard'];
    }

    public function aggregateDayReport(): void
    {
        $logAggregator = $this->getLogAggregator();

        $query = $logAggregator->queryVisitsByDimension(
            ['log_visit.example_dimension']
        );

        $table = new DataTable();
        while ($row = $query->fetch()) {
            $table->addRowFromSimpleArray([
                'label'                      => $row['label'] ?? '',
                Metrics::INDEX_NB_VISITS     => (int) $row[Metrics::INDEX_NB_VISITS],
                Metrics::INDEX_NB_UNIQ_VISITORS => (int) $row[Metrics::INDEX_NB_UNIQ_VISITORS],
            ]);
        }

        $this->getProcessor()->insertBlobRecord(
            self::RECORD_NAME,
            $table->getSerialized($this->maxRows, $this->maxRows, Metrics::INDEX_NB_VISITS)
        );
    }

    public function aggregateMultipleReports(): void
    {
        $this->getProcessor()->aggregateDataTableRecords(
            [self::RECORD_NAME],
            $this->maxRows
        );
    }
}
```

### Numeric records (simple metrics)

```php
public function aggregateDayReport(): void
{
    $processor = $this->getProcessor();
    $metricValue = $this->computeMetric();
    $processor->insertNumericRecord('{PluginName}_myMetric', $metricValue);
}

public function aggregateMultipleReports(): void
{
    $this->getProcessor()->aggregateNumericMetrics(['{PluginName}_myMetric']);
}
```

---

## LogAggregator Methods

The LogAggregator provides high-level methods to query log tables. These handle
segments, date ranges, and parameter binding automatically.

| Method | Queries | Use for |
|--------|---------|---------|
| `queryVisitsByDimension($dimensions)` | log_visit | Visit-level aggregation |
| `queryActionsByDimension($dimensions)` | log_link_visit_action | Action/pageview-level data |
| `queryConversionsByDimension($dimensions)` | log_conversion | Goal conversion data |
| `queryEcommerceItems($dimensions)` | log_conversion_item | Ecommerce product data |

Each returns a PDOStatement cursor. Iterate with `while ($row = $query->fetch())`.

### Dimension format

Dimensions are column references in the log tables:

```php
// Single dimension
$query = $logAggregator->queryVisitsByDimension(['log_visit.country']);

// Multiple dimensions (creates subtables)
$query = $logAggregator->queryVisitsByDimension([
    'log_visit.country',
    'log_visit.city',
]);
```

### Additional selects and WHERE

```php
$query = $logAggregator->queryVisitsByDimension(
    dimensions: ['log_visit.example_dimension'],
    where: 'log_visit.example_dimension IS NOT NULL',
    additionalSelects: ['SUM(log_visit.visit_total_time) AS total_time'],
);
```

---

## ArchiveProcessor Methods

### Inserting records

```php
$processor = $this->getProcessor(); // or $archiveProcessor in RecordBuilder

// Blob record (serialized DataTable)
$processor->insertBlobRecord('RecordName', $dataTable->getSerialized($maxRows));

// Multiple blob records
$processor->insertBlobRecords('RecordName', $serializedData);

// Numeric record
$processor->insertNumericRecord('RecordName', $numericValue);

// Multiple numeric records
$processor->insertNumericRecords(['Record1' => $val1, 'Record2' => $val2]);
```

### Aggregating sub-period records

```php
// Aggregate blob DataTable records (sums rows by label)
$processor->aggregateDataTableRecords(
    recordNames: ['MyPlugin_report'],
    maximumRowsInDataTableLevelZero: $maxRows,
    maximumRowsInSubDataTable: $maxRowsSub,
    columnToSortByBeforeTruncation: Metrics::INDEX_NB_VISITS
);

// Aggregate numeric records (simple sum)
$processor->aggregateNumericMetrics(['MyPlugin_metricA', 'MyPlugin_metricB']);
```

---

## Performance Best Practices

1. **Respect row limits** — Always read from config:
   ```php
   $maxRows = Config::getInstance()->General['datatable_archiving_maximum_rows_standard'];
   ```

2. **Use LogAggregator** — It handles segments, date filtering, and prepared statements.
   Raw SQL should be a last resort.

3. **Minimize queries** — Each RecordBuilder/Archiver should use as few log table
   queries as possible. Group related records that need the same query.

4. **Sort before truncation** — Always specify `$columnToSortByBeforeTruncation`
   (typically `Metrics::INDEX_NB_VISITS`) so truncation drops the least important rows.

5. **Avoid N+1 queries** — Don't query inside loops. Fetch all needed data in one
   query, then process in memory.

6. **Use DataArray for complex aggregations** — When building tables from multiple
   queries, use `Piwik\DataArray` to merge results efficiently:
   ```php
   $dataArray = new \Piwik\DataArray();
   $dataArray->sumMetricsVisits($label, $row);
   $dataArray->sumMetricsGoals($label, $row);
   $table = $dataArray->asDataTable();
   ```

7. **Partial archives** — For plugins with many reports, support partial archiving
   by checking `$this->isRequestedReport('RecordName')` in classic Archivers.
   RecordBuilders handle this automatically via `getRecordMetadata()`.

---

## Parameterized RecordBuilders

When a RecordBuilder needs parameters (e.g. one builder per entity like goal, form, or funnel),
it cannot be auto-discovered. Register it via the `Archiver.addRecordBuilders` event.

### Constructor with parameters

```php
<?php

namespace Piwik\Plugins\{PluginName}\RecordBuilders;

use Piwik\ArchiveProcessor;
use Piwik\ArchiveProcessor\Record;
use Piwik\ArchiveProcessor\RecordBuilder;

class EntityRecordBuilder extends RecordBuilder
{
    private int $entityId;
    private string $entityName;

    public function __construct(int $entityId, string $entityName)
    {
        parent::__construct();
        $this->entityId = $entityId;
        $this->entityName = $entityName;

        $this->maxRowsInTable = 500;
        $this->columnToSortByBeforeTruncation = Metrics::INDEX_NB_VISITS;
    }

    public function getRecordMetadata(ArchiveProcessor $archiveProcessor): array
    {
        return [
            Record::make(Record::TYPE_BLOB, "{PluginName}_entity_{$this->entityId}"),
            Record::make(Record::TYPE_NUMERIC, "{PluginName}_entity_{$this->entityId}_total"),
        ];
    }

    public function buildFromLogs(ArchiveProcessor $archiveProcessor): void
    {
        $logAggregator = $archiveProcessor->getLogAggregator();
        // Use $this->entityId to filter queries
        $query = $logAggregator->queryVisitsByDimension(
            dimensions: ['log_visit.some_dimension'],
            where: "log_visit.entity_id = {$this->entityId}",
        );
        // ... process results
    }

    public function buildForNonDayPeriod(ArchiveProcessor $archiveProcessor): void
    {
        $archiveProcessor->aggregateDataTableRecords(
            ["{PluginName}_entity_{$this->entityId}"],
            $this->maxRowsInTable
        );
        $archiveProcessor->aggregateNumericMetrics(
            ["{PluginName}_entity_{$this->entityId}_total"]
        );
    }
}
```

### Event registration with lazy loading

```php
// In your main plugin class
public function registerEvents(): array
{
    return [
        'Archiver.addRecordBuilders' => 'addRecordBuilders',
    ];
}

public function addRecordBuilders(array &$recordBuilders): void
{
    // Load entities from DB and create a builder for each
    $model = new Model();
    $entities = $model->getAllActiveEntities();

    foreach ($entities as $entity) {
        $recordBuilders[] = new RecordBuilders\EntityRecordBuilder(
            (int) $entity['id'],
            $entity['name']
        );
    }
}
```

---

## Custom Metrics

### Computing derived metrics (ratios, percentages)

When you need metrics like conversion rates, bounce rates, or averages that are
derived from other metrics, compute them during day archiving and store as numeric records:

```php
public function buildFromLogs(ArchiveProcessor $archiveProcessor): void
{
    $logAggregator = $archiveProcessor->getLogAggregator();

    // Get raw counts
    $query = $logAggregator->queryVisitsByDimension(
        dimensions: ['log_visit.example_dimension'],
        additionalSelects: [
            'SUM(CASE WHEN log_visit.visit_total_actions > 1 THEN 1 ELSE 0 END) AS engaged_visits',
            'SUM(log_visit.visit_total_time) AS total_time',
        ],
    );

    $table = new DataTable();
    $totalVisits = 0;
    $totalTime = 0;

    while ($row = $query->fetch()) {
        $visits = (int) $row[Metrics::INDEX_NB_VISITS];
        $totalVisits += $visits;
        $totalTime += (int) $row['total_time'];

        $table->addRowFromSimpleArray([
            'label'                      => $row['label'] ?? '',
            Metrics::INDEX_NB_VISITS     => $visits,
            Metrics::INDEX_NB_UNIQ_VISITORS => (int) $row[Metrics::INDEX_NB_UNIQ_VISITORS],
        ]);
    }

    // Store blob record
    $archiveProcessor->insertBlobRecord(
        '{PluginName}_report',
        $table->getSerialized($this->maxRowsInTable)
    );

    // Store derived numeric metrics
    $archiveProcessor->insertNumericRecord('{PluginName}_totalVisits', $totalVisits);
    $archiveProcessor->insertNumericRecord('{PluginName}_totalTime', $totalTime);
    $archiveProcessor->insertNumericRecord(
        '{PluginName}_avgTimePerVisit',
        $totalVisits > 0 ? round($totalTime / $totalVisits) : 0
    );
}
```

---

## Segment-Aware Archiving

LogAggregator handles segments automatically. When Matomo archives data with a segment
applied, LogAggregator adds the segment's WHERE conditions to all queries.

### How it works

You don't need to do anything special — just use LogAggregator methods:

```php
// This query automatically respects segments
$query = $logAggregator->queryVisitsByDimension(['log_visit.country']);

// The LogAggregator internally:
// 1. Joins necessary log tables based on the segment definition
// 2. Adds WHERE conditions for the segment
// 3. Applies date range filtering
// 4. Handles parameter binding safely
```

### When raw SQL is needed

If you must use raw SQL (e.g. for complex queries not supported by LogAggregator),
get the segment's SQL from the LogAggregator:

```php
public function buildFromLogs(ArchiveProcessor $archiveProcessor): void
{
    $logAggregator = $archiveProcessor->getLogAggregator();

    // Get segment + date WHERE clause and bindings
    $where = $logAggregator->getWhereStatement('log_visit', 'visit_last_action_time');
    $from = $logAggregator->getFrom(); // table joins for segment

    $sql = "SELECT log_visit.my_column, COUNT(*) as cnt
            FROM " . Common::prefixTable('log_visit') . " AS log_visit
            $from
            WHERE $where
            GROUP BY log_visit.my_column";

    $bind = $logAggregator->getBindings();
    $result = Db::fetchAll($sql, $bind);
}
```

---

## Action Type Filtering

When building archivers that process action-level data, filter by action type
to get the correct data:

```php
use Piwik\Tracker\Action;

public function buildFromLogs(ArchiveProcessor $archiveProcessor): void
{
    $logAggregator = $archiveProcessor->getLogAggregator();

    // Page views only (type 1)
    $query = $logAggregator->queryActionsByDimension(
        dimensions: ['log_action.name'],
        where: 'log_action.type = ' . Action::TYPE_PAGE_URL,
    );

    // Outlinks only (type 2)
    $query = $logAggregator->queryActionsByDimension(
        dimensions: ['log_action.name'],
        where: 'log_action.type = ' . Action::TYPE_OUTLINK,
    );

    // Downloads only (type 3)
    $query = $logAggregator->queryActionsByDimension(
        dimensions: ['log_action.name'],
        where: 'log_action.type = ' . Action::TYPE_DOWNLOAD,
    );
}
```

### Action type constants

| Constant | Value | Description |
|----------|-------|-------------|
| `Action::TYPE_PAGE_URL` | 1 | Page view URLs |
| `Action::TYPE_OUTLINK` | 2 | Outbound link clicks |
| `Action::TYPE_DOWNLOAD` | 3 | File downloads |
| `Action::TYPE_PAGE_TITLE` | 4 | Page titles |
| `Action::TYPE_SITE_SEARCH` | 8 | Site search keywords |
| `Action::TYPE_EVENT_CATEGORY` | 9 | Event categories |
| `Action::TYPE_EVENT_ACTION` | 11 | Event actions |
| `Action::TYPE_EVENT_NAME` | 12 | Event names |
