Data normalization is the discipline of structuring relational data so every piece of information lives in one place, every dependency is explicit, and the schema naturally prevents bad data.  This masterclass will walk from “noob” intuition to “senior engineer” judgment: how to think about normal forms, apply them step‑by‑step, and know when to stop or even denormalize for analytics.[1][2][3][4]

***

### Big picture: why normalize

- Normalization is the process of organizing tables and relationships to reduce redundancy and anomalies in inserts, updates, and deletes.[3][1]
- A well‑normalized schema encodes business rules as structure, making it hard to store impossible data (e.g., an order without a customer, two conflicting addresses) and easier to query safely.[2][5]

***

### Two mental models: before any “NF”

- First, think in terms of entities and relationships: Customers, Orders, Products, Addresses, Payments; then 1‑to‑many, many‑to‑many, and subtype relationships between them.[6][5]
- Second, think in terms of dependencies: “if I know X, I can determine Y”; these functional dependencies are what normal forms constrain.[7][3]

***

### Step 1: from mess to UNF

- Real‑world source data (CSVs, spreadsheets, APIs) is usually in unnormalized form (UNF): repeated groups, multi‑valued cells, inconsistent columns, and no stable keys.[8][3]
- At this stage, the goal is simply to identify repeating patterns and candidate keys (natural or surrogate) and list all attributes you think belong together in one “logical row”.[5][9]

Example UNF “Orders” sheet:  
`OrderId, OrderDate, CustomerName, CustomerCity, Product1, Product2, Product3` where each row has up to 3 product columns.[2][8]

***

### First Normal Form (1NF): atomic values

- 1NF requires: each table has a primary key, no repeating groups, and each column holds atomic (indivisible) values instead of lists or nested structures.[6][2]
- Practical heuristics: no arrays in a single cell, no “comma‑separated” lists, and no sets of numbered columns like Product1, Product2, Product3 in the same table.[4][8]

How to normalize to 1NF for the Orders example:  

- Introduce a separate line‑items table: `OrderLines(OrderId, LineNumber, ProductId, Quantity, ...)`, one row per product on an order.[10][2]
- Keep `Orders(OrderId, OrderDate, CustomerId, ...)` with one row per order and a primary key on `OrderId`.[2][6]

Result: no multi‑valued attributes, and every row is uniquely identifiable by its key.[4][2]

***

### Second Normal Form (2NF): no partial dependencies

- 2NF only matters when the primary key is composite: it says every non‑key attribute must depend on the whole key, not just part of it.[4][2]
- A partial dependency happens when an attribute is determined by only one component of a composite key; this leads to redundancy and anomalies.[7][4]

Classic example:  

`OrderLines(OrderId, ProductId, OrderDate, CustomerId, ProductName, UnitPrice)` with primary key `(OrderId, ProductId)`.[6][2]

- `OrderDate` and `CustomerId` depend only on `OrderId`, not `ProductId` → partial dependency.[5][4]
- `ProductName` likely depends only on `ProductId` → another partial dependency.[7][4]

To reach 2NF:  

- Move order‑level attributes to `Orders(OrderId, OrderDate, CustomerId, ...)`.[2][6]
- Move product‑level attributes to `Products(ProductId, ProductName, UnitPrice, ...)`.[10][2]

Now `OrderLines(OrderId, ProductId, Quantity)` has only attributes that depend on both parts of its key.[4][7]

***

### Third Normal Form (3NF): no transitive dependencies

- 3NF builds on 2NF: remove transitive dependencies, where a non‑key attribute depends on another non‑key attribute, which then depends on the key.[2][4]
- Informally: every non‑key column should describe the key, not other non‑key columns.[3][7]

Example:  

`Customers(CustomerId, CustomerName, Zip, City, State)` with primary key `CustomerId`.[5][6]

- `CustomerName` depends on `CustomerId` → good.[6][5]
- `Zip` depends on `CustomerId` → also good.[5][6]
- But `City` and `State` are determined by `Zip` (assuming postal data is clean), so `CustomerId → Zip → City, State` is a transitive dependency.[6][4]

3NF fix:  

- Create `ZipCodes(Zip, City, State)` and remove `City` and `State` from `Customers`, keeping a foreign key from `Customers.Zip` to `ZipCodes.Zip`.[11][6]

Now every non‑key attribute in `Customers` depends directly and only on `CustomerId`.[4][6]

***

### Boyce–Codd Normal Form (BCNF): every determinant is a key

- BCNF is a stricter version of 3NF: for every non‑trivial functional dependency $$X \to Y$$, $$X$$ must be a candidate key.[3][7]
- Many schemas that “look fine” in 3NF still violate BCNF when there are overlapping candidate keys or tricky business rules.[11][7]

