# Building Slowly Changing Dimensions (SCD) - Hands-On Lab

A practical guide to implementing SCD Type 2 tables using two different approaches: full history processing and incremental updates.

---

## Table of Contents

- Lab Setup and Prerequisites
- Understanding the Source Data
- SCD Type 2 Table Design
- Approach 1: Full History Processing
- Approach 2: Incremental Updates
- Comparing Both Approaches
- Best Practices and Pitfalls

***

## Lab Setup and Prerequisites

**Tools required:**
- Docker (for PostgreSQL)
- SQL client (DataGrip, DBeaver, or DB Visualizer)
- GitHub repo with lab exercises

**Source data:** `player_seasons` table with NBA player statistics by season[1]

**Goal:** Convert season-by-season player records into a compressed SCD Type 2 table tracking changes in:
- `scoring_class` (star, good, average, bad)
- `is_active` (boolean)

***

## Understanding the Source Data

### Input: `players` Cumulative Table

Built in Day 1 lab with array structure:[1]

```sql
-- One row per player with all seasons in array
player_name | season_stats[] | scoring_class | is_active | current_season
AC Green    | [{1996,...}]   | bad          | true      | 1996
AC Green    | [{1996,...}]   | bad          | false     | 2001
```

**Problem:** This tracks changes daily but is inefficient for storage.[1]

**Solution:** Compress into SCD Type 2 with start/end dates.

***

## SCD Type 2 Table Design

### Create Custom Type

Define struct for easier data handling:[1]

```sql
CREATE TYPE scoring_class AS ENUM ('star', 'good', 'average', 'bad');
```

### Create SCD Table Schema

```sql
CREATE TABLE players_scd (
    player_name TEXT,
    scoring_class scoring_class,
    is_active BOOLEAN,
    start_season INTEGER,
    end_season INTEGER,
    current_season INTEGER,
    PRIMARY KEY (player_name, start_season)
);
```

**Key design decisions:**

**Columns tracked:** `scoring_class` and `is_active`[1]
**Temporal range:** `start_season` to `end_season`
**Partition metadata:** `current_season` (acts like date partition in production)[1]
**Primary key:** `(player_name, start_season)` ensures one record per player per change

***

## Approach 1: Full History Processing

### Overview

Process all historical data every run to generate complete SCD table. This is simpler but processes more data.[1]

### Step 1: Identify Previous Values with LAG

Use window functions to find when values changed:[1]

```sql
WITH with_previous AS (
    SELECT 
        player_name,
        scoring_class,
        is_active,
        current_season,
        -- Look back one season
        LAG(scoring_class, 1) OVER (
            PARTITION BY player_name 
            ORDER BY current_season
        ) as previous_scoring_class,
        LAG(is_active, 1) OVER (
            PARTITION BY player_name 
            ORDER BY current_season
        ) as previous_is_active
    FROM players
    WHERE current_season <= 2021
)
SELECT * FROM with_previous;
```

**Result:**
```
player_name | scoring_class | is_active | previous_scoring_class | previous_is_active | current_season
AC Green    | bad          | true      | NULL                   | NULL               | 1996
AC Green    | bad          | true      | bad                    | true               | 1997
AC Green    | bad          | false     | bad                    | true               | 2001  -- Changed!
```

***

### Step 2: Create Change Indicators

Flag rows where values changed:[1]

```sql
WITH with_previous AS (
    -- [Previous query]
),
with_indicators AS (
    SELECT 
        player_name,
        scoring_class,
        is_active,
        current_season,
        -- Combined change indicator
        CASE 
            WHEN scoring_class != previous_scoring_class 
                OR is_active != previous_is_active 
            THEN 1 
            ELSE 0 
        END as change_indicator
    FROM with_previous
)
SELECT * FROM with_indicators;
```

**Result:**
```
player_name | scoring_class | is_active | current_season | change_indicator
AC Green    | bad          | true      | 1996          | 1  -- First record (previous was NULL)
AC Green    | bad          | true      | 1997          | 0  -- No change
AC Green    | bad          | false     | 2001          | 1  -- Changed is_active!
```

