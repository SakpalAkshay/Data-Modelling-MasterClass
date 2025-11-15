# Data Modeling - Graph Databases, Additive Dimensions & Flexible Schemas

A comprehensive guide to graph data modeling, understanding additive vs non-additive dimensions, leveraging enumerations, and flexible schema design patterns.

***

## Table of Contents

- Additive vs Non-Additive Dimensions
- The Power of Enumerations (Enums)
- The Little Book of Pipelines Pattern
- Flexible Schema Design
- Graph Data Modeling
- Practical Applications

***

## Additive vs Non-Additive Dimensions

### What is Additivity?

**Definition:** A dimension is **additive** over a specific time window if the grain of data over that window can only ever be one value at a time.[1]

**In simple terms:** Can you sum subtotals to get the correct grand total?

### Example 1: Age (Additive)

**Scenario:** US population by age[1]

```
All 1-year-olds: 4 million
All 2-year-olds: 4 million
All 3-year-olds: 4 million
...
All 115-year-olds: 100

Total US population = Sum of all age groups ✅
```

**Why it works:** A person can only be one age at a time. No overlap between groups.

***

### Example 2: Car Model (Non-Additive)

**Scenario:** Honda drivers by car model[1]

```
Civic drivers: 1,000,000
Accord drivers: 800,000
Corolla drivers: 600,000  -- Wait, that's Toyota!

Number of Honda drivers ≠ Civic + Accord + Corolla ❌
```

**Why it fails:** A person can own and drive multiple Honda models (Civic AND Accord). They'd be counted twice.[1]

**The fix:** Change the metric!

```
Number of Honda CARS = Civic cars + Accord cars + Corolla cars ✅
```

A car can't be two models simultaneously, so this is additive.[1]

---

### Time Scale Matters

**Key insight:** Additivity depends on the time window.[1]

**Example: Honda drivers over 1 second**
```
In a 1-second window:
- You can only drive ONE car
- Honda drivers = Civic drivers + Accord drivers ✅
```

**Example: Honda drivers over 1 day**
```
In a 1-day window:
- You can drive multiple cars
- Honda drivers ≠ Civic drivers + Accord drivers ❌
```

**Rule of thumb:** Can a user have two dimensional values simultaneously within the time window? If yes, it's non-additive.[1]

***

### The Essential Nature of Additivity

**Mathematical definition:** A dimension is additive over a specific time window **if and only if** the grain of data over that window can only ever be one value at a time.[1]

**Test question:** Can a user be two of these at the same time in a given day?[1]
- If YES → Non-additive (requires COUNT DISTINCT)
- If NO → Additive (can use simple SUM)

---

### How Additivity Helps

**Additive dimensions allow pre-aggregation:**

```sql
-- Additive: Use pre-aggregated subtotals
SELECT SUM(age_group_count) as total_population
FROM population_by_age;
-- Fast! Just sum the subtotals
```

**Non-additive dimensions require raw data:**

```sql
-- Non-additive: Must go back to row-level
SELECT COUNT(DISTINCT driver_id) as total_honda_drivers
FROM car_ownership
WHERE brand = 'Honda';
-- Slower! Must scan all rows and deduplicate
```

**Performance impact:** Additive aggregations can use partially aggregated data. Non-additive must scan full datasets.[1]

---

### Impact on Different Aggregation Types

**SUM aggregations:** Usually additive[1]

```sql
-- Miles driven is additive
Miles driven by Honda drivers = 
    Miles driven by Civic drivers + 
    Miles driven by Accord drivers ✅
```

You can't drive a mile in two cars simultaneously.[1]

**COUNT aggregations:** Often non-additive[1]

```sql
-- User counts are non-additive
App users ≠ iPhone users + Android users ❌
```

Some users have both devices.[1]

**RATIO metrics:** Inherit non-additivity from COUNT[1]

```sql
-- Ratio with COUNT in denominator
Miles per driver = Total miles / COUNT(DISTINCT drivers)
-- Non-additive due to COUNT DISTINCT
```

***

### Common Non-Additive Dimensions

**"User's tool" dimensions are typically non-additive:**[1]

- Device type (iPhone, Android)
- Car model (Civic, Accord)
- Browser (Chrome, Safari, Firefox)
- App version (v2.0, v3.0)

