## Project Overview

This is **dbt-date**, a dbt extension package for handling common date logic and calendar functionality. It provides a comprehensive library of date manipulation macros for dbt projects.

## Repository Structure

```
macros/
  calendar_date/      # Date manipulation macros (day_name, week_start, etc.)
  fiscal_date/        # Fiscal period/calendar macros
  _utils/             # Internal utilities (date_spine, generate_series)
  get_base_dates.sql  # Generate date spine with flexible inputs
  get_date_dimension.sql  # Build comprehensive date dimension tables

integration_tests/   # Integration test suite
  models/            # Test models (dim_date, dim_date_fiscal, etc.)
  ci/                # CI-specific dbt profiles
  docker/            # Docker configs for testing
```

## Core Architecture

### Macro Dispatch Pattern
This package uses dbt's adapter dispatch system to provide database-specific implementations. Each macro follows this pattern:

```sql
{% macro macro_name(args) %}
    {{ adapter.dispatch('macro_name', 'dbt_date')(args) }}
{% endmacro %}

{% macro default__macro_name(args) %}
    -- Default implementation
{% endmacro %}

{% macro bigquery__macro_name(args) %}
    -- BigQuery-specific implementation
{% endmacro %}
```

When adding or modifying macros, check if database-specific implementations exist (postgres__, bigquery__, snowflake__, etc.) and ensure consistency across all variants.

### Date Dimension Building
The package provides two key macros for building date dimensions:

1. **get_base_dates**: Generates a date spine (list of dates)
   - Can specify start_date/end_date OR n_dateparts (relative periods)
   - Wraps dbt_utils.date_spine for compatibility

2. **get_date_dimension**: Generates full date dimension with all attributes
   - Includes prior year comparisons, week/month/quarter/year fields
   - ISO week support alongside standard US calendar weeks
   - Different implementations for PostgreSQL vs other databases

### Timezone Handling
The package requires a timezone variable in dbt_project.yml:
```yaml
vars:
    "dbt_date:time_zone": "America/Los_Angeles"
```

Many macros accept an optional `tz` parameter to override this default. When working with timezone-aware macros, respect this pattern.

## Development Commands

### Running Tests

The integration tests verify macro functionality across all supported databases.

**Run all tests for a specific target:**
```bash
cd integration_tests
dbt build -t postgres
dbt build -t bigquery
dbt build -t snowflake
dbt build -t duckdb
```

**Run tests for multiple targets:**
```bash
cd integration_tests
./test.sh postgres bigquery snowflake
```

**Install dependencies first:**
```bash
cd integration_tests
dbt deps
```

### Testing with Docker

Local testing uses Docker Compose for database containers:
```bash
cd integration_tests
./docker-start.sh    # Start test databases
./docker-stop.sh     # Stop test databases
```

### CI/CD

CircleCI runs integration tests on every PR:
- Tests all supported databases (Postgres, BigQuery, Snowflake, DuckDB, Spark)
- Requires manual approval (workflow hold step)
- Profile configuration in `integration_tests/ci/`

## Key Concepts

### Week Handling
The package distinguishes between:
- **ISO weeks**: Start Monday, end Sunday (iso_week_start, iso_week_end, iso_week_of_year)
- **US weeks**: Start Sunday, end Saturday (week_start, week_end, week_of_year)

Always use the appropriate variant for your use case.

### Relative Date Functions
Functions like `n_days_ago`, `n_months_ago`, etc. calculate dates relative to:
- `today()` by default
- A specified date column when provided
- The configured timezone (or override via `tz` parameter)

### Fiscal Periods
The `get_fiscal_periods` macro implements 4-5-4 retail calendar logic:
- Requires a base date dimension (from `get_date_dimension`)
- Takes year_end_month and week_start_day parameters
- Useful for retail/financial reporting

## When Modifying Macros

1. **Check all database implementations**: A macro may have default__, postgres__, bigquery__, snowflake__, etc. variants
2. **Update tests**: Add/modify test cases in `integration_tests/models/test_dates.yml`
3. **Test across databases**: Run `dbt build` for affected database adapters
4. **Update README.md**: Document new macros or parameter changes
5. **Consider timezone behavior**: Ensure `tz` parameter is consistently handled

## Common Patterns

### Adding a new date macro:
1. Create file in `macros/calendar_date/` or `macros/fiscal_date/`
2. Use dispatch pattern for cross-database compatibility
3. Add test in `integration_tests/models/test_dates.sql` and `.yml`
4. Document in README.md with usage examples

### Cross-database compatibility:
- Use `dbt.type_timestamp()`, `dbt.type_int()` for type casting
- Use `dbt.date_trunc()`, `dbt.dateadd()` from dbt-core
- Implement database-specific variants when SQL syntax differs
- Test on at least Postgres, BigQuery, and Snowflake

## Dependencies

This package depends on:
- dbt-core >=1.2.0, <2.0.0
- dbt-utils (implicitly, for date_spine functionality)

The integration tests use additional packages configured in `integration_tests/packages.yml`.