***

### Step 3: Create Streak Identifier

Use cumulative sum to group consecutive unchanged values:[1]

```sql
WITH with_indicators AS (
    -- [Previous query]
),
with_streaks AS (
    SELECT 
        player_name,
        scoring_class,
        is_active,
        current_season,
        -- Running sum creates unique ID for each "streak"
        SUM(change_indicator) OVER (
            PARTITION BY player_name 
            ORDER BY current_season
        ) as streak_identifier
    FROM with_indicators
)
SELECT * FROM with_streaks
ORDER BY player_name, current_season;
```

**Result:**
```
player_name | scoring_class | is_active | current_season | streak_identifier
AC Green    | bad          | true      | 1996          | 0
AC Green    | bad          | true      | 1997          | 0  -- Same streak
AC Green    | bad          | true      | 1998          | 0  -- Same streak
AC Green    | bad          | true      | 1999          | 0  -- Same streak
AC Green    | bad          | true      | 2000          | 0  -- Same streak
AC Green    | bad          | false     | 2001          | 1  -- New streak!
AC Green    | bad          | false     | 2002          | 1  -- Same streak
```

**How it works:** The cumulative sum increments only when `change_indicator = 1`, creating unique IDs for each period of unchanged values.[1]

---

### Step 4: Collapse Streaks with Aggregation

Group by streak identifier and find min/max seasons:[1]

```sql
WITH with_streaks AS (
    -- [Previous query]
)
SELECT 
    player_name,
    scoring_class,
    is_active,
    MIN(current_season) as start_season,
    MAX(current_season) as end_season,
    2021 as current_season  -- Hard-coded for this run
FROM with_streaks
GROUP BY 
    player_name,
    streak_identifier,  -- Group by but don't include in output
    scoring_class,
    is_active
ORDER BY player_name, streak_identifier;
```

**Result (compressed SCD):**
```
player_name | scoring_class | is_active | start_season | end_season | current_season
AC Green    | bad          | true      | 1996         | 2000       | 2021
AC Green    | bad          | false     | 2001         | 2021       | 2021
```

**Compression achieved:** 26 yearly rows → 2 SCD rows (13x compression)![1]

***

### Step 5: Insert into SCD Table

```sql
INSERT INTO players_scd
WITH with_previous AS (
    -- [All CTEs from above]
),
with_indicators AS (...),
with_streaks AS (...)
SELECT 
    player_name,
    scoring_class,
    is_active,
    MIN(current_season) as start_season,
    MAX(current_season) as end_season,
    2021 as current_season
FROM with_streaks
GROUP BY player_name, streak_identifier, scoring_class, is_active
ORDER BY player_name, streak_identifier;
```

***

## Complete Full History Query

```sql
INSERT INTO players_scd
WITH with_previous AS (
    SELECT 
        player_name,
        scoring_class,
        is_active,
        current_season,
        LAG(scoring_class, 1) OVER (
            PARTITION BY player_name ORDER BY current_season
        ) as previous_scoring_class,
        LAG(is_active, 1) OVER (
            PARTITION BY player_name ORDER BY current_season
        ) as previous_is_active
    FROM players
    WHERE current_season <= 2021
),
with_indicators AS (
    SELECT 
        player_name,
        scoring_class,
        is_active,
        current_season,
        CASE 
            WHEN scoring_class != previous_scoring_class 
                OR is_active != previous_is_active 
            THEN 1 ELSE 0 
        END as change_indicator
    FROM with_previous
),
with_streaks AS (
    SELECT 
        player_name,
        scoring_class,
        is_active,
        current_season,
        SUM(change_indicator) OVER (
            PARTITION BY player_name ORDER BY current_season
        ) as streak_identifier
    FROM with_indicators
)
SELECT 
    player_name,
    scoring_class,
    is_active,
    MIN(current_season) as start_season,
    MAX(current_season) as end_season,
    2021 as current_season
FROM with_streaks
GROUP BY player_name, streak_identifier, scoring_class, is_active;
```

***

## Approach 2: Incremental Updates

### Overview

