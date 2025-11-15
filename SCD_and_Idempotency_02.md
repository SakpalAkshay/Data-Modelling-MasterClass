# Data Modeling - Slowly Changing Dimensions and Idempotency

A comprehensive guide to building idempotent pipelines and properly modeling dimensions that change over time.

---

## Table of Contents

- What is Idempotency?
- Why Idempotency Matters
- Common Idempotency Mistakes
- Slowly Changing Dimensions (SCD)
- SCD Types (0, 1, 2, 3)
- Loading Strategies
- Best Practices

***

## What is Idempotency?

**Definition:** An idempotent pipeline produces the same results regardless of when it's run, how many times it's run, or the hour/day it's executed.[1]

**Mathematical concept:** An element unchanged when operated on by itself.[1]

**Data engineering translation:** If all inputs are available, the pipeline should produce identical output whether run today, next year, or in 10 years.[1]

### The Three Pillars

Idempotent pipelines produce the same results regardless of:[1]

1. **How many times** the pipeline runs
2. **When** (what day/time) the pipeline runs
3. **The hour** the pipeline executes

***

## Why Idempotency Matters

### Hard to Troubleshoot

Non-idempotent pipelines **don't fail**—they produce **non-reproducible data**. The issue surfaces when analysts ask:[1]
- "Why do these numbers not match?"
- "Why does this table show different values than that table?"

**Example scenario:**
- Run pipeline today → Get result A
- Wait one week, backfill same date → Get result B (different from A)
- **No code changed**, but data is inconsistent

### Transitive Property Problem

If your pipeline isn't idempotent and downstream engineers depend on your data, **their pipelines become non-idempotent too**.[1]

```
Non-idempotent Pipeline A 
    ↓
Depends on: Pipeline B (now non-idempotent)
    ↓
Depends on: Pipeline C (now non-idempotent)
```

Inconsistencies bleed throughout your entire data warehouse.[1]

### Trust Erosion

Analytics teams lose trust in datasets when numbers constantly change without explanation.[1]

### Cumulative Pipelines Carry Bugs Forward

If a cumulative table depends on non-idempotent data, it **carries bugs forward every single day**. You must start the cumulation over from scratch to fix it.[1]

***

## Common Idempotency Mistakes

### 1. INSERT INTO Without TRUNCATE

**Problem:** Running the pipeline multiple times duplicates data.[1]

**Bad example:**
```sql
-- First run: 100 rows
INSERT INTO user_events 
SELECT * FROM daily_events WHERE date = '2025-01-15';

-- Second run: Now 200 rows (duplicated!)
INSERT INTO user_events 
SELECT * FROM daily_events WHERE date = '2025-01-15';
```

**Solutions:**

**Option A: MERGE**
```sql
MERGE INTO user_events target
USING (SELECT * FROM daily_events WHERE date = '2025-01-15') source
ON target.event_id = source.event_id
WHEN NOT MATCHED THEN INSERT *;

-- Running multiple times produces same result
```

**Option B: INSERT OVERWRITE**
```sql
INSERT OVERWRITE TABLE user_events
PARTITION (date = '2025-01-15')
SELECT * FROM daily_events WHERE date = '2025-01-15';

-- Each run overwrites the partition with same data
```

**Rule:** Never use `INSERT INTO` unless paired with `TRUNCATE`. Always prefer `MERGE` or `INSERT OVERWRITE`.[1]

***

### 2. Start Date Without End Date

**Problem:** Unbounded date filters produce different results depending on when pipeline runs.[1]

**Bad example:**
```sql
-- Terrible: unbounded window
SELECT * FROM events
WHERE date >= '2025-01-10';

-- Run on Jan 15: Returns 5 days
-- Run on Jan 20: Returns 10 days
-- Not idempotent!
```

**Good example:**
```sql
-- Bounded window
SELECT * FROM events
WHERE date >= '2025-01-10'
  AND date < '2025-01-17';

-- Always returns 7 days, regardless of run date
```

**Additional benefit:** Prevents out-of-memory exceptions during backfills by limiting data scanned.[1]

***

### 3. Incomplete Partition Sensors

**Problem:** Pipeline runs before all upstream inputs are ready.[1]

**Scenario:**
- Pipeline depends on tables A, B, and C
- Only checks if table A is ready
- Runs before B and C finish
- Produces incomplete results

**In production:**
```
Day 1: Runs at 2 AM, only table A ready → Incomplete data
Day 2: Runs at 2 AM, only table A ready → Incomplete data
```