**Why:** Users can have multiple "tools" and switch between them within a time window.

---

### Facebook's Solution: Milky Way Framework

At Facebook, this problem was so pervasive they built a framework called **Milky Way** to handle non-additive aggregations efficiently.[1]

**What it solved:**
- Automatic detection of non-additive dimensions
- Efficient COUNT DISTINCT operations
- Prevented analysts from accidentally double-counting users

**Lesson:** At scale, non-additive dimensions require special infrastructure support.

***

## The Power of Enumerations (Enums)

### What Are Enums?

**Definition:** A data type with a finite, predefined set of possible values.[1]

**Example from NBA data:**
```sql
CREATE TYPE scoring_class AS ENUM ('star', 'good', 'average', 'bad');
```

***

### When to Use Enums

**Rule of thumb:** Less than 50 possible values[1]

**Good candidates:**
- Scoring class (4 values: star, good, average, bad)
- Notification channel (4 values: SMS, email, push, logged_out_push)[1]
- Payment status (5 values: pending, approved, declined, refunded, canceled)

**Bad candidates:**
- Country (200+ values - too many!)[1]
- User ID (millions of values)
- Product SKU (thousands of values)

---

### Benefits of Enums

**1. Built-in Data Quality**[1]

When invalid values appear, the pipeline **fails** rather than silently corrupting data.

```sql
-- Enum defined
CREATE TYPE channel AS ENUM ('sms', 'email', 'push');

-- Invalid value causes pipeline failure
INSERT INTO notifications VALUES ('telegram');  -- ERROR: invalid enum value

-- Forces data quality at ingestion
```

***

**2. Built-in Static Fields**[1]

Enums can carry additional metadata in code.

**Example from Airbnb unit economics:**[1]

```python
class LineItem(Enum):
    FEES = "fees"
    COUPONS = "coupons"
    INFRASTRUCTURE_COST = "infrastructure_cost"
    
    @property
    def revenue_or_cost(self):
        if self == LineItem.FEES:
            return "revenue"
        elif self == LineItem.COUPONS:
            return "cost"
        elif self == LineItem.INFRASTRUCTURE_COST:
            return "cost"
```

**Benefit:** Static metadata shipped with code—more efficient than broadcast joins.[1]

---

**3. Built-in Documentation**[1]

Enums provide self-documenting schemas.

```sql
-- What values are possible?
SELECT enum_range(NULL::scoring_class);
-- Result: {star, good, average, bad}
```

With plain strings, you'd need to query the data itself to discover possible values.

***

**4. Perfect for Sub-Partitioning**[1]

Enums enable exhaustive partition management.

**Example: Notifications at Facebook**[1]

```sql
-- Partition by date AND channel
CREATE TABLE notifications_partitioned (
    ...
) PARTITIONED BY (date, channel);

-- channel is enum ('sms', 'email', 'push', 'logged_out_push')
```

**Why this works:**
- Know all possible partitions upfront
- Can wait for ALL partitions before processing downstream
- No missing data due to unknown partition values

**Pipeline dependency check:**
```python
# Wait for all enum values before proceeding
wait_for_partitions = [
    f"notifications/date=2025-01-15/channel={channel}"
    for channel in ChannelEnum
]
```

***

### Thrift: Sharing Enums Across Systems

**Thrift** is a schema management framework that shares enums between logging and ETL layers.[1]

**Benefits:**
- Logging code and pipeline code use same enum definitions
- Changes to enum automatically propagate
- Prevents schema drift between systems

**Example:**
```thrift
// Shared enum definition
enum NotificationChannel {
    SMS = 1,
    EMAIL = 2,
    PUSH = 3,
    LOGGED_OUT_PUSH = 4
}
```

Both application logging AND data pipelines reference this single source of truth.

***

## The Little Book of Pipelines Pattern

### The Problem

You're building a pipeline that integrates data from **many disparate sources** (30-50+ tables):[1]
- Different schemas
- Different data quality characteristics
- Different source systems
- All need to combine into ONE shared schema

**Challenge:** How do you manage this complexity without creating a 500-column table that's 90% NULL?

---

### The Solution: Enumerated Pipeline Pattern

**Core concept:** Group all data sources into an **enumerated list**, then use that list to drive pipeline logic.[1]

**Architecture:**