Process only new data (2022) and update existing SCD records. More complex but much more efficient.[1]

### The Challenge

When new data arrives, three scenarios exist:[1]

1. **Unchanged records:** Extend `end_season` by 1
2. **Changed records:** Close old record, create new one
3. **New records:** Insert completely new players

***

### Step 1: Define Data Sources

```sql
WITH last_season_scd AS (
    -- Current records that might change
    SELECT * FROM players_scd
    WHERE current_season = 2021
      AND end_season = 2021  -- Only "open" records
),
historical_scd AS (
    -- Completed records that never change
    SELECT * FROM players_scd
    WHERE current_season = 2021
      AND end_season < 2021  -- Already closed
),
this_season_data AS (
    -- New season data
    SELECT * FROM players
    WHERE current_season = 2022
)
```

**Why separate historical?** Records already closed (like Michael Jordan 1996-1998) will never change, so we can pass them through without processing.[1]

---

### Step 2: Unchanged Records

Extend `end_season` for players whose values didn't change:[1]

```sql
unchanged_records AS (
    SELECT 
        ts.player_name,
        ts.scoring_class,
        ts.is_active,
        ls.start_season,  -- Keep original start
        ts.current_season as end_season,  -- Extend to 2022
        ts.current_season
    FROM last_season_scd ls
    JOIN this_season_data ts
        ON ls.player_name = ts.player_name
    WHERE ts.scoring_class = ls.scoring_class
      AND ts.is_active = ls.is_active  -- Both match = unchanged
)
```

**Example:**
```
-- Last season (2021)
AC Green | bad | false | 2001 | 2021

-- Unchanged in 2022 → Extend
AC Green | bad | false | 2001 | 2022
```

***

### Step 3: Changed Records (Complex!)

When values change, we need **two records**:
1. Close the old record (set `end_season = 2021`)
2. Open a new record (set `start_season = 2022`, `end_season = 2022`)

**Create custom type for handling both records:**

```sql
CREATE TYPE scd_type AS (
    scoring_class scoring_class,
    is_active BOOLEAN,
    start_season INTEGER,
    end_season INTEGER
);
```

**Use array to generate both records, then unnest:**

```sql
changed_records AS (
    SELECT 
        ts.player_name,
        UNNEST(ARRAY[
            -- Old record (closed)
            ROW(
                ls.scoring_class,
                ls.is_active,
                ls.start_season,
                ls.end_season  -- Already 2021, stays same
            )::scd_type,
            -- New record (opened)
            ROW(
                ts.scoring_class,
                ts.is_active,
                ts.current_season,  -- Start at 2022
                ts.current_season   -- End at 2022
            )::scd_type
        ]) as records
    FROM this_season_data ts
    LEFT JOIN last_season_scd ls
        ON ts.player_name = ls.player_name
    WHERE (ts.scoring_class != ls.scoring_class
           OR ts.is_active != ls.is_active)
      AND ls.player_name IS NOT NULL  -- Not new players
),
unnested_changed_records AS (
    SELECT 
        player_name,
        (records).scoring_class,
        (records).is_active,
        (records).start_season,
        (records).end_season
    FROM changed_records
)
```

**Example:**
```
-- Player Aaron Brooks improved from 'average' to 'good'

-- Last season (2021)
Aaron Brooks | average | true | 2020 | 2021

-- Changed in 2022 → Two records:
Aaron Brooks | average | true | 2020 | 2021  -- Closed old
Aaron Brooks | good    | true | 2022 | 2022  -- Started new
```

***

### Step 4: New Records

Brand new players who weren't in last season's SCD:[1]

```sql
new_records AS (
    SELECT 
        ts.player_name,
        ts.scoring_class,
        ts.is_active,
        ts.current_season as start_season,
        ts.current_season as end_season
    FROM this_season_data ts
    LEFT JOIN last_season_scd ls
        ON ts.player_name = ls.player_name
    WHERE ls.player_name IS NULL  -- Not in last season
)
```

***

### Step 5: Combine All Records