**During backfill:**
```
All tables (A, B, C) already complete → Full data
```

**Result:** Production and backfill produce different data.[1]

**Solution:** Check for **full set** of all input partitions before running.

```python
# Airflow example
wait_for_all_inputs = ExternalTaskSensor(
    task_id='wait_for_inputs',
    external_dag_id=['table_a', 'table_b', 'table_c'],
    # Wait for all three before proceeding
)
```

***

### 4. Parallel Processing of Sequential Dependencies (Depends on Past)

**Problem:** Cumulative pipelines require yesterday's output as input. Running days in parallel breaks this.[1]

**Cumulative pattern reminder:**
```
Yesterday's data + Today's data → Today's output (becomes tomorrow's "yesterday")
```

**What happens without sequential processing:**
```
Backfill starts all days in parallel:
- Day 5 starts before Day 4 completes
- Day 5 can't find "yesterday" data (Day 4)
- Day 5 starts cumulation from scratch
- History lost!
```

**Solution: Enable sequential processing**

```python
# Airflow example
task = PythonOperator(
    task_id='cumulative_build',
    depends_on_past=True,  # Forces sequential execution
    wait_for_downstream=True
)
```

**Trade-off:** Cumulative pipelines cannot parallelize backfills—must run one day at a time.[1]

***

### 5. Relying on "Latest" Partitions

**Problem:** Pipeline pulls most recent available data instead of specific date.[1]

**Real-world example from Facebook:**

Zach built a fake accounts state transition tracker that depended on `dim_all_users`.[1]

**The setup:**
```sql
-- dim_all_fake_accounts relied on LATEST dim_all_users
WITH latest_users AS (
    SELECT * FROM dim_all_users
    ORDER BY partition_date DESC
    LIMIT 1  -- Takes most recent available
)
```

**What happened:**
- Some days: `dim_all_users` ready for today → Uses today's data
- Other days: `dim_all_users` delayed → Uses yesterday's data
- **Result:** Fake account transitions sometimes off by one day

**Consequences:**
- Numbers didn't match between tables
- Took over a month to debug
- Caused data quality issues
- Contributed to Zach quitting Facebook[1]

**The ONE exception:**
You can rely on latest partitions **only** when:
1. Backfilling (not production)
2. Using properly modeled SCD Type 2 tables[1]

**Correct approach:**
```sql
-- Always specify exact date
WITH todays_users AS (
    SELECT * FROM dim_all_users
    WHERE partition_date = '2025-01-15'  -- Explicit date
)
```

***

## More Idempotency Implications

### Backfill vs Production Discrepancies

Non-idempotent pipelines create different data in production vs backfill:[1]
- Production run (Jan 15) → Result A
- Backfill same date (Jan 22) → Result B

### Unit Tests Can't Catch These Bugs

Unit tests pass even with non-idempotent pipelines because tests don't replicate temporal production behavior.[1]

**Why tests fail to catch:**
- Tests run in controlled environment with mocked data
- Production has real timing dependencies
- Tests don't simulate "run today vs run next week" scenarios

### Silent Failures

Pipelines don't throw errors—they just produce wrong data. No alerts, no logs, just incorrect results.[1]

***

## Slowly Changing Dimensions (SCD)

### What is an SCD?

A **slowly changing dimension** is an attribute that changes over time.[1]

**Examples of SCDs:**
- Age (changes yearly)
- Favorite food ("lasagna" in 2000s → "curry" in 2020s)[1]
- Phone type (Android → iPhone)
- Country (Dominican Republic → USA)
- Job title (Junior Analyst → Senior Analyst → Manager)
- Device preference (catflix → dogflix)[1]

**Fixed dimensions (NOT slowly changing):**
- Birthday
- Eye color (usually)
- Birth country
- Account creation date

***

### Slowly vs Rapidly Changing

**Slowly changing:** Changes infrequently (months/years)
- Ideal for SCD modeling
- High compression benefits

**Rapidly changing:** Changes frequently (days/hours)
- Consider daily snapshots instead
- SCD modeling loses efficiency

**Example:**
- **Age:** Changes once per year → Great for SCD
- **Heart rate:** Changes minute-to-minute → Not suitable for SCD (treat as metric)

**Rule of thumb:** The slower the change frequency, the better SCD modeling performs.[1]

***

## Three Modeling Approaches for Changing Dimensions

### Latest Snapshot