```
Source 1 ─┐
Source 2 ─┼─→ Little Book of Enums ─→ Shared Logic ETL ─→ DQ Checks ─→ Final Output
Source 3 ─┘       (enum values)           (maps to           (custom per
Source N ─┘                               shared schema)      enum value)
```

***

### How It Works

**Step 1: Define enumeration of all sources**

```python
class UnitEconomicsLineItem(Enum):
    FEES = "fees"
    COUPONS = "coupons"
    CREDITS = "credits"
    INSURANCE = "insurance"
    INFRASTRUCTURE_COST = "infrastructure_cost"
    TAXES = "taxes"
    REFERRAL_CREDIT = "referral_credit"
```

***

**Step 2: Each enum value has a source function**

```python
def get_fees_data(date):
    return spark.sql("""
        SELECT booking_id, fee_amount as amount, 'revenue' as type
        FROM payments.fees WHERE date = '{date}'
    """)

def get_coupons_data(date):
    return spark.sql("""
        SELECT booking_id, coupon_amount as amount, 'cost' as type
        FROM promotions.coupons WHERE date = '{date}'
    """)
```

Each source function maps disparate schemas to **shared schema** (booking_id, amount, type).

***

**Step 3: Shared logic processes all sources identically**

```python
for line_item in UnitEconomicsLineItem:
    # Get data from source function
    df = line_item.source_function(date)
    
    # Apply shared transformations
    df = apply_currency_conversion(df)
    df = apply_accounting_rules(df)
    
    # Run custom DQ checks per enum value
    dq_check = line_item.data_quality_check
    if not dq_check(df):
        raise Exception(f"DQ failed for {line_item}")
    
    # Write to partitioned output
    df.write.mode("overwrite").partitionBy("date", "line_item").save(output_path)
```

***

**Step 4: Output is sub-partitioned by enum**

```
output/
├── date=2025-01-15/
│   ├── line_item=fees/
│   ├── line_item=coupons/
│   ├── line_item=credits/
│   ├── line_item=insurance/
│   └── line_item=infrastructure_cost/
```

***

### Key Features

**Custom DQ checks per enum value:**[1]

Even though schema is shared, data quality expectations differ:

```python
class LineItem(Enum):
    FEES = ("fees", validate_fees)
    COUPONS = ("coupons", validate_coupons)
    
def validate_fees(df):
    # Fees should be positive and between $0-$1000
    return df.filter((df.amount > 0) & (df.amount < 1000)).count() == df.count()

def validate_coupons(df):
    # Coupons should be negative and between -$500-$0
    return df.filter((df.amount < 0) & (df.amount > -500)).count() == df.count()
```

**Benefits:**
- Shared schema for querying
- Custom validation per partition
- Enum provides documentation of all possible values

***

### Generating the "Little Book"

**Convert enum to table:**[1]

```python
# Python enum
class LineItem(Enum):
    FEES = "fees"
    COUPONS = "coupons"

# Convert to table (tiny, maybe 20-40 rows)
little_book_df = spark.createDataFrame([
    {"line_item": item.value, 
     "source_table": item.source_table,
     "dq_threshold": item.dq_threshold}
    for item in LineItem
])

little_book_df.write.mode("overwrite").save("little_book_table")
```

**Usage:**
- **Python code:** Pass enum directly to source functions
- **SQL code:** Join against little_book_table for DQ checks and thresholds

***

### Real-World Use Cases

**1. Airbnb Unit Economics**[1]

```
Enum values: fees, coupons, credits, insurance, infrastructure_cost, taxes, referral_credit
Sources: 40+ different tables
Output: One table with shared schema, partitioned by line_item
```

***

**2. Netflix Infrastructure Graph**[1]

```
Enum values: applications, databases, servers, codebases, cicd_jobs
Sources: 70-80 different infrastructure systems
Output: Graph database with nodes/edges, partitioned by vertex_type
```

***

**3. Facebook Family of Apps**[1]

```
Enum values: facebook, instagram, whatsapp, messenger, oculus, threads
Sources: Each app's user database
Output: Unified user table, partitioned by app
```

***

### When to Use This Pattern

**Indicators you need this:**
- Pipeline has 30+ upstream dependencies
- High "variety" problem (many different schemas to integrate)
- Need to maintain schema consistency across many sources
- Want to parallelize processing by partition