```sql
SELECT player_name, scoring_class, is_active, start_season, end_season
FROM historical_scd  -- Pass through unchanged history

UNION ALL

SELECT player_name, scoring_class, is_active, start_season, end_season
FROM unchanged_records  -- Extended records

UNION ALL

SELECT player_name, scoring_class, is_active, start_season, end_season
FROM unnested_changed_records  -- Both old (closed) and new (opened)

UNION ALL

SELECT player_name, scoring_class, is_active, start_season, end_season
FROM new_records;  -- Brand new players
```

***

## Complete Incremental Query

```sql
INSERT INTO players_scd
WITH last_season_scd AS (
    SELECT * FROM players_scd
    WHERE current_season = 2021 AND end_season = 2021
),
historical_scd AS (
    SELECT player_name, scoring_class, is_active, start_season, end_season
    FROM players_scd
    WHERE current_season = 2021 AND end_season < 2021
),
this_season_data AS (
    SELECT * FROM players WHERE current_season = 2022
),
unchanged_records AS (
    SELECT 
        ts.player_name, ts.scoring_class, ts.is_active,
        ls.start_season, ts.current_season as end_season
    FROM last_season_scd ls
    JOIN this_season_data ts ON ls.player_name = ts.player_name
    WHERE ts.scoring_class = ls.scoring_class
      AND ts.is_active = ls.is_active
),
changed_records AS (
    SELECT 
        ts.player_name,
        UNNEST(ARRAY[
            ROW(ls.scoring_class, ls.is_active, ls.start_season, ls.end_season)::scd_type,
            ROW(ts.scoring_class, ts.is_active, ts.current_season, ts.current_season)::scd_type
        ]) as records
    FROM this_season_data ts
    LEFT JOIN last_season_scd ls ON ts.player_name = ls.player_name
    WHERE (ts.scoring_class != ls.scoring_class OR ts.is_active != ls.is_active)
      AND ls.player_name IS NOT NULL
),
unnested_changed_records AS (
    SELECT 
        player_name,
        (records).scoring_class,
        (records).is_active,
        (records).start_season,
        (records).end_season
    FROM changed_records
),
new_records AS (
    SELECT 
        ts.player_name, ts.scoring_class, ts.is_active,
        ts.current_season as start_season, ts.current_season as end_season
    FROM this_season_data ts
    LEFT JOIN last_season_scd ls ON ts.player_name = ls.player_name
    WHERE ls.player_name IS NULL
)
SELECT * FROM historical_scd
UNION ALL SELECT * FROM unchanged_records
UNION ALL SELECT * FROM unnested_changed_records
UNION ALL SELECT * FROM new_records;
```

***

## Comparing Both Approaches

### Full History Processing

**Pros:**
- Simpler logic—easier to understand and maintain[1]
- No dependency on yesterday's data
- Can parallelize backfills (process multiple date ranges simultaneously)
- Self-correcting—any mistakes get fixed on next run

**Cons:**
- Processes all history every run[1]
- Multiple expensive window functions across entire dataset
- Slower for large dimensional tables
- More compute resources required

**When to use:**
- Dimensional data is relatively small (< millions of entities)[1]
- Team prioritizes simplicity over efficiency
- Development speed more important than runtime performance

**Zach's experience at Airbnb:** Used this approach for unit economics SCD. Processed all bookings back to 2016 daily. "Don't get caught up making every pipeline a Ferrari"—shipping business value matters more than marginal optimization.[1]

---

### Incremental Updates

**Pros:**
- Processes 10-20x less data[1]
- Much faster execution
- More efficient use of compute resources
- Scalable to billions of entities

**Cons:**
- Complex logic with many edge cases[1]
- Harder to debug and maintain
- Sequential dependency (must run days in order)
- Cannot parallelize backfills
- Errors compound forward

**When to use:**
- Large dimensional tables (billions of entities like Facebook users)[1]
- Production systems at scale
- Team has strong SQL expertise
- Runtime performance is critical

---

### Performance Comparison

