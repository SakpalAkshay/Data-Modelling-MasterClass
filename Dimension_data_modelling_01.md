# Data Modeling Fundamentals - Complex Data Types and Cumulation

A comprehensive guide to advanced dimensional data modeling techniques, complex data types, and cumulative table design.

***

## Dimensions: The Building Blocks

**Dimensions** are attributes of an entity that provide context and descriptive information. Think of dimensions like coordinates that define a space—they help locate and describe data entities.[1]

### Types of Dimensions

**Identifier Dimensions**
Uniquely identify an entity.[1]

- User ID
- Social Security Number  
- Device ID
- Product SKU

**Example:**
```sql
user_id: 12345
device_id: "ABC-789-XYZ"
```

**Attribute Dimensions**
Provide additional context but aren't critical for identification. They come in two flavors:[1]

**1. Slowly Changing Dimensions (SCD)**
Attributes that change over time.[1]

**Example:**
- Your favorite food (was lasagna as a child, now spicy curry)
- Job title (Junior Analyst → Senior Analyst → Manager)
- User address (moved from New York → California)

```sql
-- User's favorite food changed over time
user_id | favorite_food | valid_from
501     | lasagna      | 2010-01-01
501     | spicy_curry  | 2020-05-15
```

**2. Fixed Dimensions**
Attributes that never change once set.[1]

**Example:**
- Birthday (born on January 15, 1990)
- Phone manufacturer (iPhone made by Apple)
- Account creation date

```sql
user_id | birthday    | account_created
501     | 1990-01-15 | 2018-03-20
```

***

## Know Your Data Customer

Data modeling requires **empathy**—understanding who will consume your data shapes how you should model it.[1]

### Customer Types

**Data Analysts & Scientists**
- **Need:** Easy-to-query flat tables with primitive types (integers, decimals, strings)
- **Want:** Simple aggregations (SUM, AVG, COUNT) without complex transformations
- **Example:**
```sql
-- Good for analysts
SELECT country, AVG(purchase_amount)
FROM purchases
GROUP BY country;
```

**Other Data Engineers**
- **Need:** Master data that's complete and normalized
- **Can handle:** Complex data types (structs, arrays, maps)
- **Use case:** Join your tables to create downstream datasets

**Example:**
```sql
-- Engineers can work with arrays
user_id | activity_dates ARRAY
501     | [2025-01-01, 2025-01-02, 2025-01-05]
```

**Machine Learning Models**
- **Need:** Identifier + flattened feature columns (mostly decimals/integers)
- **Prefer:** Wide format with consistent data types
- **Less concerned:** About column naming conventions

**Example:**
```sql
user_id | feature_1 | feature_2 | feature_3 | label
501     | 0.85     | 120       | 0.32      | 1
```

**Customers/End Users**
- **Should receive:** Charts, dashboards, visualizations—not raw tables
- **Exception:** Analytical products where users are data-savvy

***

## The Data Modeling Continuum

Three data modeling paradigms exist on a spectrum:[1]

### OLTP (Online Transaction Processing)

**Purpose:** Optimize for single-row lookups and writes[1]
**Used by:** Software engineers for production systems
**Characteristics:**
- Third normal form (3NF)
- Minimal data duplication
- Many foreign keys and constraints
- Fast single-entity queries

**Example:**
```sql
-- Normalized tables with foreign keys
users:          user_id | name | email
orders:         order_id | user_id | total
order_items:    item_id | order_id | product_id | quantity
```

***

### Master Data (Middle Ground)

**Purpose:** Complete, normalized dimensional data for data engineers[1]
**Characteristics:**
- De-duplicated entities
- Complete definitions
- May use complex data types
- Bridges transactional and analytical systems

**Example:**
```sql
-- One complete row per user with all dimensions
user_master:
user_id | name  | email           | signup_date | last_active | activity_history ARRAY
501     | Alice | alice@email.com | 2024-01-15  | 2025-01-10  | [2025-01-08, 2025-01-09, 2025-01-10]
```

**Real-world case:** At Facebook, the `dim_all_users` table had 10,000 downstream pipelines depending on it as the source of truth for all user data.[1]

---

### OLAP (Online Analytical Processing)

**Purpose:** Optimize for aggregations across populations[1]
**Used by:** Analysts for slice-and-dice analysis
**Characteristics:**
- Data duplication is acceptable
- Optimized for GROUP BY queries
- Multiple rows per entity common
- Fast aggregations without joins

