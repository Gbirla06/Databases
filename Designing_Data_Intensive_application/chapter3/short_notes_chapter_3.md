# Short Notes: Chapter 3 - Storage and Retrieval

## Core Idea
Databases make two fundamental choices: **how to write data** to disk and **how to find it** again. These choices define performance characteristics entirely.

---

## 1. Log + Hash Index (Simplest DB)
* **Write:** Append key-value to a file → O(1), fastest possible.
* **Read:** HashMap in memory maps `key → byte_offset` on disk → O(1).
* **Compaction:** Periodically merge log segments, keep only latest value per key.
* **Limitation:** All keys must fit in RAM.

---

## 2. SSTables + LSM-Trees (Write-Optimized)
* **SSTable:** Log segment where keys are **sorted**. Allows sparse in-memory index (one pointer per few KB).
* **LSM-Tree Write Path:**
  1. Write → WAL (crash safety) → Memtable (in-memory sorted tree)
  2. Memtable full → flush as SSTable to disk (L0)
  3. Background compaction merges SSTables across levels (L1, L2, L3...)
* **Bloom Filter:** Probabilistic structure that quickly rules out SSTables that don't have a key → avoids unnecessary disk I/O.
* **Used by:** Cassandra, RocksDB, LevelDB, HBase, Elasticsearch.
* **Best for:** Write-heavy, high-throughput workloads.

---

## 3. B-Trees (Read-Optimized Standard)
* Data organized in **fixed-size pages** (~4KB), mirroring disk block size.
* **Reads:** Traverse tree from Root → Branch → Leaf. O(log n).
* **Writes:** Find leaf, insert in sorted order. Full page → **split** into two pages.
* **WAL (Write-Ahead Log):** Every modification journaled before page is touched → crash safety.
* **Used by:** PostgreSQL, MySQL, SQLite, Oracle.
* **Best for:** Read-heavy, transactional (OLTP) workloads.

---

## 4. B-Tree vs LSM-Tree Quick Comparison

| Property       | B-Tree              | LSM-Tree               |
|----------------|---------------------|------------------------|
| Write Speed    | Slower              | Faster (sequential)    |
| Read Speed     | Faster              | Slower (multi-level)   |
| Best For       | OLTP (reads)        | Write-heavy            |
| Used By        | PostgreSQL, MySQL   | Cassandra, RocksDB     |

---

## 5. OLTP vs OLAP
| | OLTP | OLAP |
|---|---|---|
| Query Type | Many small, fast | Few, massive |
| Scale | Small rows | Millions/billions of rows |
| Example | User login, order insert | Revenue by region report |
| Storage | Row-oriented DB | Column-oriented DB / Data Warehouse |

---

## 6. Column-Oriented Storage
* Stores all values for one **column** contiguously instead of all columns for one **row**.
* Dramatically reduces disk I/O for analytical queries (only reads relevant columns).
* Enables excellent **compression** (repeated values in a column compress better).
* **Used by:** BigQuery, Amazon Redshift, Apache Parquet, ClickHouse.

---

## Key Takeaways
* Every index speeds up reads but **slows down writes** — choose indexes deliberately.
* Write-heavy? → LSM-Tree. Read-heavy? → B-Tree. Analytical? → Columnar.
* Write Amplification (one logical write → many physical writes) affects SSD lifespan.
* `fsync()` is the most critical latency source — it forces OS buffer flush to physical disk.
* Always separate OLAP workloads into a **Data Warehouse** — never run analytics on your production OLTP DB.