**Benefits:**
- Adding new source = add enum value + source function
- Everything else (DQ, output, orchestration) automatically works
- Self-documenting (query enum to see all possible partitions)
- Scalable (process partitions in parallel)

***

## Flexible Schema Design

### The Problem

You're integrating 40 tables into one shared schema. Do you:
- **Option A:** Create 500 columns (one from each source table) → 90% NULL values
- **Option B:** Use flexible schema with maps → Compact and extensible

**Answer:** Option B with flexible schemas.[1]

***

### What is a Flexible Schema?

**Definition:** Schema that can accommodate varying column sets without ALTER TABLE statements.[1]

**Primary tool:** The **MAP** data type

```sql
-- Rigid schema (400 columns)
CREATE TABLE products (
    product_id INT,
    col1 TEXT, col2 TEXT, col3 TEXT, ... col400 TEXT  -- Nightmare!
);

-- Flexible schema (4 columns)
CREATE TABLE products (
    product_id INT,
    product_type TEXT,  -- Enum!
    core_attributes STRUCT<name TEXT, price DECIMAL>,
    other_properties MAP<TEXT, TEXT>  -- Flexible!
);
```

***

### When to Use Flexible Schemas

**Scenario 1: Many disparate sources with different columns**

```sql
-- Electronics products
{color: "black", warranty_years: 2, voltage: "110V"}

-- Clothing products
{color: "blue", size: "M", material: "cotton"}

-- Food products
{expiration_date: "2025-12-31", calories: 250, allergens: "nuts"}
```

All can store in `other_properties MAP<TEXT, TEXT>` without schema changes.

***

**Scenario 2: Low-use columns**

Columns queried <1% of the time can go into `other_properties` map rather than bloating the core schema.[1]

---

### Advantages of Flexible Schemas

**1. No ALTER TABLE needed**[1]

```sql
-- Just add new key to map
INSERT INTO products VALUES (
    123,
    'electronics',
    ROW('Laptop', 999.99),
    MAP('color', 'silver', 'new_feature', 'touchscreen')  -- New column!
);
```

***

**2. Manage many more columns**[1]

**Limit:** Maps support up to 65,000 keys (Java byte length limit)[1]
**Reality:** Most maps have 100-1000 keys
**Benefit:** Far exceeds practical column limits of rigid schemas

***

**3. No NULL explosion**[1]

With rigid schema:
```
product_id | color | warranty_years | size | material | expiration_date | calories
123        | black | 2             | NULL | NULL     | NULL           | NULL
```

With flexible schema:
```
product_id | other_properties
123        | {color: black, warranty_years: 2}
```

Only present keys are stored—no NULLs.

***

### Disadvantages of Flexible Schemas

**1. Terrible compression**[1]

**Why:** Column names stored as data in every row instead of once in schema.

**Example:**
```
Rigid schema:
Header: [product_id, color, price]  -- Stored once
Row 1:  [123, "black", 999.99]
Row 2:  [124, "white", 899.99]
Row 3:  [125, "silver", 1099.99]

Flexible schema (map):
Row 1:  [123, {"color": "black", "price": 999.99}]     -- "color" and "price" stored
Row 2:  [124, {"color": "white", "price": 899.99}]     -- "color" and "price" stored again!
Row 3:  [125, {"color": "silver", "price": 1099.99}]   -- "color" and "price" stored again!
```

**Impact:** Maps and JSON have the worst compression of all data types.[1]

---

**2. More complex queries**

```sql
-- Rigid schema
SELECT color FROM products WHERE color = 'black';

-- Flexible schema
SELECT other_properties['color'] FROM products 
WHERE other_properties['color'] = 'black';
```

***

### Best Practice: Hybrid Approach

**Core columns:** Use rigid schema (good compression, easy queries)
**Variable columns:** Use maps (flexibility)

```sql
CREATE TABLE products (
    product_id INT,           -- Core: rigid
    product_type TEXT,        -- Core: rigid (enum!)
    name TEXT,                -- Core: rigid
    price DECIMAL,            -- Core: rigid
    other_properties MAP<TEXT, TEXT>  -- Variable: flexible
);
```

**Rule:** If column is queried frequently (>10% of queries), make it rigid. Otherwise, map.[1]

---

## Graph Data Modeling

