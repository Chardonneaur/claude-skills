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
