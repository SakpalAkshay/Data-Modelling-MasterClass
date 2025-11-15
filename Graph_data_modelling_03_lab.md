# Building an NBA Player Network Graph - Hands-On Lab

A practical guide to implementing graph data modeling in PostgreSQL, building vertices and edges to analyze player relationships and performance patterns.

---

## Table of Contents

- Lab Overview and Setup
- Creating Graph Schema (Vertices and Edges)
- Building Vertices (Players, Teams, Games)
- Building Edges (Relationships)
- Advanced Graph Queries
- Performance Patterns

***

## Lab Overview and Setup

**Goal:** Build a graph database to analyze NBA player relationships including:
- Which players play with each other
- Which players play against each other
- Which teams players belong to
- Game-level performance comparisons

**Tools:**
- PostgreSQL (via Docker)
- SQL client (DataGrip, DB Visualizer, or DBeaver)
- GitHub repo with source data

**Source tables:**
- `game_details` - Player statistics per game
- `games` - Game-level information
- `teams` - Team metadata

***

## Creating Graph Schema

### Step 1: Define Vertex Types

Enumerate all entity types in the graph:[1]

```sql
CREATE TYPE vertex_type AS ENUM (
    'player',
    'team', 
    'game'
);
```

**Why these three?**
- **Player:** Individual athletes
- **Team:** Organizations players belong to
- **Game:** Events where players compete

**Note:** Arena was considered but excluded because it has a 1:1 relationship with teams (redundant).[1]

---

### Step 2: Create Vertices Table

**Universal vertex schema** - works for any graph:[1]

```sql
CREATE TABLE vertices (
    identifier TEXT,           -- Unique ID for entity
    type vertex_type,          -- Which enum value
    properties JSON,           -- Flexible metadata (PostgreSQL doesn't have MAP)
    PRIMARY KEY (identifier, type)
);
```

**Why JSON instead of MAP?** PostgreSQL lacks a native MAP type, so JSON serves as flexible key-value storage.[1]

---

### Step 3: Define Edge Types

Enumerate all relationship types (verbs connecting entities):[1]

```sql
CREATE TYPE edge_type AS ENUM (
    'plays_against',  -- Player vs player (different teams)
    'shares_team',    -- Player + player (same team)
    'plays_in',       -- Player participated in game
    'plays_on'        -- Player belongs to team
);
```

**Edge type naming convention:** Use verb phrases that complete "Subject [edge_type] Object".[1]

***

### Step 4: Create Edges Table

**Universal edge schema** - seven columns for any graph:[1]

```sql
CREATE TABLE edges (
    subject_identifier TEXT,
    subject_type vertex_type,
    object_identifier TEXT,
    object_type vertex_type,
    edge_type edge_type,
    properties JSON,
    PRIMARY KEY (subject_identifier, subject_type, 
                 object_identifier, object_type, edge_type)
);
```

**Why such a long primary key?** Ensures uniqueness for each specific relationship. Same two entities can have multiple edge types.[1]

**Alternative:** Some graph databases use a surrogate `edge_id` for simplicity, but this lab uses natural keys.[1]

---

## Building Vertices

### Vertex 1: Games

**Easiest to start** - already de-duplicated, no aggregation needed.[1]

```sql
INSERT INTO vertices
SELECT 
    game_id AS identifier,
    'game'::vertex_type AS type,
    JSON_BUILD_OBJECT(
        'points_home', points_home,
        'points_away', points_away,
        'winning_team', CASE 
            WHEN home_team_wins = 1 THEN home_team_id 
            ELSE visitor_team_id 
        END
    ) AS properties
FROM games;
```

**Result:** ~9,300 game vertices.[1]

**Properties stored:**
- Home/away scores
- Winning team ID

***

### Vertex 2: Players

**Requires aggregation** - pull career statistics from `game_details`.[1]

```sql
WITH players_agg AS (
    SELECT 
        player_id AS identifier,
        MAX(player_name) AS player_name,  -- Dedupes name variations
        COUNT(1) AS number_of_games,
        SUM(points) AS total_points,
        ARRAY_AGG(DISTINCT team_id) AS teams  -- All teams played for
    FROM game_details
    GROUP BY player_id
)
INSERT INTO vertices
SELECT 
    identifier,
    'player'::vertex_type AS type,
    JSON_BUILD_OBJECT(
        'player_name', player_name,
        'number_of_games', number_of_games,
        'total_points', total_points,
        'teams', teams
    ) AS properties
FROM players_agg;
```