**Structure:** One row per entity with current value only[1]

```sql
user_id | favorite_food | last_updated
501     | curry         | 2025-01-15
```

**NEVER DO THIS** for analytical pipelines.[1]

**Why it's bad:**
When backfilling historical data, it pulls the latest value into old records.

**Example problem:**
```
Zach's Facebook post from 2012 (age 18)
Backfill in 2025 (age 29)
Result: "Zach made this terrible post at age 29" 
Reality: "He was 18, just a teenager"
```

**When it's acceptable:** OLTP systems where only current state matters (never for analytics).[1]

***

### Daily Partition Snapshots

**Structure:** One row per entity per day[1]

```sql
user_id | favorite_food | date
501     | lasagna      | 2008-01-15
501     | lasagna      | 2008-01-16
501     | lasagna      | 2008-01-17
...
501     | curry        | 2020-05-15
501     | curry        | 2020-05-16
```

**Pros:**
- Fully idempotent[1]
- Simple to understand and query
- No complex date range logic
- Backfills work perfectly

**Cons:**
- Data duplication (365 rows per year per entity)
- Higher storage costs

**When to use:**
- Max Beauchemin's (Apache Airflow creator) preferred approach[1]
- Storage is cheap, data quality bugs are expensive[1]
- Team prioritizes simplicity over compression

***

### Slowly Changing Dimension (Compressed)

**Structure:** One row per value change with start/end dates[1]

```sql
user_id | favorite_food | start_date  | end_date
501     | lasagna      | 2000-01-01 | 2010-01-01
501     | curry        | 2010-01-01 | 9999-12-31  -- Current value
```

**Compression example:**
- Daily snapshots: 365 rows for one unchanging year
- SCD: 1 row for entire year
- **365x compression**

**Pros:**
- Massive storage savings
- Idempotent when properly implemented[1]
- Complete historical accuracy

**Cons:**
- More complex queries (date range filtering)
- Requires careful modeling

---

## Max Beauchemin vs Zach Debate

**Max's position:** SCDs are inherently non-idempotent; just use daily snapshots.[1]

**Zach's position:** Properly modeled SCD Type 2 is idempotent AND provides compression benefits.[1]

**Key insight:** Both approaches can work. Choose based on:
- Team SQL skill level
- Storage constraints
- Dimension change frequency
- Complexity tolerance

***

## SCD Types Explained

### SCD Type 0: Fixed Dimensions

**Definition:** Dimension never changes once set.[1]

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    birthday DATE,           -- Never changes
    birth_country TEXT,      -- Never changes
    account_created_at TIMESTAMP  -- Never changes
);
```

**Structure:**
- No temporal columns needed
- No start/end dates
- Just identifier + dimension value

**When to use:** You're **certain** the dimension will never change.[1]

**Idempotent:** Yes (value is immutable)

---

### SCD Type 1: Latest Value Only

**Definition:** Overwrite with new value, lose history.[1]

```sql
-- User changes phone preference
UPDATE users
SET phone_type = 'iPhone'
WHERE user_id = 501;

-- Previous value ("Android") is lost forever
```

**Result:**
```sql
user_id | phone_type | updated_at
501     | iPhone     | 2025-01-15
-- No record that user ever had Android
```

**When it's acceptable:** OLTP systems (online apps) that only care about current state.[1]

**For analytics: NEVER USE THIS**.[1]

**Why it's terrible:**
- Backfills pull latest value into historical records
- Not idempotent
- Loses valuable historical context
- Can get you "canceled" if you backfill social media posts with wrong age dimension[1]

---

### SCD Type 2: Full History with Date Ranges (GOLD STANDARD)

**Definition:** Track every value change with start and end dates.[1]

**Structure:**
```sql
CREATE TABLE user_dimensions (
    user_id INT,
    favorite_food TEXT,
    start_date DATE,
    end_date DATE,
    is_current BOOLEAN,
    PRIMARY KEY (user_id, start_date)
);
```

**Example data:**
```sql
user_id | favorite_food | start_date  | end_date    | is_current
501     | lasagna      | 2000-01-01 | 2010-01-01 | false
501     | curry        | 2010-01-01 | 9999-12-31 | true
```

**How to query historical value:**
```sql
-- What was Zach's favorite food on 2008-05-15?
SELECT favorite_food
FROM user_dimensions
WHERE user_id = 501
  AND '2008-05-15' BETWEEN start_date AND end_date;

