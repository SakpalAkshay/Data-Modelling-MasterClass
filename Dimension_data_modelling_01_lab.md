# Data Modeling Lab - Cumulative Table Design with Structs and Arrays

A hands-on guide to building cumulative dimension tables using complex data types in SQL.

***

## Lab Setup

**Prerequisites:**
- Docker installed (to spin up PostgreSQL)
- SQL client (DataGrip, DB Visualizer, or DBeaver)
- GitHub repo cloned for homework exercises

**Tools used in this lab:**
- PostgreSQL with struct and array support
- DataGrip as SQL client

***

## The Problem: Temporal Cardinality

### Starting Table: `player_seasons`

**Current structure:**
```sql
-- One row per player per season
player_name | age | height | college | season | gp | points | rebounds | assists
AC Green    | 22  | 6-9    | Oregon  | 1996   | 82 | 8.5    | 7.2      | 1.1
AC Green    | 23  | 6-9    | Oregon  | 1997   | 82 | 9.2    | 7.5      | 1.3
AC Green    | 24  | 6-9    | Oregon  | 1998   | 82 | 8.8    | 7.1      | 1.2
```

**Issues:**
- **Massive duplication:** Player attributes (height, college, draft info) repeated for every season[1]
- **Shuffle problem:** Joining this table breaks sorting and destroys run-length encoding compression[1]
- **Temporal explosion:** Millions of rows when you only have thousands of unique players[1]

**Goal:** Create one row per player with seasons stored in an array.[1]

***

## Step 1: Create Custom Struct Type

Define a `season_stats` struct to hold seasonal statistics.[1]

```sql
-- Create custom type for season statistics
CREATE TYPE season_stats AS (
    season INTEGER,
    gp INTEGER,           -- games played
    points REAL,
    rebounds REAL,
    assists REAL
);
```

**Why struct?**
- Groups related fields of **different data types** together[1]
- Represents a "table within a table"
- Perfect for seasonal metrics that vary by year

***

## Step 2: Design Cumulative Table Schema

Create the `players` table with one row per player.[1]

```sql
CREATE TABLE players (
    player_name TEXT,
    height TEXT,
    college TEXT,
    country TEXT,
    draft_year TEXT,
    draft_round TEXT,
    draft_number TEXT,
    season_stats season_stats[],    -- Array of structs
    current_season INTEGER,
    PRIMARY KEY (player_name, current_season)
);
```

**Key design decisions:**