**Why MAX(player_name)?** Player names might have slight variations (Dennis Rodman vs Dennis K. Rodman). MAX picks one consistently.[1]

**Result:** ~1,500 player vertices.[1]

---

### Vertex 3: Teams

**Handling duplicates** - source data had unexpected triplicates.[1]

```sql
WITH teams_deduped AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY team_id) AS r_num
    FROM teams
)
INSERT INTO vertices
SELECT 
    team_id AS identifier,
    'team'::vertex_type AS type,
    JSON_BUILD_OBJECT(
        'abbreviation', abbreviation,
        'nickname', nickname,
        'city', city,
        'arena', arena,
        'year_founded', year_founded
    ) AS properties
FROM teams_deduped
WHERE r_num = 1;  -- Keep only first occurrence
```

**Deduplication pattern:** Use `ROW_NUMBER()` window function to identify duplicates, then filter to `r_num = 1`.[1]

**Result:** 30 team vertices.[1]

***

## Building Edges

### Edge 1: Plays In (Player → Game)

**Simplest edge** - already at correct grain in `game_details`.[1]

```sql
WITH deduped AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY player_id, game_id
        ) AS r_num
    FROM game_details
)
INSERT INTO edges
SELECT 
    player_id AS subject_identifier,
    'player'::vertex_type AS subject_type,
    game_id AS object_identifier,
    'game'::vertex_type AS object_type,
    'plays_in'::edge_type AS edge_type,
    JSON_BUILD_OBJECT(
        'start_position', start_position,
        'points', points,
        'team_id', team_id,
        'team_abbreviation', team_abbreviation
    ) AS properties
FROM deduped
WHERE r_num = 1;
```

**Why dedupe again?** Data import created unintended duplicates (3x of everything). This pattern ensures idempotency.[1]

**Properties stored:** Game-specific context like position, points, team played for.

***

### Edge 2: Plays Against + Shares Team (Player ↔ Player)

**Most complex edge** - requires self-join and conditional logic.[1]

#### The Self-Join Pattern

```sql
WITH filtered AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY player_id, game_id
        ) AS r_num
    FROM game_details
    WHERE r_num = 1
),
aggregated AS (
    SELECT 
        f1.player_id AS subject_player_id,
        MAX(f1.player_name) AS subject_player_name,  -- Handle name variations
        f2.player_id AS object_player_id,
        MAX(f2.player_name) AS object_player_name,
        CASE 
            WHEN f1.team_abbreviation = f2.team_abbreviation 
            THEN 'shares_team'::edge_type
            ELSE 'plays_against'::edge_type
        END AS edge_type,
        COUNT(1) AS num_games,
        SUM(f1.points) AS subject_points,
        SUM(f2.points) AS object_points
    FROM filtered f1
    JOIN filtered f2 
        ON f1.game_id = f2.game_id           -- Same game
        AND f1.player_name != f2.player_name -- Different players
        AND f1.player_id > f2.player_id      -- Prevent duplicate edges
    GROUP BY f1.player_id, f2.player_id, edge_type
)
INSERT INTO edges
SELECT 
    subject_player_id AS subject_identifier,
    'player'::vertex_type AS subject_type,
    object_player_id AS object_identifier,
    'player'::vertex_type AS object_type,
    edge_type,
    JSON_BUILD_OBJECT(
        'num_games', num_games,
        'subject_points', subject_points,
        'object_points', object_points
    ) AS properties
FROM aggregated;
```

#### Key Techniques

**1. Self-join creates player pairs**
```sql
FROM filtered f1
JOIN filtered f2 ON f1.game_id = f2.game_id
```
Every player in game matched with every other player in same game.

***

**2. Conditional edge type assignment**
```sql
CASE 
    WHEN f1.team_abbreviation = f2.team_abbreviation 
    THEN 'shares_team'
    ELSE 'plays_against'
END
```
Same game but different teams → `plays_against`
Same game and same team → `shares_team`

***

**3. Preventing duplicate edges**
```sql
WHERE f1.player_id > f2.player_id
```