### What Makes Graph Modeling Different?

**Traditional data modeling:** Entity-focused (what are the attributes?)
**Graph data modeling:** Relationship-focused (how are things connected?)[1]

**Shift in mindset:**
- From "What columns does User have?"
- To "How is User connected to Product, Device, Location?"

***

### The Universal Graph Schema

**Secret sauce:** Every graph database uses the same schema. Master this once, apply everywhere.[1]

---

### Vertices (Nodes)

**Schema:** Exactly 3 columns[1]

```sql
CREATE TABLE vertices (
    identifier TEXT,           -- Unique ID
    type TEXT,                -- Node type (enum!)
    properties MAP<TEXT, TEXT> -- Flexible attributes
);
```

**Example:**

```sql
-- Player vertex
identifier | type    | properties
'Michael Jordan' | 'player' | {height: '6-6', position: 'SG', draft_year: '1984'}

-- Team vertex
'Chicago Bulls' | 'team' | {city: 'Chicago', founded: '1966', championships: '6'}
```

**Key insight:** We don't care about individual columns. We care about the **type** and **connections**.[1]

***

### Edges (Relationships)

**Schema:** 7 columns[1]

```sql
CREATE TABLE edges (
    subject_identifier TEXT,   -- Source node ID
    subject_type TEXT,         -- Source node type
    object_identifier TEXT,    -- Target node ID
    object_type TEXT,          -- Target node type
    edge_type TEXT,            -- Relationship type (verb!)
    properties MAP<TEXT, TEXT>, -- Relationship attributes
    PRIMARY KEY (subject_identifier, subject_type, object_identifier, edge_type)
);
```

***

### Edge Type as Verb

**Pattern:** Think subject-verb-object sentences[1]

```
Michael Jordan [plays on] Chicago Bulls
Michael Jordan [plays against] John Stockton
John Stockton [plays on] Utah Jazz
Chicago Bulls [plays in] Eastern Conference
```

**Common edge types:**
- `plays_on`
- `plays_against`
- `plays_with`
- `has_a`
- `is_a`
- `belongs_to`

***

### Example: NBA Player Graph

**Vertices:**

```sql
-- Players
identifier       | type    | properties
'Michael Jordan' | 'player' | {height: '6-6', position: 'SG'}
'John Stockton'  | 'player' | {height: '6-1', position: 'PG'}

-- Teams
'Chicago Bulls' | 'team' | {city: 'Chicago', conference: 'Eastern'}
'Utah Jazz'     | 'team' | {city: 'Salt Lake City', conference: 'Western'}
```

**Edges:**

```sql
subject_identifier | subject_type | object_identifier | object_type | edge_type       | properties
'Michael Jordan'   | 'player'     | 'Chicago Bulls'   | 'team'      | 'plays_on'      | {seasons: '13', years: '1984-1998'}
'John Stockton'    | 'player'     | 'Utah Jazz'       | 'team'      | 'plays_on'      | {seasons: '19', years: '1984-2003'}
'Michael Jordan'   | 'player'     | 'John Stockton'   | 'player'    | 'plays_against' | {finals_meetings: '2', years: '1997,1998'}
```

**Visual representation:**

```
[Michael Jordan] ─(plays_on)→ [Chicago Bulls]
       ↓
   (plays_against)
       ↓
[John Stockton] ─(plays_on)→ [Utah Jazz]
```

***

### Why Graph Modeling is Powerful

**Traditional approach (relational):**

```sql
-- Find teammates of Michael Jordan's opponents
SELECT DISTINCT p3.name
FROM players p1
JOIN player_opponents po ON p1.id = po.player_id
JOIN players p2 ON po.opponent_id = p2.id
JOIN player_teams pt ON p2.id = pt.player_id
JOIN player_teams pt2 ON pt.team_id = pt2.team_id
JOIN players p3 ON pt2.player_id = p3.id
WHERE p1.name = 'Michael Jordan'
  AND p3.id != p2.id;
```

**Messy joins, multiple scans, expensive shuffles**.[1]

***

**Graph approach:**

