# Chapter 3: Storage and Retrieval

## Beginner-Friendly Explanation
Imagine you own a library. When someone returns a book, you can either:
1. **Throw it anywhere** on the nearest available shelf (fast to insert, slow to find later).
2. **File it precisely** in the Dewey Decimal System (takes longer to file, but finding it later is instant).

Databases face the exact same trade-off. They need to decide: how do we physically write data to disk so that we can store it *fast* AND retrieve it *fast* later?

This chapter peels back the curtain and shows you exactly what is happening inside the database engine at the disk level. Two dominant storage engine families emerge: **Log-Structured (LSM-Trees)** and **Page-Oriented (B-Trees)**.

---

## Core Concepts
* **Storage Engines:** Low-level components responsible for physically writing data to disk and reading it back.
* **Index:** A data structure that speeds up reads by organizing a pointer map of where data lives on disk. It slows down writes (you have to update the index every time you write).
* **Hash Index:** The simplest index — an in-memory key-value map pointing to byte offsets on disk.
* **SSTables (Sorted String Tables):** Log segments where key-value pairs are sorted by key.
* **LSM-Trees (Log-Structured Merge-Trees):** A write-optimized storage engine used in LevelDB, RocksDB, Cassandra.
* **B-Trees:** The universal standard index structure used in PostgreSQL, MySQL, Oracle, and almost every RDBMS.
* **OLTP vs OLAP:** Online Transaction Processing (many small, fast queries) vs. Online Analytical Processing (few, massive analytical queries over historical data).
* **Column-Oriented Storage:** Storing data by column rather than row for extreme analytical query speed.

---

## Deep Understanding

### 1. The Simplest Possible Database (Log + Hash Index)
Let's build a conceptual database in pure Bash. A database at its most fundamental is just:
```bash
# WRITE: Append key=value to a file
db_set() { echo "$1,$2" >> database.log }

# READ: Scan the file for the latest key
db_get() { grep "^$1," database.log | tail -n 1 | cut -d',' -f2 }
```
* **Write:** Appending to a file is the fastest possible disk operation. O(1).
* **Read:** Scanning the entire file is hideously slow. O(n).

To make reads fast, we add a **Hash Index** — an in-memory HashMap where:
```text
key  -->  byte_offset_in_file
"user1" --> 0 bytes
"user2" --> 47 bytes
"user3" --> 93 bytes
```
Now reads are O(1) — jump directly to the byte offset on disk.

**The Problem:** We keep appending forever. The file grows infinitely. Solution: **Compaction & Segmentation** — periodically merge segments, throwing away duplicate keys (only keeping the most recent value).

---

### 2. SSTables and LSM-Trees (Write-Optimized)
The log-based approach has a flaw: the Hash Index only works if all keys fit in RAM. What if you have billions of keys?

**SSTables (Sorted String Tables)** solve this by requiring that keys inside each log segment are **sorted alphabetically**. Now you only need a *sparse* index in memory (one pointer every few kilobytes), because you can binary-search within the sorted segment.

```text
SSTable Segment (sorted by key):
[banana: 5] [cherry: 12] [grape: 3] [mango: 18]

Sparse In-Memory Index:
"banana" --> offset 0
"grape"  --> offset 200

To find "cherry": Jump to "banana" offset, scan forward until you find "cherry". Fast!
```

**LSM-Tree (Log-Structured Merge-Tree)** = the algorithm for managing SSTables:
1. **Writes go to an in-memory sorted structure (Memtable)** — fast in-memory insertion.
2. **When the Memtable fills up** → flush it to disk as an SSTable segment.
3. **Background compaction** merges multiple SSTable segments together, removing obsolete versions of keys.

```text
LSM-Tree Write Path:

[Write Request] 
     |
     v
[WAL (Write-Ahead Log)] -- crash safety
     |
     v
[Memtable (in-memory sorted tree - Red-Black or AVL Tree)]
     |
     | (Memtable full -> flush)
     v
[SSTable L0 on disk]
     |
     | (compaction)
     v
[SSTable L1, L2, L3... (progressively larger, sorted on disk)]
```

**Used by:** LevelDB, RocksDB, Cassandra, HBase, Elasticsearch.

---

### 3. B-Trees (Read-Optimized Standard)
B-Trees are the dominant index structure in almost every RDBMS. Instead of append-only log segments, B-Trees organize data into **fixed-size pages** (traditionally 4KB, mirroring disk block sizes).

```text
B-Tree Structure:

                   [Root Page]
                   [10 | 20 | 30]
                  /   |    |    \
              [1-9] [11-19] [21-29] [31+]
              (Leaf Pages with actual data)
```