-- Result: lasagna
```

**How to query current value:**
```sql
-- Option 1: Use is_current flag
SELECT favorite_food
FROM user_dimensions
WHERE user_id = 501 AND is_current = TRUE;

-- Option 2: Filter by far future date
SELECT favorite_food
FROM user_dimensions
WHERE user_id = 501
  AND end_date = '9999-12-31';
```

**End date conventions:**

Two common approaches:[1]

1. **Far future date:** `9999-12-31` (Airbnb's approach)[1]
2. **NULL:** Current records have `NULL` end_date

**Airbnb's reasoning:** Explicit date is clearer than NULL in SQL logic.

**Additional column: `is_current`**

Boolean flag for easy current-value filtering:[1]
```sql
ALTER TABLE user_dimensions ADD COLUMN is_current BOOLEAN;

-- Update on inserts
UPDATE user_dimensions SET is_current = FALSE WHERE user_id = 501;
INSERT INTO user_dimensions VALUES (501, 'sushi', '2025-01-15', '9999-12-31', TRUE);
```

***

### SCD Type 3: Original and Current Only

**Definition:** Store only two values—original and current.[1]

**Structure:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    original_phone TEXT,
    current_phone TEXT,
    original_phone_date DATE,
    current_phone_date DATE
);
```

**Example:**
```sql
user_id | original_phone | current_phone | original_phone_date | current_phone_date
501     | Android       | iPhone        | 2018-01-01         | 2023-05-15
```

**Problem: Changes more than once**
```
2018: Android (original)
2020: Windows Phone → Lost!
2023: iPhone (current)
```

Windows Phone period completely disappears.

**Pros:**
- Still one row per entity
- No date range filtering needed

**Cons:**
- **Not idempotent**—loses history after second change[1]
- Incomplete historical picture
- Fails backfill tests

**Verdict:** "Gross middle ground". Don't use.[1]

***

## SCD Type Summary

| Type | Description | Idempotent? | Use Case |
|------|-------------|-------------|----------|
| **Type 0** | Fixed, never changes | ✅ Yes (if truly fixed) | Birthday, creation date |
| **Type 1** | Latest value only | ❌ No | OLTP only, **never analytics** |
| **Type 2** | Full history (date ranges) | ✅ Yes | **Gold standard for analytics** |
| **Type 3** | Original + current only | ❌ No | Avoid |

**Zach's recommendation:** Only use Type 0 and Type 2.[1]

---

## Loading SCD Type 2 Tables

### Two Approaches

**1. Load Entire History**
Generate complete SCD from all daily data every run.[1]

```sql
-- Process all history every day
WITH all_daily_snapshots AS (
    SELECT user_id, favorite_food, date
    FROM user_daily_history
    WHERE date >= '2015-01-01'  -- All time
)
-- Collapse into SCD with LAG/LEAD logic
SELECT 
    user_id,
    favorite_food,
    date as start_date,
    LEAD(date, 1, '9999-12-31') OVER (
        PARTITION BY user_id ORDER BY date
    ) as end_date
FROM (
    SELECT DISTINCT user_id, favorite_food, date
    FROM all_daily_snapshots
    WHERE favorite_food != LAG(favorite_food) OVER (
        PARTITION BY user_id ORDER BY date
    )
);
```

**Pros:**
- Simpler logic
- No dependency on yesterday
- Can parallelize backfills

**Cons:**
- Processes all history daily
- More compute expensive
- Slower for large datasets

---

**2. Load Incrementally (Cumulative)**
Process only new day, update existing SCD.[1]

```sql
-- Yesterday's SCD + today's changes
WITH yesterday_scd AS (
    SELECT * FROM user_scd WHERE end_date = '9999-12-31'
),
today_snapshot AS (
    SELECT user_id, favorite_food FROM user_daily WHERE date = '2025-01-15'
)
SELECT 
    COALESCE(y.user_id, t.user_id) as user_id,
    COALESCE(t.favorite_food, y.favorite_food) as favorite_food,
    CASE 
        WHEN t.favorite_food != y.favorite_food THEN
            -- Close old record, start new one
            '2025-01-15'
        ELSE y.start_date
    END as start_date,
    '9999-12-31' as end_date
FROM yesterday_scd y
FULL OUTER JOIN today_snapshot t ON y.user_id = t.user_id;
```

**Pros:**
- Only processes one day
- Efficient for production
- Scales to large datasets