**Problem:** Self-join creates two edges for each relationship:
- Michael Jordan → Scottie Pippen
- Scottie Pippen → Michael Jordan

**Solution:** String comparison ensures only one direction is kept. Since player_id is consistent, this creates deterministic single edges.[1]

---

**4. Handling name variations**
```sql
MAX(f1.player_name) AS subject_player_name
```

Player names might vary slightly in data. Using MAX in GROUP BY picks one consistent version.[1]

***

**Example result:**
```sql
-- Tim Duncan and Tony Parker played 142 games together
subject_identifier: 1234 (Tim Duncan)
object_identifier: 5678 (Tony Parker)
edge_type: 'shares_team'
properties: {num_games: 142, subject_points: 2840, object_points: 1980}
```

***

## Advanced Graph Queries

### Query 1: Basic Vertex Counting

```sql
SELECT type, COUNT(*) 
FROM vertices 
GROUP BY type;
```

**Result:**
```
type    | count
--------|------
game    | 9,300
player  | 1,500
team    | 30
```

***

### Query 2: Player Career Stats via Join

```sql
SELECT 
    v.properties->>'player_name' AS player_name,
    (v.properties->>'number_of_games')::INTEGER AS games,
    (v.properties->>'total_points')::REAL AS total_points,
    (v.properties->>'total_points')::REAL / 
        NULLIF((v.properties->>'number_of_games')::REAL, 0) AS ppg
FROM vertices v
WHERE v.type = 'player'
ORDER BY ppg DESC
LIMIT 10;
```

**JSON extraction syntax:** `properties->>'key'` extracts value as text.[1]

**Type casting:** Must cast JSON strings to numbers for arithmetic.[1]

---

### Query 3: Player Performance by Opponent

**The power of graph modeling** - compare individual matchup performance to career average.[1]

```sql
SELECT 
    v.properties->>'player_name' AS player_name,
    e.object_identifier AS opponent_id,
    -- Career average
    (v.properties->>'total_points')::REAL / 
        NULLIF((v.properties->>'number_of_games')::REAL, 0) AS career_ppg,
    -- Matchup average
    (e.properties->>'subject_points')::REAL / 
        NULLIF((e.properties->>'num_games')::REAL, 0) AS matchup_ppg,
    (e.properties->>'num_games')::INTEGER AS games_vs_opponent
FROM vertices v
JOIN edges e 
    ON v.identifier = e.subject_identifier
    AND v.type = e.subject_type
WHERE e.edge_type = 'plays_against'
    AND v.properties->>'player_name' LIKE '%Jordan%'
ORDER BY games_vs_opponent DESC;
```

**Insight:** See if Michael Jordan performs better/worse against specific opponents compared to his career average.[1]

**Example output:**
```
player_name      | opponent_id | career_ppg | matchup_ppg | games_vs_opponent
Michael Jordan   | 5678        | 30.1       | 33.2        | 28
Michael Jordan   | 1234        | 30.1       | 27.8        | 24
```

Jordan averages 33.2 PPG against opponent 5678 (above his 30.1 career average) but only 27.8 against opponent 1234.

***

### Query 4: Aggregation Performance - The Speed Benefit

**Why pre-aggregate?** Compare query speeds:

**Before (raw game_details scan):**
```sql
SELECT 
    f1.player_name,
    f2.player_name AS opponent,
    AVG(f1.points) AS avg_points
FROM game_details f1
JOIN game_details f2 
    ON f1.game_id = f2.game_id 
    AND f1.team_id != f2.team_id
GROUP BY f1.player_name, f2.player_name;

-- Runtime: 10+ seconds (scanning millions of rows)
```

**After (using graph edges):**
```sql
SELECT 
    subject_identifier,
    object_identifier,
    (properties->>'subject_points')::REAL / 
        (properties->>'num_games')::REAL AS avg_points
FROM edges
WHERE edge_type = 'plays_against';

-- Runtime: <1 second (pre-aggregated, smaller dataset)
```

**Performance gain:** 10x+ faster because the heavy aggregation already happened during edge creation.[1]

***

## Common Pitfalls and Solutions

### Pitfall 1: Data Type Issues with JSON

**Problem:** JSON stores everything as text. Arithmetic fails without casting.[1]

