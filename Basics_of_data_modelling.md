# Data Modeling Fundamentals

A comprehensive guide to understanding data modeling for data engineers and analysts.

## Table of Contents

- Overview
- Three Levels of Data Modeling
- Physical Modeling Techniques
- Practical Application
- Key Takeaways

---

## Overview

Data modeling creates the structure of data that businesses use to make better decisions. It's the foundational skill that determines how effectively you can deliver data products.

### Why It Matters

The asset data professionals produce is structured data. Poor modeling leads to:
- Wasted time on low-value requirements
- Inefficient queries and slow performance
- Difficulty answering business questions
- Technical debt that compounds over time

---

## Three Levels of Data Modeling

### Conceptual Data Modeling

**Purpose:** High-level business requirements and data feasibility

**Key Questions:**
- What data does the business need?
- What data sources are available?
- What relationships exist between data entities?
- Is it ROI-positive to acquire this data?

**Example:**
A data scientist requests historical weather data for the past 10 years across 500 locations. Before building this table, evaluate:
- Is this data accessible? (Yes, via weather API)
- How long will it take? (2 weeks to build pipeline)
- How often will it be used? (One-time analysis)
- **Decision:** Push back and offer a sampled dataset instead

**Pro Tip:** Deep business understanding lets you identify and eliminate low-value requirements early, saving months of work.

***

### Logical Data Modeling

**Purpose:** Define facts and dimensions and their relationships

**Core Concepts:**

**Facts** = Events/Actions (verbs)
- User logs in
- Product purchased
- Message sent
- Ad clicked

Facts are immutable—once a user logs in at 6:03 PM, that timestamp cannot change.

**Dimensions** = Entities/Actors (nouns)
- Users
- Products
- Devices
- Locations
- Posts

Dimensions provide context to facts.

**Example:**

```
Fact: User clicked an ad

Related Dimensions:
- User (location: China, age: 28, account_type: premium)
- Device (type: mobile, OS: iOS 16)
- Browser (name: Safari, version: 16.2)
- Ad (campaign_id: 12345, category: electronics)
```

The logical model maps how these entities connect—user owns device, device runs browser, browser displays ad.

***

### Physical Data Modeling

**Purpose:** Actual implementation—schema, data types, storage, compression

**Key Decisions:**
- Column names and data types
- Storage format (Parquet, ORC, CSV)
- Partitioning strategy
- Compression methods
- Indexing approach

**Example:**

```sql
-- Physical implementation of user_clicks fact table
CREATE TABLE user_clicks (
    event_id BIGINT,
    user_id INT,
    device_id INT,
    browser_id INT,
    ad_id INT,
    clicked_at TIMESTAMP,
    session_duration_sec INT
)
PARTITIONED BY (event_date DATE)
STORED AS PARQUET
COMPRESSION SNAPPY;
```

***

## Physical Modeling Techniques

### Dimensional Data Modeling (Traditional)

**Concept:** Separate facts from dimensions, join when needed

**Structure:**
- Fact tables contain metrics and foreign keys
- Dimension tables contain descriptive attributes
- Queries join facts to dimensions

**Example:**

```sql
-- Fact table: purchases
purchase_id | user_id | product_id | amount | purchase_date
1001        | 501     | 2001       | 49.99  | 2025-01-15

-- Dimension table: users
user_id | name  | country | signup_date
501     | Alice | USA     | 2024-06-10

-- Query with join
SELECT u.country, SUM(p.amount) as total_revenue
FROM purchases p
JOIN users u ON p.user_id = u.user_id
GROUP BY u.country;
```

**Pros:**
- Minimal data duplication
- Easy to update dimension attributes
- Most foundational technique

**Cons:**
- Joins required for analysis
- Can be slow at scale

---

### One Big Table (OBT)

**Concept:** Embed all dimensional context directly into the fact table

**Structure:**
- All context pre-joined
- Wide tables with many columns
- Often uses complex data types (structs, arrays, maps)

**Example:**

```sql
-- One Big Table: user_events_obt
event_id | event_type | user_name | user_country | device_type | device_os | browser_name | event_time
1001     | click      | Alice     | USA          | mobile      | iOS       | Safari       | 2025-01-15 14:23:00
1002     | purchase   | Bob       | Canada       | desktop     | Windows   | Chrome       | 2025-01-15 14:25:30
```

**Compute-Storage Tradeoff:**
- Storage: Higher (user_country duplicated for every event)
- Compute: Lower (no joins needed, faster aggregations)

**Pros:**
- Fast aggregations without joins
- Simpler queries

**Cons:**
- Data duplication
- Higher storage costs
- Updates to dimensions require updating many rows

**When to Use:** High-volume analytics where query speed matters more than storage costs.

---

### Data Vault

**Concept:** Preserve data in its rawest, most untransformed state

**Structure:**
- Hubs (unique business keys)
- Links (relationships between hubs)
- Satellites (descriptive attributes with history)

**Philosophy:** Related to ELT (Extract-Load-Transform)—bring data in raw, transform later

**Example:**

```sql
-- Raw inputs table (Data Vault style)
input_id | source_system | raw_json                              | loaded_at
1        | pricing_api   | {"price": 99, "discount": 10, ...}   | 2025-01-15
2        | availability  | {"min_stay": 3, "max_stay": 14, ...} | 2025-01-15

-- You can always trace back to original data if transformation had issues
```

**Pros:**
- Complete audit trail
- Easy to reprocess if transformations were wrong
- Handles source system changes well

**Cons:**
- More complex to query
- Strict methodology can feel restrictive

***

## Practical Application

### Real-World Example: Airbnb Pricing Model

Zach combined all three techniques:

**Data Vault Layer:** Captured raw host pricing rules
```
inputs_table:
- daily_price rules
- discount rules  
- length_of_stay requirements
- advance_booking rules
```

**Processing:** Transformed raw rules into actual prices

**One Big Table Layer:** Created `listing_pricing` table
```sql
listing_pricing:
- listing_id
- all_nights ARRAY<STRUCT<night_date, price, available>>
- pricing_rules STRUCT<base_price, discounts, restrictions>
```

Used complex data types (struct, array) to embed night-level detail in single column.

**Result:** Fast aggregations for pricing analytics without joining multiple tables.

---

## Key Takeaways

Data modeling is more art than science—you can mix techniques based on business needs.

**Best Practices:**
- Start with conceptual modeling to validate requirements
- Use dimensional modeling as your default (most foundational)
- Consider One Big Table when query performance is critical
- Apply Data Vault when audit trails and data lineage matter
- Don't rigidly follow one "religion"—adapt to your tools and requirements

**Career Impact:**
- Strong conceptual modeling = fewer wasted sprints
- Solid logical modeling = maintainable data architecture
- Efficient physical modeling = fast queries and happy stakeholders

The goal is matching business requirements to technical implementation while balancing storage costs, query performance, and maintainability.

***

**Additional Resources:**
- Dimensional modeling deep dives available on Data with Zach's channel
- Practice by modeling familiar domains (e-commerce, social media, healthcare)
- Study existing data models in your organization

[1](https://www.youtube.com/watch/WDwNow61JVE)