**Cons:**
- More complex logic
- Sequential backfills only
- Dependency on yesterday's output

***

### Zach's Airbnb Experience

Built "unit economics" SCD tracking booking payment changes (payments, refunds, adjustments).[1]

**Setup:**
- Always used "load entire history" approach
- Processed all transactions back to 2016 daily
- Felt inefficient but worked reliably

**Lesson learned:**
"Don't get caught up making every pipeline a Ferrari".[1]

**Key question:** Is optimizing this the most valuable use of your time?

**Trade-off:**
- Spend weeks building incremental loader
- OR ship other business-critical features

**Decision:** Shipped pricing, availability, and other high-impact projects instead of optimizing this one pipeline.[1]

**Takeaway:** Marginal efficiency gains often aren't worth opportunity cost. Focus on business value.

***

## Practical Example: User Age SCD

### Daily Snapshots (Max's approach)

```sql
-- Every day
user_id | age | date
501     | 18  | 2012-01-30
501     | 18  | 2012-01-31
501     | 18  | 2012-02-01
...
501     | 19  | 2013-01-30  -- Birthday!
501     | 19  | 2013-01-31
...
501     | 29  | 2025-01-14
501     | 29  | 2025-01-15

-- 4,748 rows (13 years × 365 days)
```

***

### SCD Type 2 (Zach's approach)

```sql
user_id | age | start_date  | end_date    | is_current
501     | 18  | 2012-01-30 | 2013-01-30 | false
501     | 19  | 2013-01-30 | 2014-01-30 | false
501     | 20  | 2014-01-30 | 2015-01-30 | false
501     | 21  | 2015-01-30 | 2016-01-30 | false
501     | 22  | 2016-01-30 | 2017-01-30 | false
501     | 23  | 2017-01-30 | 2018-01-30 | false
501     | 24  | 2018-01-30 | 2019-01-30 | false
501     | 25  | 2019-01-30 | 2020-01-30 | false
501     | 26  | 2020-01-30 | 2021-01-30 | false
501     | 27  | 2021-01-30 | 2022-01-30 | false
501     | 28  | 2022-01-30 | 2023-01-30 | false
501     | 29  | 2023-01-30 | 9999-12-31 | true

-- 12 rows (one per year)
-- 395x compression!
```

**Query comparison:**

```sql
-- Daily: What was age on 2015-06-10?
SELECT age FROM user_daily WHERE user_id = 501 AND date = '2015-06-10';

-- SCD: What was age on 2015-06-10?
SELECT age FROM user_scd 
WHERE user_id = 501 
  AND '2015-06-10' BETWEEN start_date AND end_date;
```

Both queries return `21`, but SCD uses 395x less storage.

***

## Best Practices Summary

### Idempotency Checklist

✅ Use `MERGE` or `INSERT OVERWRITE`, never plain `INSERT INTO`[1]
✅ Always include end date with start date (bounded windows)[1]
✅ Check for complete set of input partitions[1]
✅ Enable sequential processing for cumulative pipelines[1]
✅ Never rely on "latest" partitions in production[1]

### SCD Checklist

✅ Use SCD Type 0 for fixed dimensions[1]
✅ Use SCD Type 2 for analytics-quality slowly changing dimensions[1]
✅ Never use SCD Type 1 for analytics[1]
✅ Avoid SCD Type 3[1]
✅ Consider daily snapshots if team unfamiliar with SCD complexity[1]

### Career Wisdom

✅ Prioritize business value over engineering perfection[1]
✅ "Good enough" pipelines that ship are better than perfect pipelines that don't[1]
✅ Opportunity cost matters—what else could you build?[1]

***

## Key Takeaways

**Idempotency is critical**:[1]
- Production and backfill must produce same results
- Non-idempotent pipelines cause silent failures
- Bugs cascade through downstream tables
- Data quality issues erode trust

**SCD Type 2 is the gold standard** for dimensional modeling in analytics:[1]
- Maintains complete history
- Fully idempotent when done right
- Massive compression for slowly changing data
- Industry standard (Airbnb, Facebook, Netflix)

**Simplicity vs efficiency trade-off:**
- Daily snapshots: Simple, reliable, higher storage
- SCD Type 2: Complex, efficient, lower storage
- Both can be idempotent—choose based on context

**Remember:** Storage is cheap. Data quality bugs and engineer time are expensive.[1]

[1](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/45417937/255b3cbd-9eda-448f-a429-1e80ec0cd734/paste.txt)