**Example:**
```sql
-- Flattened for analysis (duplicated user data)
user_events_flattened:
user_id | user_name | user_country | event_type | event_date
501     | Alice     | USA          | login      | 2025-01-10
501     | Alice     | USA          | purchase   | 2025-01-10
502     | Bob       | Canada       | login      | 2025-01-10

-- Fast aggregation
SELECT user_country, COUNT(*) as event_count
FROM user_events_flattened
WHERE event_type = 'purchase'
GROUP BY user_country;
```

***

### Metrics (Aggregated)

**Purpose:** Single number summaries[1]

**Example:**
40 transactional tables → 1 master data table → 1 OLAP cube → 1 metric (average listing price across all Airbnb)

---

## Mismatching Needs = Pain

**Modeling OLTP like OLAP:**
- **Symptom:** Slow online app performance
- **Cause:** Pulling unnecessary duplicated data

**Modeling OLAP like OLTP:**
- **Symptom:** Expensive joins for every analytical query
- **Cause:** Too many normalized tables requiring shuffle operations

**Solution:** Use master data as the middle layer.[1]

***

## Cumulative Table Design

**Purpose:** Build master data that holds complete history by accumulating changes over time.[1]

### How It Works

Cumulative tables use a **full outer join** pattern between today's data and yesterday's cumulated history.[1]

**Pattern:**
```
Yesterday's cumulated data (all history)
    +
Today's new data
    ↓
FULL OUTER JOIN
    ↓
Today's cumulated data (becomes tomorrow's "yesterday")
```

**Example:**

```sql
-- Day 1: Initial load (yesterday is NULL)
user_id | last_active | days_since_active
501     | 2025-01-01  | 0

-- Day 2: User 501 inactive, user 502 appears
-- Full outer join between yesterday and today
WITH yesterday AS (
    SELECT * FROM user_cumulative WHERE date = '2025-01-01'
),
today AS (
    SELECT user_id, activity_date FROM user_activity WHERE date = '2025-01-02'
)
SELECT 
    COALESCE(y.user_id, t.user_id) as user_id,
    COALESCE(t.activity_date, y.last_active) as last_active,
    CASE 
        WHEN t.user_id IS NULL THEN y.days_since_active + 1
        ELSE 0
    END as days_since_active
FROM yesterday y
FULL OUTER JOIN today t ON y.user_id = t.user_id;

-- Result:
user_id | last_active | days_since_active
501     | 2025-01-01  | 1  -- Inactive today, increment counter
502     | 2025-01-02  | 0  -- New user appeared
```

***

### State Transition Tracking

Cumulative tables excel at tracking user state changes:[1]

**Growth Accounting Pattern:**

| Yesterday Active | Today Active | State        |
|------------------|--------------|--------------|
| No               | Yes          | New          |
| Yes              | Yes          | Retained     |
| Yes              | No           | Churned      |
| No (was active)  | Yes          | Resurrected  |

**Example:**
```sql
-- Track user lifecycle states
SELECT 
    user_id,
    CASE 
        WHEN yesterday_active = FALSE AND today_active = TRUE 
            AND ever_active = FALSE THEN 'New'
        WHEN yesterday_active = FALSE AND today_active = TRUE 
            AND ever_active = TRUE THEN 'Resurrected'
        WHEN yesterday_active = TRUE AND today_active = FALSE THEN 'Churned'
        WHEN yesterday_active = TRUE AND today_active = TRUE THEN 'Retained'
    END as user_state
FROM user_cumulative;
```

***

### Pros of Cumulative Design

**Historical Analysis Without GROUP BY**[1]
- Query just the latest partition, not 90 days of data
- All history stored in arrays on single row
- Massively scalable for historical queries

**Example:**
```sql
-- Instead of scanning 90 days:
SELECT user_id FROM daily_activity 
WHERE date BETWEEN '2024-10-01' AND '2024-12-31'
GROUP BY user_id;

-- Just select from latest:
SELECT user_id, activity_last_90_days 
FROM user_cumulative 
WHERE date = '2024-12-31'
AND CARDINALITY(activity_last_90_days) > 0;
```

**State Transitions Made Easy**[1]
Churned/resurrected analysis becomes simple row comparisons.

***

### Cons of Cumulative Design

