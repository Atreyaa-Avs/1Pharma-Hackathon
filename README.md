# 1Pharma_Hack - Medicine Search Database Solution

A **FastAPI-based web service** for PostgreSQL database introspection and medicine search with advanced indexing strategies for optimal performance.

---

## 🛠️ Setup Instructions (Bash)

```bash
# Step 1: Start PostgreSQL Docker Container
docker run --name 1Pharma_Hackathon \
  -e POSTGRES_PASSWORD=mysecretkey \
  -p 5431:5432 \
  -d postgres

# Step 2: Check Container Status
docker ps -a

# Step 3: Copy Schema File to Container
docker cp ./schema.sql 1Pharma_Hackathon:/schema.sql

# Step 4: Access PostgreSQL in the Container
docker exec -it 1Pharma_Hackathon psql -U postgres

# Inside psql, execute:
# Load schema
\i /schema.sql
# Verify table creation
\dt
# Exit psql
\q

# Step 5: Run Python Import Script (outside container)
python3 importing_data.py

# Step 6: Start the FastAPI Server
python3 main.py

---

## 🗄️ Schema.sql Detailed Explanation

The database schema is optimized for high-performance medicine search with multiple indexing strategies.

### Table Structure

- **medicines** → Primary table storing medicine information (`id, sku_id, name, manufacturer, pricing, etc.`).
- **search_name** → Normalized column for dosage-independent searches.
- **tsv** → Materialized `tsvector` column for full-text search.

### Performance Indexes

- **Prefix Search Index:** `idx_medicines_lower_name_varchar_pattern` → fast `ILIKE 'prefix%'`.
- **Trigram Indexes:** GIN indexes for fuzzy/substring search on `name`, `manufacturer`, `composition`.
- **Full-text Index:** GIN index on `tsvector` for advanced search.

### Trigger Function

The trigger `medicines_tsv_trigger()` automatically:

- Maintains normalized `search_name`.
- Builds weighted `tsvectors`.

---

## 📂 File Descriptions

### Core Application Files

- **main.py** → FastAPI app with four endpoints (prefix, substring, fulltext, fuzzy). Uses asyncpg pooling.
- **conn_test.py** → FastAPI DB introspection API with `/tables` and `/table/{table_name}`.
- **import_data.py** → Bulk JSON importer using PostgreSQL `COPY`.

### Frontend

- **index.html** → User interface with:
  - 500ms debouncing
  - Pagination (5 results per page)
  - Four search modes

### Database Schema

- **schema.sql** → Optimized PostgreSQL schema for search performance.

---

## 🎨 Frontend UI Features

- **Search Modes:** Prefix, Substring, Full-text, Fuzzy.
- **Debouncing:** 500ms delay before sending requests.
- **Pagination:** Client-side, 5 results per page.

---

## ⚡ Performance Approach

### Indexing Strategy

- **Prefix Search** → `varchar_pattern_ops` index.
- **Substring / Fuzzy Search** → PostgreSQL `pg_trgm` GIN index.
- **Full-text Search** → Weighted `tsvector`:
  - **A →** name
  - **B →** composition
  - **C →** manufacturer

### Connection Management

- **main.py** → asyncpg connection pooling (1–100).
- **import_data.py** → Bulk data load via `COPY`.

### Search Query Optimization

- **Prefix:** `lower(name) LIKE lower($1) || '%'`.
- **Substring:** Trigram similarity scoring.
- **Full-text:** `ts_rank(tsv, query)`.
- **Fuzzy:** Trigram similarity (>0.2) + prefix priority.

---

## 📊 Benchmark Results

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
