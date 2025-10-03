# 1Pharma_Hackathon - Medicine Search Database Solution

A **FastAPI-based web service** for PostgreSQL database introspection and medicine search with advanced indexing strategies for optimal performance.

---

## Overview
This PostgreSQL schema is optimized for **high-performance medicine search**, supporting multiple search modes: prefix, substring, fuzzy, and full-text.

---

## 🛠️ Setup Instructions (Bash)

```bash
# Step 1: Start PostgreSQL Docker Container
docker run --name 1Pharma_Hackathon \
  -e POSTGRES_PASSWORD=mysecretkey \
  -p 5431:5432 \
  -d postgres
```

```bash
# Step 2: Check Container Status
docker ps -a
```

```bash
# Step 3: Copy Schema File to Container
docker cp ./schema.sql 1Pharma_Hackathon:/schema.sql
```

```bash
# Step 4: Access PostgreSQL in the Container
docker exec -it 1Pharma_Hackathon psql -U postgres

# Inside psql, execute:
# Load schema
\i /schema.sql
# Verify table creation
\dt
# Exit psql
\q
```

```bash
# Step 5: Run Python Import Script (outside container)

#Windows
python importing_data.py

#Mac
python3 importing_data.py
```

```bash
# Step 6: Start the FastAPI Server

#Windows
python main.py

#Mac
python3 main.py

```
---

## Schema.sql Detailed Explanation

The database schema is optimized for high-performance medicine search with multiple indexing strategies.

### Table: `medicines`

Stores all medicine-related information.

| Column | Purpose |
|--------|---------|
| `id` | Primary key |
| `sku_id` | Optional stock-keeping ID |
| `name` | Medicine name (core search field) |
| `search_name` | Normalized name (dosage info removed) |
| `manufacturer_name` | Manufacturer for search/filter |
| `marketer_name` | Optional marketer info |
| `type` | Medicine type (e.g., allopathy) |
| `price` | Numeric price |
| `pack_size_label` | Packaging info |
| `short_composition` | Active ingredients |
| `is_discontinued` | Status flag |
| `available` | Stock availability |
| `slug` | SEO-friendly URL |
| `image_url` | Image reference |
| `tsv` | Precomputed `tsvector` for full-text search |

---

### Performance Indexes

- **Prefix Search Index:** `idx_medicines_lower_name_varchar_pattern` → fast `ILIKE 'prefix%'`.
- **Trigram Indexes:** GIN indexes for fuzzy/substring search on `name`, `manufacturer`, `composition`.
- **Full-text Index:** GIN index on `tsvector` for advanced search.

---

## Indexing Strategy

### 1. `idx_medicines_lower_name_varchar_pattern`
- **Type:** Functional B-Tree index  
- **Purpose:** Optimizes **prefix searches** like `ILIKE 'prefix%'`.  
- **How it works:** Indexes the lowercase version of `name` for fast pattern matching.  
- **Benefit:** Queries such as `WHERE lower(name) LIKE 'para%'` are very fast, avoiding full table scans.

### 2. `idx_medicines_name_trgm`
- **Type:** GIN index with trigrams  
- **Purpose:** Supports **substring and fuzzy searches** on `name`.  
- **How it works:** Breaks text into trigrams and uses similarity functions.  
- **Benefit:** Efficiently handles searches like `ILIKE '%acet%'` or fuzzy matches like `'Paracetmol'`.

### 3. `idx_medicines_manufacturer_trgm`
- **Type:** GIN trigram index  
- **Purpose:** Optimizes **search by manufacturer**.  
- **Benefit:** Quickly finds medicines by partial manufacturer input, e.g., `'Gsk' → 'Glaxo SmithKline'`.

### 4. `idx_medicines_composition_trgm`
- **Type:** GIN trigram index  
- **Purpose:** Optimizes **search by active ingredients / composition**.  
- **Benefit:** Supports queries like `'acetaminophen'` even with partial or fuzzy matches.

### 5. `idx_medicines_tsv_gin`
- **Type:** GIN index on `tsvector`  
- **Purpose:** Enables **full-text search** across multiple fields.  
- **How it works:** Weighted `tsvector` combines:
  - **A →** `name`  
  - **B →** `short_composition`  
  - **C →** `manufacturer_name`  
- **Benefit:** Allows relevance-ranked full-text searches prioritizing medicine names.

### 6. `idx_medicines_name_lower`
- **Type:** B-Tree index on `lower(name)`  
- **Purpose:** Optional index to speed up **similarity ordering**.  
- **Benefit:** Useful for queries that order by similarity or need quick case-insensitive sorting.

---

### Trigger Function

The trigger `medicines_tsv_trigger()` automatically:

- Maintains normalized `search_name`.
- Builds weighted `tsvectors`.

---

## File Descriptions

### Core Application Files

- **main.py** → FastAPI app with four endpoints (prefix, substring, fulltext, fuzzy). Uses asyncpg pooling.
- **connection_test.py** → FastAPI DB introspection API with `/tables` and `/table/{table_name}`.
- **importing_data.py** → Bulk JSON importer using PostgreSQL `COPY`.

### Frontend

- **index.html** → User interface with:
  - 500ms debouncing
  - Pagination (5 results per page)
  - Four search modes

### Database Schema

- **schema.sql** → Optimized PostgreSQL schema for search performance.

---

## Frontend UI Features

- **Search Modes:** Prefix, Substring, Full-text, Fuzzy.
- **Debouncing:** 500ms delay before sending requests.
- **Modals:** Client-side, Elegant Modal displayed for each medicine.

---

## Performance Approach

### Indexing Strategy

- **Prefix Search** → `varchar_pattern_ops` index.
- **Substring / Fuzzy Search** → PostgreSQL `pg_trgm` GIN index.
- **Full-text Search** → Weighted `tsvector`:
  - **A →** name
  - **B →** composition
  - **C →** manufacturer

### Connection Management

- **main.py** → asyncpg connection pooling (1–100).
- **importing_data.py** → Bulk data load via `COPY`.

### Search Query Optimization

- **Prefix:** `lower(name) LIKE lower($1) || '%'`.
- **Substring:** Trigram similarity scoring.
- **Full-text:** `ts_rank(tsv, query)`.
- **Fuzzy:** Trigram similarity (>0.2) + prefix priority.

---

## Benchmark Results

Below are the measured average latency and throughput for each search mode, based on 50 concurrent requests per query.

| Query      | Endpoint          | Avg Latency (ms) | Throughput (req/s) |
| ---------- | ----------------- | ---------------- | ------------------ |
| Ava        | /search/prefix    | 434.880          | 102.23             |
| Injection  | /search/substring | 2097.66          | 17.99              |
| antibiotic | /search/fulltext  | 253.71           | 104.8              |
| Avastn     | /search/fuzzy     | 8308.11          | 5.247              |

- **Avg Latency (ms):** Average response time per request.
- **Throughput (req/s):** Number of requests handled per second.

---
