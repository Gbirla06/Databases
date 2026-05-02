# Short Notes: Chapter 2 - Data Models and Query Languages

## 1. The Big Three Data Models
* **Document Model (NoSQL, e.g., MongoDB):**
  * Data stored as self-contained JSON documents.
  * **Pros:** Great for unstructured data, avoids object-relational impedance mismatch, excellent data locality (fast to read a whole document).
  * **Cons:** Terrible at Many-to-Many relationships (requires manual application-side JOINs or data duplication).
  * **Schema Concept:** Schema-on-read (implicit schema enforced by the app code).
* **Relational Model (SQL, e.g., PostgreSQL):**
  * Data stored in strict tables, rows, and columns.
  * **Pros:** Standardized, Handles Many-to-Many relationships easily via Database JOINs, proven ecosystem.
  * **Cons:** Slower to scale out horizontally, object-mismatch requires ORM frameworks.
  * **Schema Concept:** Schema-on-write (strict database enforced schemas).
* **Graph Model (e.g., Neo4j):**
  * Designed for highly connected data (Nodes and Edges). Good for social graphs, recommendations.

## 2. Query Languages
* **Imperative:** Step-by-step code telling the DB *how* to get data. (Hard to automatically optimize).
* **Declarative (SQL):** Tell the DB *what* you want. The DB's Query Optimizer figures out the fastest physical execution path. Better for long-term evolvability.

## 3. Storage Locality
* If you need an entire object at once, a Document DB naturally stores it in one contiguous block on the physical disk for fast retrieval.
* An RDBMS splits a complex object across multiple normalization tables, requiring multiple disk jumps (or complex B-tree memory fetches).

## Key Takeaway
Don't use NoSQL just because it's newer. Use Document DBs when data relationships are a "tree" (one-to-many). Use Relational or Graph DBs when relationships are a "web" (many-to-many). Modern massive architectures often mix both (Polyglot Persistence).