```sql
-- ❌ Wrong - treats as string, returns '9' (max alphabetically)
SELECT MAX(properties->>'points') FROM edges;

-- ✅ Correct - casts to integer first
SELECT MAX((properties->>'points')::INTEGER) FROM edges;
```

***

### Pitfall 2: NULL Handling in CASE Expressions

**Problem:** `NULL != NULL` evaluates to NULL (not TRUE), causing rows to be filtered out.[1]

```sql
-- ❌ Wrong - filters out NULLs
WHERE f1.team_abbreviation != f2.team_abbreviation

-- ✅ Better - explicitly handle NULLs
WHERE f1.team_abbreviation IS DISTINCT FROM f2.team_abbreviation
```

***

### Pitfall 3: Duplicate Edges from Self-Joins

**Problem:** Self-join creates bidirectional edges (A→B and B→A).[1]

**Solution:** String comparison to keep only one direction:
```sql
WHERE f1.player_id > f2.player_id  -- Deterministic, keeps only one
```

**Alternative:** Could keep both edges to simplify queries (always start from subject), but doubles storage.[1]

***

### Pitfall 4: GROUP BY with Enums

**Problem:** Forgot to include enum in SELECT but not in GROUP BY causes errors.[1]

**Anti-pattern:**
```sql
SELECT col1, col2, col3, col4, col5
FROM table
GROUP BY 1, 2, 3, 4, 5;  -- Fragile! Reordering breaks query
```

**Best practice:**
```sql
SELECT col1, col2, col3, col4, col5
FROM table
GROUP BY col1, col2, col3, col4, col5;  -- Explicit, safe
```

"Don't do this in production" - positional GROUP BY is coding interview shorthand only.[1]

***

## Performance Patterns

### Pattern 1: Deduplicate Early

**Apply deduplication immediately** after reading source tables:[1]

```sql
WITH deduped AS (
    SELECT *, 
        ROW_NUMBER() OVER (PARTITION BY key_columns) AS r_num
    FROM source_table
)
SELECT * FROM deduped WHERE r_num = 1;
```

**Benefit:** Reduces data volume for all downstream operations.

---

### Pattern 2: Aggregate Once, Query Many

**Heavy aggregation during edge creation**, simple queries after:[1]

```sql
-- Expensive: Self-join + aggregation (run once during ETL)
CREATE TABLE edges AS 
SELECT ...
FROM filtered f1
JOIN filtered f2 ...
GROUP BY ...;

-- Cheap: Simple scan (run thousands of times by analysts)
SELECT * FROM edges WHERE edge_type = 'plays_against';
```

***

### Pattern 3: Pre-compute Derived Metrics

**Include calculated fields** in properties during insert:[1]

```sql
JSON_BUILD_OBJECT(
    'total_points', SUM(points),
    'games', COUNT(*),
    'ppg', SUM(points) / COUNT(*)  -- Pre-compute average
)
```

**Trade-off:** Larger storage, but instant retrieval of common metrics.

***

## Key Takeaways

**Graph modeling shifts focus** from entities to relationships:[1]
- Traditional: "What attributes does Player have?"
- Graph: "How is Player connected to Game, Team, other Players?"

**Universal schema is actually simple**:[1]
- **Vertices:** 3 columns (identifier, type, properties)
- **Edges:** 7 columns (subject_id, subject_type, object_id, object_type, edge_type, properties, PK)

**Self-joins are powerful but tricky**:[1]
- Enable same-table relationships (player vs player)
- Require careful filtering to avoid duplicate edges
- String comparison (`f1.id > f2.id`) creates deterministic single edges

**Enums prevent data quality issues**:[1]
- Invalid edge types cause pipeline failures (good!)
- Better than silent corruption from typos

**Pre-aggregation is the performance secret**:[1]
- Heavy lifting during ETL (run once daily)
- Lightning-fast analytical queries (run thousands of times)
- 10-100x speedup compared to raw data scans

**JSON flexibility comes with trade-offs**:[1]
- ✅ No schema changes needed for new properties
- ❌ Terrible compression (column names stored per row)
- ❌ Must cast types for arithmetic

**Remember:** The same universal schema (vertices + edges) powers graphs at Netflix (70-80 vertex types), Facebook (cross-app user identity), and Airbnb (unit economics).[1]

[1](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/45417937/545be11f-7e74-426f-b8cd-e8e777e1c004/paste.txt)