Example pattern:  

`Teaching(Professor, Course, Room)` with dependencies: each course is always taught in one room, and each professor teaches exactly one course, but a course might be taught by multiple professors.[11][3]

- Functional dependencies might include `Course → Room` and `Professor → Course`.[3][7]
- `Professor` is a key (since knowing professor determines course and room), but `Course` is not a key if multiple professors can teach the same course; yet `Course → Room` exists, violating BCNF.[11][3]

BCNF fix: decompose into:  

- `ProfessorCourse(Professor, Course)` and `CourseRoom(Course, Room)`, eliminating the problematic dependency.[7][3]

In practice, BCNF is where senior engineers start thinking carefully about edge dependencies and whether the extra joins are worth it.[11][5]

***

### Higher normal forms (4NF, 5NF, 6NF): when they matter

- 4NF deals with multivalued dependencies (e.g., independently varying sets of attributes per key) and ensures no non‑trivial multivalued dependency other than a key.[3][7]
- 5NF deals with join dependencies where a relation can be reconstructed from multiple projections but has no redundant combinations; it is rare in everyday OLTP schema design.[7][3]

For most line‑of‑business systems, aiming for 3NF or BCNF with awareness of 4NF scenarios is enough; 5NF and beyond show up in specialized modeling or very complex constraints.[8][5]

***

### The “anomalies” lens: what normalization prevents

- Insertion anomaly: you cannot store a fact unless you also create unrelated facts (e.g., cannot add a new product without creating an order row).[10][5]
- Update anomaly: you must update the same logical fact in many rows (e.g., customer address stored in multiple orders) and risk inconsistent data.[10][5]
- Deletion anomaly: deleting one fact accidentally deletes another (e.g., removing the last order for a product also loses the only record of that product’s existence).[10][11]

When reviewing a design, explicitly test for these anomalies to spot normalization issues quickly.[9][11]

***

### Step‑by‑step normalization workflow (how seniors think)

When handed a messy design or new feature, a senior engineer typically does:[9][5]

1. **List entities and keys**  
   - Identify natural keys (like `Email`, `ISOCode`, `TaxId`) and decide where surrogate keys (`Id` identity/UUID) are justified.[5][6]

2. **Capture dependencies**  
   - For each entity, write sentences: “Knowing X, I can determine Y”, capturing business rules, not guesses.[3][4]

3. **Enforce 1NF**  
   - Remove repeating groups, multi‑valued attributes, keyless tables; introduce line tables for 1‑to‑many lists.[2][6]

4. **Enforce 2NF and 3NF**  
   - On any composite‑key table, check for partial and transitive dependencies, then decompose into new tables as needed.[4][7]

5. **Check for BCNF hotspots**  
   - Where there are multiple candidate keys or unusual business rules, verify that every determinant is a candidate key, or decide explicitly to live with a controlled violation.[11][3]

6. **Validate against use cases**  
   - Test inserts, updates, deletes, and core queries; ensure joins are reasonable for expected workloads.[9][5]

***

### Normalization vs denormalization: performance trade‑offs

- Strict normalization minimizes redundancy and anomalies but can introduce many joins, which may impact performance for read‑heavy analytics or complex reports.[5][2]
- Denormalization intentionally duplicates some data (e.g., caching a `CustomerCity` on `Orders`) for performance or convenience, while using application logic, triggers, or pipelines to keep it consistent.[12][2]

Typical senior‑level pattern:  

- Model the “source of truth” OLTP schema in 3NF/BCNF.[9][5]
- Build derived denormalized tables, marts, or materialized views for analytics and dashboards.[12][2]

***

### SQL patterns: how normalization looks in code

- Creating normalized tables usually means splitting a wide table and adding foreign keys, as in:  

```sql
CREATE TABLE Customers (
    CustomerID   INT PRIMARY KEY,
    CustomerName VARCHAR(100)
);

CREATE TABLE Orders (
    OrderID     INT PRIMARY KEY,
    CustomerID  INT NOT NULL,
    OrderDate   DATE NOT NULL,
    FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
);
```


- Transitioning towards 3NF often involves `ALTER TABLE` to drop columns and create new tables capturing dependent data, with foreign keys to preserve relationships.[10][6]

Practicing these transformations on sample schemas (like customers, orders, projects) rapidly builds intuition.[13][4]

***

[17](https://www.youtube.com/watch?v=rBPQ5fg_kiY)
[18](https://www.reddit.com/r/SQL/comments/1f852xd/what_is_this_normalization_and_how_can_i/)
[19](https://stackoverflow.com/questions/13173615/normalizing-values-in-sql-query)