```sql
-- Find teammates of Michael Jordan's opponents
SELECT DISTINCT e3.object_identifier
FROM edges e1  -- Michael Jordan plays against
JOIN edges e2 ON e1.object_identifier = e2.subject_identifier  -- Those players play on
JOIN edges e3 ON e2.object_identifier = e3.object_identifier   -- Teammates on same team
WHERE e1.subject_identifier = 'Michael Jordan'
  AND e1.edge_type = 'plays_against'
  AND e2.edge_type = 'plays_on'
  AND e3.edge_type = 'plays_on'
  AND e3.subject_identifier != e2.subject_identifier;
```

**Much simpler, focused on relationships**.[1]

***

### Netflix Infrastructure Graph Example

**70-80 different vertex types in one graph**:[1]

```sql
-- Vertices
identifier        | type          | properties
'recommendation'  | 'application' | {language: 'Java', team: 'Algo'}
'prod_db_123'     | 'database'    | {db_type: 'PostgreSQL', size_gb: 500}
'server_456'      | 'server'      | {region: 'us-west', os: 'Linux'}
'rec_repo'        | 'codebase'    | {language: 'Java', loc: 50000}

-- Edges
subject_identifier | subject_type  | object_identifier | object_type | edge_type    | properties
'recommendation'   | 'application' | 'prod_db_123'     | 'database'  | 'depends_on' | {queries_per_sec: 1000}
'recommendation'   | 'application' | 'server_456'      | 'server'    | 'runs_on'    | {instances: 20}
'rec_repo'         | 'codebase'    | 'recommendation'  | 'application' | 'implements' | {version: 'v2.3'}
```

**Power:** Query across all infrastructure to find cascading impacts of failures.[1]

---

### Facebook Family of Apps Graph

**Connecting user identities across apps**:[1]

```sql
-- Vertices
identifier        | type             | properties
'fb_user_12345'   | 'facebook_user'  | {name: 'Alice', joined: '2010-01-01'}
'ig_user_67890'   | 'instagram_user' | {username: '@alice', followers: 5000}
'wa_user_11111'   | 'whatsapp_user'  | {phone: '+1234567890'}

-- Edges (identity linking)
subject_identifier | subject_type      | object_identifier | object_type       | edge_type  | properties
'fb_user_12345'    | 'facebook_user'   | 'ig_user_67890'   | 'instagram_user'  | 'same_as'  | {confidence: 0.99}
'fb_user_12345'    | 'facebook_user'   | 'wa_user_11111'   | 'whatsapp_user'   | 'same_as'  | {confidence: 0.95}
```

**Use case:** Cross-app analytics while respecting per-app privacy boundaries.[1]

***

## Practical Applications Summary

### When to Use Each Pattern

| Pattern | Use When | Example |
|---------|----------|---------|
| **Additive dimensions** | User can only have one value at a time | Age, country, current_plan |
| **Non-additive dimensions** | User can have multiple values simultaneously | Devices, car models, apps |
| **Enums** | <50 possible values, need data quality | Scoring class, notification channel, status |
| **Little Book pattern** | 30+ disparate sources → shared schema | Unit economics, infrastructure inventory |
| **Flexible schemas (maps)** | Variable columns across sources | Product attributes, metadata, low-use fields |
| **Graph modeling** | Focus on relationships > entities | Infrastructure dependencies, social networks |

***

## Key Takeaways

**Additivity fundamentally changes aggregation strategy**:[1]
- Additive: Pre-aggregate with SUM
- Non-additive: Must use COUNT DISTINCT on raw data
- Test: Can entity have two values simultaneously?

**Enumerations provide free data quality and documentation**:[1]
- Limit: <50 values
- Benefits: Pipeline failures on invalid values, static metadata, sub-partitioning

**Little Book of Pipelines scales to massive data integration**:[1]
- Enum-driven pipeline orchestration
- Shared schema with custom DQ checks
- Self-documenting partitions

**Flexible schemas trade compression for extensibility**:[1]
- Maps allow unlimited columns without ALTER TABLE
- Worst compression of any data type
- Best for low-query-frequency columns

**Graph modeling shifts focus from entities to relationships**:[1]
- Universal schema: 3 columns for vertices, 7 for edges
- Edge types are verbs connecting nodes
- Simplifies relationship queries that are messy in relational databases

**Remember:** Choose the right tool for the problem. Don't force every dataset into the same modeling paradigm.[1]

[1](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/45417937/a977f407-54ff-48f1-a522-65b889a83f42/paste.txt)