* **Pages** are the fundamental read/write unit.
* Each page has references (pointers on disk) to child pages.
* To **find a key**: Start at root, follow pointers down the tree until hitting a leaf page. O(log n).
* To **insert a key**: Find the page, insert in sorted order. If the page is full, **split** it into two pages and update the parent. This can cascade upward.

**Write-Ahead Log (WAL):** Before modifying any B-Tree page, the operation is recorded in a crash-recovery journal. If the database crashes mid-split, the WAL is replayed to restore consistency.

---

### 4. B-Tree vs LSM-Tree: The Central Trade-off

```text
╔══════════════════╦════════════════════════╦══════════════════════════╗
║ Property         ║ B-Tree                 ║ LSM-Tree                 ║
╠══════════════════╬════════════════════════╬══════════════════════════╣
║ Write Speed      ║ Slower (random writes) ║ FASTER (sequential       ║
║                  ║                        ║ appends, batched)        ║
╠══════════════════╬════════════════════════╬══════════════════════════╣
║ Read Speed       ║ FASTER (single tree    ║ Slower (check Memtable + ║
║                  ║ traversal)             ║ multiple SSTables)       ║
╠══════════════════╬════════════════════════╬══════════════════════════╣
║ Write Amplif.    ║ Higher (page + WAL)    ║ Also exists (compaction) ║
╠══════════════════╬════════════════════════╬══════════════════════════╣
║ Space            ║ Some fragmentation     ║ More efficient with      ║
║                  ║ (page splits)          ║ compression              ║
╠══════════════════╬════════════════════════╬══════════════════════════╣
║ Best For         ║ Read-heavy workloads   ║ Write-heavy workloads    ║
║ Typical Use      ║ PostgreSQL, MySQL       ║ Cassandra, HBase,        ║
║                  ║                        ║ RocksDB                  ║
╚══════════════════╩════════════════════════╩══════════════════════════╝
```

---

### 5. OLTP vs OLAP

```text
OLTP (Online Transaction Processing):
- Many small, fast queries
- User-facing operations (login, checkout, update record)
- Touches small number of rows per query
- Example: "Get order #12345 for user ABC"

OLAP (Online Analytical Processing):
- Few, massive queries over millions/billions of rows
- Business intelligence and reporting (not user-facing)
- Example: "What were total sales by region for Q3 2024?"
```

These two workloads have dramatically different patterns. Running OLAP queries on your OLTP database is catastrophic — you'll block user transactions with your massive full-table scans. That's why we have **Data Warehouses** (Redshift, BigQuery, Snowflake) — separate, purpose-built systems for OLAP queries.

---

### 6. Column-Oriented Storage

Traditional (row-oriented) storage lays out all columns for one row next to each other:
```text
Row-Oriented Layout:
[user1, Alice, 28, NYC] [user2, Bob, 35, LA] [user3, Carol, 22, Chicago]
```

For OLAP queries like `SELECT AVG(age) FROM users`, you still read Alice's name and city — wasted disk reads.

Column-Oriented Storage lays out all values for one *column* next to each other:
```text
Column-Oriented Layout:
Names:  [Alice, Bob, Carol]
Ages:   [28, 35, 22]
Cities: [NYC, LA, Chicago]
```

Now `SELECT AVG(age)` reads only the Ages column. It skips everything else. Plus, columns compress incredibly well (repeated values like city names can be run-length or bitmap encoded).

**Used by:** Amazon Redshift, Google BigQuery, Apache Parquet, Apache Cassandra (partially), ClickHouse.

---

## Real-World Examples

* **LSM-Tree in practice (Cassandra):** Instagram uses Cassandra to store hundreds of billions of photo metadata records. Cassandra's LSM-Tree architecture handles a massive volume of writes every second (likes, follows, new photos) — the write-optimized nature is exactly what they need.
* **B-Tree in practice (PostgreSQL):** A bank's transaction system uses PostgreSQL. Every transaction needs to be reliably read back immediately. B-Trees give fast, predictable single-row lookups with strong consistency guarantees.
* **Column storage (BigQuery):** An e-commerce company runs nightly analytical queries — "total revenue by product category for all orders over the last year" — over 10 billion rows. With row storage this would take hours. With BigQuery's columnar storage and distributed query engine, it takes seconds.

---

## Visual Diagrams

```text
Storage Engine Decision Tree:

Is your workload Write-Heavy?
       |
       YES ----> LSM-Tree (Cassandra, RocksDB, HBase)
       |
       NO
       |
Is it Read-Heavy with Point Lookups?
       |
       YES ----> B-Tree (PostgreSQL, MySQL, SQLite)
       |
       NO
       |
Is it Analytical (scanning massive datasets)?
       |
       YES ----> Columnar Storage (BigQuery, Redshift, ClickHouse)
```