**Fixed dimensions** (player-level, don't change):
- `player_name`, `height`, `college`, `country`
- Draft information (year, round, number)

**Temporal dimensions** (season-level, stored in array):
- Season statistics in `season_stats` array

**Cumulation metadata:**
- `current_season`: Tracks which date the cumulation represents[1]
- Primary key: `(player_name, current_season)` ensures uniqueness per cumulation snapshot

***

## Step 3: The Seed Query (Full Outer Join Pattern)

Build the initial 1996 season using the full outer join pattern.[1]

### The Pattern

```sql
WITH yesterday AS (
    SELECT * FROM players
    WHERE current_season = 1995  -- This is NULL on first run (seed query)
),
today AS (
    SELECT * FROM player_seasons
    WHERE season = 1996
)
SELECT 
    -- Fixed dimensions (coalesce from today or yesterday)
    COALESCE(t.player_name, y.player_name) as player_name,
    COALESCE(t.height, y.height) as height,
    COALESCE(t.college, y.college) as college,
    COALESCE(t.country, y.country) as country,
    COALESCE(t.draft_year, y.draft_year) as draft_year,
    COALESCE(t.draft_round, y.draft_round) as draft_round,
    COALESCE(t.draft_number, y.draft_number) as draft_number,
    
    -- Temporal dimension (array building logic)
    CASE 
        WHEN y.season_stats IS NULL THEN 
            -- First season for this player
            ARRAY[ROW(t.season, t.gp, t.points, t.rebounds, t.assists)::season_stats]
        WHEN t.season IS NOT NULL THEN 
            -- Active player: append new season to history
            y.season_stats || ARRAY[ROW(t.season, t.gp, t.points, t.rebounds, t.assists)::season_stats]
        ELSE 
            -- Retired player: carry forward existing history
            y.season_stats
    END as season_stats,
    
    -- Metadata
    COALESCE(t.season, y.current_season + 1) as current_season
FROM today t
FULL OUTER JOIN yesterday y ON t.player_name = y.player_name;
```

### Logic Breakdown

**Case 1: New player (`y.season_stats IS NULL`)**
- Create initial array with single season
- Example: Player debuts in 1996

**Case 2: Active player (`t.season IS NOT NULL`)**
- Concatenate (`||`) new season to existing array
- Builds history incrementally
- Example: Player has 1996 record, now adding 1997

**Case 3: Retired player (`ELSE`)**
- Carry forward existing array without modification
- No new stats to add (player didn't play this season)
- Example: Player retired in 1998, we're processing 2000

***

## Step 4: Insert and Iterate

**First run (1996):**
```sql
INSERT INTO players
-- [Full query from Step 3]
-- Result: 450 players with single season in array
```

**Second run (1997):**
```sql
-- Change CTEs to:
WITH yesterday AS (
    SELECT * FROM players WHERE current_season = 1996
),
today AS (
    SELECT * FROM player_seasons WHERE season = 1997
)
-- [Same SELECT logic]
-- Result: Some players have 2 seasons, some have 1, new rookies added
```

**Pattern:**
Increment both `yesterday` and `today` by one year for each iteration.[1]

```
1995 → 1996 (seed)
1996 → 1997
1997 → 1998
1998 → 1999
...
2000 → 2001
```

***

## Step 5: Add Derived Metrics

Enhance the table with calculated dimensions.[1]

### Years Since Last Season

Tracks how long a player has been retired.[1]

```sql
-- Add to CREATE TABLE
years_since_last_season INTEGER

-- Add to SELECT query
CASE 
    WHEN t.season IS NOT NULL THEN 0  -- Played this season
    ELSE y.years_since_last_season + 1  -- Increment retirement counter
END as years_since_last_season
```

**Example:**
```
Michael Jordan:
- 1998: years_since_last_season = 0 (active)
- 1999: years_since_last_season = 1 (retired)
- 2000: years_since_last_season = 2 (still retired)
- 2001: years_since_last_season = 0 (comeback!)
```

***

### Scoring Class (Enumeration)

Categorize players based on performance.[1]

```sql
-- Create enum type
CREATE TYPE scoring_class AS ENUM ('star', 'good', 'average', 'bad');

-- Add to CREATE TABLE
scoring_class scoring_class

-- Add to SELECT query
CASE 
    WHEN t.season IS NOT NULL THEN
        -- Classify current season performance
        CASE 
            WHEN t.points > 20 THEN 'star'::scoring_class
            WHEN t.points > 15 THEN 'good'::scoring_class
            WHEN t.points > 10 THEN 'average'::scoring_class
            ELSE 'bad'::scoring_class
        END
    ELSE 
        -- Carry forward classification for retired players
        y.scoring_class
END as scoring_class
```

**Benefits:**
- Type safety: Only valid values allowed
- Compact storage: Enums stored as integers internally
- Clear semantics: Explicit categories

***

## Step 6: Querying Cumulative Tables

### Example 1: View Latest State

```sql
SELECT * 
FROM players 
WHERE current_season = 2001;
```

**Result:** One row per player with complete history through 2001.[1]

```
player_name     | season_stats                                           | scoring_class | years_since_last_season
Michael Jordan  | [{1996,82,30.4,6.6,4.3}, {1997,82,29.6,5.9,4.3}, ...]| star          | 0
AC Green        | [{1996,82,8.5,7.2,1.1}, {1997,82,9.2,7.5,1.3}, ...]  | average       | 0
```

***

### Example 2: Unnest to Explode Back to Rows

Convert array back to traditional row-per-season format.[1]

```sql
SELECT 
    player_name,
    (UNNEST(season_stats)).*
FROM players
WHERE current_season = 2001;
```

**Result:**
```
player_name     | season | gp | points | rebounds | assists
Michael Jordan  | 1996   | 82 | 30.4   | 6.6      | 4.3
Michael Jordan  | 1997   | 82 | 29.6   | 5.9      | 4.3
Michael Jordan  | 2001   | 60 | 22.9   | 5.7      | 5.2
AC Green        | 1996   | 82 | 8.5    | 7.2      | 1.1
...
```

**Critical insight:** After unnesting, all seasons for the same player remain **consecutive** and **sorted**. This preserves run-length encoding compression even after joins.[1]

***

### Example 3: Historical Analysis Without GROUP BY

Find players with biggest improvement from first to latest season.[1]

```sql
SELECT 
    player_name,
    (season_stats[1]).points as first_season_points,
    (season_stats[CARDINALITY(season_stats)]).points as latest_season_points,
    CASE 
        WHEN (season_stats[1]).points = 0 THEN 1
        ELSE (season_stats[CARDINALITY(season_stats)]).points / 
             (season_stats[1]).points
    END as improvement_ratio
FROM players
WHERE current_season = 2001
  AND scoring_class = 'star'
ORDER BY improvement_ratio DESC;
```

**Result:**
```
player_name      | first_season_points | latest_season_points | improvement_ratio
Tracy McGrady    | 7.0                 | 26.8                 | 3.83
Dirk Nowitzki    | 8.2                 | 21.8                 | 2.66
Kobe Bryant      | 7.6                 | 28.5                 | 3.75
```

**Why this is powerful:**
- **No GROUP BY needed**[1]
- **No shuffle operations**[1]
- **Infinitely parallelizable**[1]
- Access both first and last season with simple array indexing
- Query runs in milliseconds on billions of rows

***

## PostgreSQL-Specific Syntax

### Array Indexing
```sql
season_stats[1]                        -- First element
season_stats[CARDINALITY(season_stats)] -- Last element
```

### Accessing Struct Fields
```sql
-- Option 1: Direct access (requires cast)
(season_stats[1]).*

-- Option 2: Dot notation
(season_stats[1]).points

-- Option 3: Cast then access
(season_stats[1])::season_stats.*
```

### Array Concatenation
```sql
existing_array || ARRAY[new_element]
```

### Creating Structs from Rows
```sql
ROW(col1, col2, col3)::custom_type
```

***

## The Michael Jordan Example

Demonstrates handling player comebacks.[1]

**Career timeline:**
```sql
SELECT player_name, season_stats, years_since_last_season
FROM players
WHERE player_name = 'Michael Jordan';
```

**1998 (Last Bulls season):**
```
season_stats: [{1996,...}, {1997,...}, {1998,...}]
years_since_last_season: 0
```

**2000 (Retired):**
```
season_stats: [{1996,...}, {1997,...}, {1998,...}]  -- No new seasons added
years_since_last_season: 2
```

**2001 (Wizards comeback):**
```
season_stats: [{1996,...}, {1997,...}, {1998,...}, {2001,...}]  -- Gap preserved!
years_since_last_season: 0  -- Reset to active
```

The array naturally captures **career gaps** without special handling.[1]

---

## Key Advantages of Cumulative Design

### Historical Queries Without Shuffles

Traditional approach (requires GROUP BY):
```sql
SELECT player_name, 
       MIN(points) as first_season_points,
       MAX(points) as latest_season_points
FROM player_seasons
GROUP BY player_name;  -- Expensive shuffle!
```

Cumulative approach (no GROUP BY):
```sql
SELECT player_name,
       season_stats[1].points,
       season_stats[CARDINALITY(season_stats)].points
FROM players;  -- No shuffle, pure map operation
```

**Performance difference:** 100x+ faster on large datasets.[1]

---

### Compression Preservation Through Joins

**Problem:** Joining normalized tables shuffles data and breaks compression.[1]

**Solution:** Join at player level, unnest after.[1]

```sql
-- Step 1: Join at entity level (players, not seasons)
WITH enriched_players AS (
    SELECT p.*, t.team_city, t.team_name
    FROM players p
    JOIN teams t ON p.player_name = t.player_name
)

-- Step 2: Unnest preserves sorting
SELECT 
    player_name,
    team_city,
    (UNNEST(season_stats)).*
FROM enriched_players
WHERE current_season = 2001;
```

**Result:** All seasons for "AC Green" stay consecutive, maintaining run-length encoding.[1]

***

### Incrementally Buildable

**Daily pipeline:**
```sql
-- Yesterday: 2025-01-14
-- Today: 2025-01-15

WITH yesterday AS (SELECT * FROM players WHERE current_season = '2025-01-14'),
     today AS (SELECT * FROM daily_activity WHERE date = '2025-01-15')
-- [Full outer join logic]
```

**Backfill:**
Must run sequentially, one day at a time.[1]
```
Day 1 → Day 2 → Day 3 → ... → Day N
```

Cannot parallelize (each day depends on previous day's output).

***

## Trade-offs and Considerations

### Pros

**Query performance:**
- Historical analysis without GROUP BY (no shuffle)[1]
- Array indexing for first/last values
- Infinitely parallelizable queries

**Compression:**
- Preserves run-length encoding through joins[1]
- Compact storage (one row per entity)

**Flexibility:**
- Handle gaps (retirements, comebacks) naturally
- Easy state transition tracking

### Cons

**Backfill limitations:**
- Sequential only (cannot parallelize)[1]
- Slow for historical rebuilds
- Each day depends on previous day

**PII management:**
- Must actively prune deleted users[1]
- Requires separate deletion tracking
- Data grows indefinitely without maintenance

**Complexity:**
- More complex SQL syntax
- Requires understanding of structs/arrays
- Debugging harder than flat tables

***

## Best Practices

**When to use cumulative tables:**
- Master data layers for data engineers[1]
- Complete history needed per entity
- Temporal queries dominate (first/last, trends)
- Compression and performance critical

**When to avoid:**
- Direct analyst/scientist consumption (prefer flat OLAP cubes)[1]
- Real-time updates needed
- Backfills must be parallelizable
- Team unfamiliar with complex types

**Hybrid approach:**
- Cumulative master data (for engineers)
- Exploded OLAP cubes (for analysts)
- Best of both worlds

***

## Complete Working Example

```sql
-- 1. Create types
CREATE TYPE season_stats AS (
    season INTEGER,
    gp INTEGER,
    points REAL,
    rebounds REAL,
    assists REAL
);

CREATE TYPE scoring_class AS ENUM ('star', 'good', 'average', 'bad');

-- 2. Create table
CREATE TABLE players (
    player_name TEXT,
    height TEXT,
    college TEXT,
    country TEXT,
    draft_year TEXT,
    draft_round TEXT,
    draft_number TEXT,
    season_stats season_stats[],
    scoring_class scoring_class,
    years_since_last_season INTEGER,
    current_season INTEGER,
    PRIMARY KEY (player_name, current_season)
);

-- 3. Daily cumulation query
INSERT INTO players
WITH yesterday AS (
    SELECT * FROM players WHERE current_season = 2000
),
today AS (
    SELECT * FROM player_seasons WHERE season = 2001
)
SELECT 
    COALESCE(t.player_name, y.player_name) as player_name,
    COALESCE(t.height, y.height) as height,
    COALESCE(t.college, y.college) as college,
    COALESCE(t.country, y.country) as country,
    COALESCE(t.draft_year, y.draft_year) as draft_year,
    COALESCE(t.draft_round, y.draft_round) as draft_round,
    COALESCE(t.draft_number, y.draft_number) as draft_number,
    CASE 
        WHEN y.season_stats IS NULL THEN 
            ARRAY[ROW(t.season, t.gp, t.points, t.rebounds, t.assists)::season_stats]
        WHEN t.season IS NOT NULL THEN 
            y.season_stats || ARRAY[ROW(t.season, t.gp, t.points, t.rebounds, t.assists)::season_stats]
        ELSE y.season_stats
    END as season_stats,
    CASE 
        WHEN t.season IS NOT NULL THEN
            CASE 
                WHEN t.points > 20 THEN 'star'::scoring_class
                WHEN t.points > 15 THEN 'good'::scoring_class
                WHEN t.points > 10 THEN 'average'::scoring_class
                ELSE 'bad'::scoring_class
            END
        ELSE y.scoring_class
    END as scoring_class,
    CASE 
        WHEN t.season IS NOT NULL THEN 0
        ELSE y.years_since_last_season + 1
    END as years_since_last_season,
    COALESCE(t.season, y.current_season + 1) as current_season
FROM today t
FULL OUTER JOIN yesterday y ON t.player_name = y.player_name;
```

***

## Key Takeaways

**Cumulative table design** is a powerful pattern for building master data with complete history:[1]

- Use **full outer join** between yesterday and today to incrementally build history
- Store temporal dimensions in **arrays** to avoid cardinality explosion
- Use **structs** to group related seasonal metrics
- **No GROUP BY** needed for historical queries (massive performance gain)
- **Preserves sorting** through joins, maintaining compression
- Perfect middle ground between OLTP normalization and OLAP denormalization

**Remember:** Query without GROUP BY = infinitely parallelizable = blazing fast.[1]

[1](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/45417937/2dc205e4-2f09-4248-9cd0-b9884b883182/paste.txt)