| Metric | Full History | Incremental |
|--------|--------------|-------------|
| Data scanned | All seasons (1996-2022) | Only 2021 + 2022 |
| Window functions | 2 full passes | 0 |
| Aggregations | 1 on full dataset | Multiple on small subsets |
| Backfill speed | Can parallelize | Must run sequentially |
| Code complexity | Medium | High |
| Debugging ease | Easy | Difficult |

***

## Best Practices and Pitfalls

### Critical Assumptions

**NULL handling:** This implementation assumes `scoring_class` and `is_active` are **never NULL**.[1]

**Why this matters:**
```sql
WHERE scoring_class != previous_scoring_class
-- NULL != NULL evaluates to NULL (not TRUE), filtering the row out
```

**Solutions:**
1. Enforce NOT NULL constraints on tracked columns
2. Use `IS DISTINCT FROM` operator (handles NULLs correctly)
3. COALESCE to default values before comparison

```sql
-- Option 1: IS DISTINCT FROM
WHERE scoring_class IS DISTINCT FROM previous_scoring_class

-- Option 2: COALESCE
WHERE COALESCE(scoring_class, 'unknown') != COALESCE(previous_scoring_class, 'unknown')
```

***

### Window Function Performance

**Problem:** Window functions on unsorted data cause expensive shuffles.[1]

**Optimization:**
- Pre-partition data by `player_name`
- Pre-sort by `current_season`
- Use appropriate cluster keys in production data warehouses

---

### Skew Issues

**Problem:** Players with many changes (like Aaron Brooks who changed every season) create disproportionate rows, causing data skew in group by operations.[1]

**Mitigation:**
- Monitor cardinality of `streak_identifier` per player
- Consider hybrid approach: use daily snapshots for rapidly changing dimensions
- Apply salting techniques for high-cardinality entities

***

### Primary Key Selection

**Correct:** `(player_name, start_season)`
**Incorrect:** `(player_name, current_season)`

**Why:** A player can have only one record per `start_season`, but during incremental updates, multiple records might share the same `current_season`.[1]

***

### Quality Checks

**Before deploying incremental SCD:**

```sql
-- Check 1: No overlapping date ranges
SELECT player_name, COUNT(*)
FROM players_scd
GROUP BY player_name
HAVING COUNT(DISTINCT start_season) != COUNT(*);

-- Check 2: No gaps in date ranges
WITH ordered_records AS (
    SELECT 
        player_name,
        end_season,
        LEAD(start_season) OVER (PARTITION BY player_name ORDER BY start_season) as next_start
    FROM players_scd
)
SELECT * FROM ordered_records
WHERE end_season + 1 != next_start AND next_start IS NOT NULL;

-- Check 3: Consistent value within date range
SELECT player_name, start_season, end_season, scoring_class
FROM players_scd ps1
WHERE EXISTS (
    SELECT 1 FROM players_scd ps2
    WHERE ps1.player_name = ps2.player_name
      AND ps1.start_season = ps2.start_season
      AND ps1.scoring_class != ps2.scoring_class
);
```

***

### Scale Considerations

**Airbnb scale** (millions of listings): Full history processing works fine[1]

**Facebook scale** (billions of users): Incremental processing necessary[1]

**Rule of thumb:** If full history query takes < 30 minutes and fits in memory, stick with simpler approach.

***

## Key Takeaways

**SCD Type 2 implementation requires careful consideration of trade-offs**:[1]

- **Full history:** Simple, self-correcting, parallelizable, but processes more data
- **Incremental:** Efficient, scalable, but complex and sequential

**Window functions are powerful** for change detection:[1]
- LAG identifies changes
- SUM creates streak identifiers
- MIN/MAX collapse streaks

**NULL handling is critical**—always validate assumptions about data quality.[1]

**Don't over-optimize prematurely**—ship business value first, optimize later if needed. Zach's unit economics SCD ran full history processing daily at Airbnb and worked great.[1]

**Dimensional data is smaller than fact data**—you can often afford to be less efficient with dimensions.[1]

**Remember:** The best data model is the one that ships and provides value to the business.[1]

[1](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/45417937/2d1ce770-2e70-462a-bb16-abab7a5e6224/paste.txt)