---

## Step-by-Step Walkthroughs

### What happens when you write a row in PostgreSQL?
1. Transaction starts, assigned a Transaction ID.
2. The write is first recorded in the **WAL (Write-Ahead Log)** — crash safety.
3. The page containing the relevant B-Tree leaf is located and loaded into the **Buffer Pool** (in-memory cache).
4. The row is inserted into the page.
5. The page is marked as "dirty" (modified but not yet flushed to disk).
6. Eventually, a background process (**checkpointer**) flushes dirty pages to the actual data files on disk.

### What happens when you write a row in Cassandra?
1. Write is recorded in the **Commit Log** (disk, sequential — crash safety).
2. Write is inserted into the **Memtable** (in-memory sorted data structure). Done — write acknowledged instantly.
3. When Memtable reaches threshold → flushed to a new **SSTable** on disk.
4. Background **compaction** periodically merges SSTables, removing obsolete records.

---

## Beginner Mistakes to Avoid
* **"Adding more indexes = faster queries":** Indexes speed up reads but **slow down writes**. Every write must update every index you have. Over-indexing an OLTP database is a real production mistake.
* **"My OLTP database can handle analytics too":** Running a complex `GROUP BY` with no indexes on a production OLTP database causes a full table scan that can lock tables and degrade performance for all users.
* **"LSM-Tree reads are instant":** A read in an LSM-Tree storage engine may have to check the Memtable AND multiple SSTable levels. This is slower than B-Tree point lookups. Bloom Filters are used to quickly skip SSTables that definitely don't contain the requested key.

---

## Intermediate to Advanced Insights
* **Write Amplification:** Both B-Trees and LSM-Trees suffer from write amplification — one logical write by the application can result in many physical writes to disk (WAL, page writes, compaction). This is critical on SSDs, which have limited write cycles (endurance).
* **Bloom Filters:** An LSM-Tree optimization. A probabilistic data structure that tells you with 100% certainty if a key is *not* in a particular SSTable. This avoids unnecessary disk reads when searching for non-existent keys. "Definitely Not Here" vs "Probably Here."
* **Compaction Strategies:** Different LSM-Tree databases use different compaction strategies:
  * **Size-Tiered (Cassandra default):** Group SSTables of similar size together and merge.
  * **Leveled (LevelDB):** Organize SSTables into levels with strict size limits. Better for reads, but more I/O during compaction.

---

## Mastery Notes
* **Buffer Pool / Page Cache:** Both RDBMS (PostgreSQL's buffer pool) and the OS (page cache) try to keep frequently-accessed disk pages in RAM to avoid I/O. Understanding when you are hitting RAM vs disk is key to understanding query latency patterns.
* **The Elephant in the Room — fsync():** After writing the WAL, a database must call `fsync()` to force the OS to flush the data to physical disk. Without `fsync()`, a crash could corrupt data that was "written" but still sitting in OS buffers. `fsync()` is synchronous and very slow — it's one of the most critical latency sources in any database.
* **Log-Structured everything:** The insight from LSM-Trees has spread far beyond storage engines. Kafka (message queue), HDFS (distributed filesystem), and most distributed databases prefer the log-structured, append-only model because it is the perfect fit for sequential disk I/O and crash recovery.

---

## Summary
At the hardware level, databases make a fundamental choice about how to write and find data. **B-Trees** (used in PostgreSQL, MySQL) organize data in balanced tree pages — optimal for read-heavy, transactional workloads. **LSM-Trees** (used in Cassandra, RocksDB) use append-only log segments sorted into SSTables — optimal for write-heavy, high-throughput workloads. For analytical workloads, **Column-Oriented Storage** (BigQuery, Redshift) reads only the columns you query, dramatically reducing disk I/O. Choosing the right storage engine is the silent foundation of every high-performance system.

---

## Practice Questions

**Beginner:**
1. What is a database index, and why does adding one slow down writes?
2. In B-Trees, what is a "page" and why is its size usually aligned to 4KB?
3. What is the difference between OLTP and OLAP workloads? Give one example of each.

**Intermediate:**
1. Explain the LSM-Tree write path end-to-end, from a client write request all the way down to data being stored in an SSTable on disk.
2. Why is a Bloom Filter critical for read performance in LSM-Tree based storage engines?

**Advanced:**
1. Explain "write amplification" — what causes it in both B-Trees and LSM-Trees? What is the practical consequence of high write amplification on SSD-based storage? How would you architect around it?