**Sequential Backfills Only**[1]
- Must backfill day-by-day (can't parallelize)
- Each day depends on the previous day's output
- Daily data can backfill all dates in parallel

**PII Management Complexity**[1]
- Must actively filter deleted/hibernated users
- Need separate deletion tracking table
- Data grows indefinitely without pruning

**Example pruning logic:**
```sql
-- Remove users inactive for 180+ days
WHERE days_since_active < 180
OR last_active > CURRENT_DATE - INTERVAL '180 days'
```

***

## Compactness vs Usability Tradeoff

### The Spectrum

**Most Usable**[1]
- Identifier + flat primitive types (strings, integers, decimals)
- Easy WHERE and GROUP BY
- **For:** Analysts, data scientists

**Example:**
```sql
user_id | name  | country | age | revenue
501     | Alice | USA     | 28  | 150.00
```

**Middle Ground (Complex Types)**[1]
- Arrays, structs, maps
- More compact, slightly harder to query
- **For:** Data engineers (master data)

**Example:**
```sql
user_id | profile STRUCT<name, country, age> | purchases ARRAY<DECIMAL>
501     | {Alice, USA, 28}                   | [50.00, 75.00, 25.00]
```

**Most Compact**[1]
- Identifier + compressed binary blobs
- Requires decompression/decoding
- **For:** Production systems serving apps

**Example:**
```sql
user_id | data_blob BYTES
501     | 0x4A7F8E... (compressed calendar data)
```

***

## Complex Data Types

### Struct (Table Within a Table)

**Use when:** You have related fields with different data types[1]

**Example:**
```sql
-- User profile as struct
user_id | profile STRUCT<name STRING, age INT, premium BOOLEAN>
501     | {Alice, 28, true}

-- Query struct fields
SELECT user_id, profile.name, profile.age
FROM users
WHERE profile.premium = true;
```

***

### Array (Ordered List)

**Use when:** You have ordered, repeating values of the same type[1]

**Example:**
```sql
-- User's last 7 login dates
user_id | login_dates ARRAY<DATE>
501     | [2025-01-10, 2025-01-09, 2025-01-08, 2025-01-07]

-- Query array
SELECT user_id
FROM users
WHERE ARRAY_CONTAINS(login_dates, DATE '2025-01-10');
```

***

### Map (Key-Value Pairs)

**Use when:** You have variable/unknown number of keys with same value type[1]

**Example:**
```sql
-- Product attributes (keys vary by product)
product_id | attributes MAP<STRING, STRING>
101        | {color: red, size: large, material: cotton}
102        | {color: blue, weight: 2.5kg}

-- Query map
SELECT product_id, attributes['color']
FROM products
WHERE attributes['color'] = 'red';
```

**Limitation:** All values must be same data type (all strings, all integers, etc.)[1]

***

### Array of Struct (Powerful Combination)

**Use when:** You have multiple records with mixed data types[1]

**Example:**
```sql
-- User's purchase history
user_id | purchases ARRAY<STRUCT<date DATE, amount DECIMAL, product STRING>>
501     | [{2025-01-10, 49.99, shirt}, {2025-01-08, 29.99, hat}]

-- Query nested structure
SELECT 
    user_id,
    purchase.date,
    purchase.amount
FROM users
CROSS JOIN UNNEST(purchases) AS purchase
WHERE purchase.amount > 40.00;
```

***

## Temporal Cardinality Explosion

**Problem:** When dimensions have a time component, cardinality explodes.[1]

**Airbnb Example:**
- 6 million listings
- Each listing has ~365 nights in calendar
- Do you model as 6M rows or 2.2 billion rows?

### Three Modeling Approaches

**1. Compact (Array)**
```sql
-- One row per listing with array of nights
listing_id | nights ARRAY<STRUCT<date, price, available>>
12345      | [{2025-01-15, 150.00, true}, {2025-01-16, 150.00, false}, ...]
```

**Pros:** Compact storage, preserves sorting
**Cons:** Harder to query individual nights

**2. Exploded (Denormalized)**
```sql
-- One row per listing-night
listing_id | night_date  | price  | available
12345      | 2025-01-15 | 150.00 | true
12345      | 2025-01-16 | 150.00 | false
...
```

**Pros:** Easy to query and filter
**Cons:** 2.2 billion rows, vulnerable to shuffle issues

**3. Hybrid (Both)**
Master data uses arrays; OLAP cubes explode for analysis.[1]

***

## Run-Length Encoding & Sorting

**Run-length encoding** is a compression technique that replaces repeated values with a count.[1]

### How It Works

**Uncompressed:**
```
player_name: [AC Green, AC Green, AC Green, AC Green, AC Green]
season:      [1985,     1986,     1987,     1988,     1989]
```

**Run-length encoded:**
```
player_name: [AC Green, 5]  -- "AC Green" appears 5 times
season:      [1985, 1986, 1987, 1988, 1989]  -- Still stored individually
```

**Critical insight:** This only works when data is **sorted**. If data is shuffled, compression breaks.[1]

---

### The Shuffle Problem

**Scenario:** You have a sorted, compressed table of player statistics.[1]

**Original (sorted):**
```
player_name | season | points
AC Green    | 1985   | 850
AC Green    | 1986   | 920
AC Green    | 1987   | 880
...
```

**After JOIN (shuffled):**
```
player_name | season | points | team_name
Tim Duncan  | 1997   | 1200   | Spurs
AC Green    | 1985   | 850    | Lakers
Kobe Bryant | 1996   | 750    | Lakers
AC Green    | 1986   | 920    | Lakers  -- No longer consecutive!
...
```

**Result:** Run-length encoding compression is destroyed. File size increases dramatically.[1]

---

### Solution: Arrays Preserve Sorting

**Strategy:** Store temporal data in arrays, join at entity level, then explode.[1]

**Step 1: Model with arrays**
```sql
-- One row per player with seasons array
player_name | seasons ARRAY<STRUCT<year INT, points INT>>
AC Green    | [{1985, 850}, {1986, 920}, {1987, 880}, ...]
```

**Step 2: Join at player level (no shuffle of seasons)**
```sql
SELECT 
    p.player_name,
    p.seasons,
    t.team_name
FROM players p
JOIN teams t ON p.player_name = t.player_name;
```

**Step 3: Explode after join (sorting preserved)**
```sql
SELECT 
    player_name,
    season.year,
    season.points,
    team_name
FROM joined_data
CROSS JOIN UNNEST(seasons) AS season;
```

**Result:** All "AC Green" rows remain consecutive, preserving run-length encoding compression.[1]

***

## Real-World Case Study: Airbnb Pricing

Zach built a hybrid model combining multiple techniques:[1]

**1. Data Vault Layer (Raw Inputs)**
```sql
-- Preserve raw pricing rules
pricing_inputs:
input_id | source      | raw_rules JSON
1        | host_app    | {"base_price": 100, "weekend_premium": 1.2}
2        | discounts   | {"weekly": 0.9, "monthly": 0.8}
```

**2. Processing Layer**
Transform raw rules into actual prices per night.

**3. One Big Table (Output)**
```sql
-- Listing pricing with array of nights
listing_pricing:
listing_id | nights ARRAY<STRUCT<date, price, available, rules>>
12345      | [{2025-01-15, 150, true, {...}}, ...]
```

**Benefits:**
- Audit trail via Data Vault
- Fast aggregations via One Big Table
- Used complex types (struct, array) for compactness
- 95% reduction in data size compared to exploded format[1]

---

## Key Takeaways

**Data modeling is art, not science**[1]
- Mix and match techniques based on business needs
- Don't rigidly follow one "religion"
- Match modeling approach to your data customer

**Dimensional modeling hierarchy:**
1. **OLTP:** Normalized, single-row optimization
2. **Master Data:** Complete, de-duplicated entities (may use complex types)
3. **OLAP:** Denormalized, aggregation-optimized
4. **Metrics:** Fully aggregated summaries

**Cumulative table design:**
- Perfect for master data with complete history
- Full outer join pattern (yesterday + today)
- Trade-off: Historical power vs. sequential backfills

**Complex data types:**
- **Struct:** Different types, fixed keys
- **Array:** Same type, ordered values
- **Map:** Same type, variable keys
- **Array of Struct:** Most powerful combination

**Sorting and compression:**
- Run-length encoding requires sorted data
- Joins (shuffle) break sorting
- Arrays preserve sorting through joins
- Can achieve same file size with 6M rows or 2B rows if properly sorted

**Remember:** Understanding your data customer is the foundation of effective data modeling.[1]

[1](https://www.youtube.com/watch/WDwNow61JVE)
[2](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/45417937/f7c548a4-7a29-46ad-b7be-c2dd45d24b82/paste.txt)
