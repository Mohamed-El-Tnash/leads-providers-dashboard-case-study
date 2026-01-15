# Leads Providers Dashboard

A production-grade BI dashboard built with Apache Superset to analyze 7.5M+ lead records, tracking distribution, duplication, and provider overlap.

> ⚠️ **Note:** This is a documentation-only repository. No production credentials or raw client data included.

---

## 📺 See It In Action

Full walkthrough video: [Leads Providers Dashboard - Full Application Walkthrough](https://youtu.be/2elyIq8kp7o)

---

## Project Goals

- Centralize **7.5M+ lead records** from 300+ fragmented CSV/Excel files
- Normalize highly inconsistent client-provided data
- Design a scalable analytical database schema
- Eliminate expensive runtime joins for fast BI queries
- Deliver interactive dashboards with sub-second response times
- Enable role-based access control for different user types

---

## Tech Stack

| Layer            | Technology             |
| ---------------- | ---------------------- |
| Language         | Python, SQL            |
| Package Manager  | pip                    |
| Data Processing  | Pandas                 |
| Database         | PostgreSQL             |
| DB Driver        | psycopg2               |
| BI Platform      | Apache Superset        |
| Containerization | Docker, Docker Compose |

> **Note:** Superset internally uses Flask, SQLAlchemy, Celery, Redis, and React. These are treated as managed infrastructure since Superset is deployed as a containerized service.

---

## The Data Challenge

**Input:** 300+ CSV and Excel files with inconsistent formatting

**Issues:**
- Phone numbers in multiple formats (dashes, spaces, parentheses)
- Misspelled or invalid state/area values
- No unique identifiers across files
- Duplicate leads submitted by multiple providers

**Output:** Single unified dataset with 7.5M clean, analytics-ready records

---

## Data Pipeline

### Cleaning & Transformation
Custom Python pipeline to:
- Normalize phone numbers (digits-only, consistent length)
- Validate and standardize state/area values
- Remove corrupt or unparseable rows
- Deduplicate while preserving provider attribution
- Merge all sources into single CSV

### Database Schema Design

**1. `leads` table** - Stores unique leads
```sql
CREATE TABLE leads (
    id SERIAL PRIMARY KEY,
    phone VARCHAR(15) UNIQUE NOT NULL,
    area VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
- Enforces phone number uniqueness
- Enables fast regional filtering

**2. `providers` table** - Reference table for providers
```sql
CREATE TABLE providers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
- Clean normalized lookup
- Extensible for future metadata

**3. `lead_sources` table** - Junction table tracking submissions
```sql
CREATE TABLE lead_sources (
    id SERIAL PRIMARY KEY,
    lead_id INTEGER REFERENCES leads(id) ON DELETE CASCADE,
    provider_id INTEGER REFERENCES providers(id) ON DELETE CASCADE,
    submitted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (lead_id, provider_id)
);
```
- Enables duplication analysis
- Tracks which provider submitted which lead
- Supports overlap metrics

### Final Ingestion Results

| Table          | Records   |
| -------------- | --------- |
| leads          | 6,365,496 |
| providers      | 4         |
| lead_sources   | 7,559,525 |

---

## Performance Optimization

### Materialized View Strategy

To avoid expensive joins at query time, created a pre-computed view:

```sql
CREATE MATERIALIZED VIEW dashboard_leads_view AS
SELECT
    l.phone,
    l.area,
    p.name AS provider,
    STRING_AGG(DISTINCT p2.name, ', ') AS all_providers,
    COUNT(DISTINCT p2.id) AS provider_count,
    l.created_at
FROM leads l
JOIN lead_sources ls ON l.id = ls.lead_id
JOIN providers p ON ls.provider_id = p.id
-- ... (aggregation logic)
```

**Why this matters:**
- Eliminates runtime joins across 7.5M rows
- Pre-computes provider overlap metrics
- Enables sub-second dashboard responses
- Superset queries a flat, indexed view instead of joining 3 tables

### Indexing Strategy

```sql
-- Base tables
CREATE INDEX idx_leads_phone ON leads(phone);
CREATE INDEX idx_leads_area ON leads(area);
CREATE INDEX idx_lead_sources_lead_id ON lead_sources(lead_id);
CREATE INDEX idx_lead_sources_provider_id ON lead_sources(provider_id);

-- Materialized view (dashboard queries)
CREATE INDEX idx_dashboard_phone ON dashboard_leads_view(phone);
CREATE INDEX idx_dashboard_area ON dashboard_leads_view(area);
CREATE INDEX idx_dashboard_provider ON dashboard_leads_view(provider);
CREATE INDEX idx_dashboard_provider_count ON dashboard_leads_view(provider_count);

-- Full-text search on provider lists
CREATE INDEX idx_dashboard_all_providers 
ON dashboard_leads_view 
USING gin(to_tsvector('english', all_providers));
```

---

## Dashboard Features

### Key Metrics
- **Total Leads:** All lead records
- **Unique Leads:** Deduplicated phone numbers
- **Uniquely Sourced:** Leads from only one provider

### Visualizations
- **Bar Chart:** Leads by geographic area (zoom-enabled)
- **Pie Chart:** Lead distribution by provider (click-to-filter)
- **Data Table:** Full record view with search and sorting

**Interactive Filtering:**
- All charts cross-filter each other
- Click a provider → see their geographic distribution
- Click an area → see provider breakdown

---

## Access Control

### Admin Role
- Full dashboard editing
- Dataset management
- SQL Lab access
- User administration

### Viewer Role
- Read-only dashboard access
- No chart editing
- No direct SQL queries
- Cannot see underlying data structure

---

## Deployment

Superset deployed via Docker Compose:

```bash
git clone https://github.com/apache/superset.git
docker compose -f docker-compose-non-dev.yml up -d
```

**Setup steps:**
1. Launch containerized Superset
2. Connect PostgreSQL as datasource
3. Register `dashboard_leads_view` as dataset
4. Build charts and dashboard
5. Configure role-based permissions

---

## Architecture Highlights

**Data Normalization**
- Star schema with fact table (`lead_sources`) and dimension tables (`leads`, `providers`)
- Enables flexible analysis without data duplication

**Query Optimization Strategy**
- Materialized view eliminates 3-table joins on every query
- Strategic indexing covers all common filter/sort operations
- GIN index enables fast text search on provider lists

**Scalability Considerations**
- Normalized schema supports growth (new providers, metadata)
- Materialized view can be refreshed on schedule or on-demand
- Superset caching layer reduces database load

**Role-Based Security**
- Row-level security can be added via Superset's RLS feature
- Dataset permissions control access at table level
- Separate roles for analysts vs executives

---

## Future Improvements

- Add dedicated `areas` dimension table for geographic hierarchy
- Implement scheduled materialized view refresh (hourly/daily)
- Time-series analysis for lead volume trends

---

## License

This is a proprietary BI project. Shared for reference only — no copying or redistribution allowed.
