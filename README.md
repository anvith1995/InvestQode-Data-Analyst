  # Futures & Options (F&O) Analytics – 3NF DuckDB Data Model

## Overview
This project demonstrates how large-scale **Futures & Options (F&O)** market data (~2.5 million rows) can be transformed from raw, denormalized files into a **Third Normal Form (3NF)** relational schema using **DuckDB and SQL**.

The goal of this design is to:
- Eliminate redundancy
- Ensure data integrity
- Improve analytical query performance
- Enable scalable financial market analysis

---

## ER Diagram
The ER Diagram for this project was designed using **Lucidchart** and visually represents the normalized structure and relationships between entities.  
:contentReference[oaicite:0]{index=0}

![image alt](https://github.com/anvith1995/InvestQode-Data-Analyst/blob/1205924735a284ecc8fb4ece98b595e07517be79/ER%20Diagram.png)
---

## Data Model Overview (3NF)

The raw dataset was normalized into **four core tables**.

---

### 1. Exchange
Stores trading venues.

- Currently includes **NSE** data only
- Designed to support additional exchanges in the future

**Why this table exists**
- Prevents exchange duplication in transactional data
- Makes the schema extensible across markets

---

### 2. Instruments
Stores unique tradable instruments.

**Attributes**
- `symbol` (e.g., ACC, NIFTY, ADANIENT, BANKNIFTY)
- `instrument_type` (FUTSTK, OPTSTK, FUTIDX, OPTIDX)

Each row represents a **unique combination of symbol and instrument type**.

**Why this table exists**
- Central reference for all Futures & Options instruments
- Avoids repeated storage of symbol and instrument metadata

---

### 3. Contracts
Represents tradeable contracts derived from instruments.

A contract is uniquely defined by:
- `instrument_id`
- `expiry_date`
- `strike_price`
- `option_type`

**Primary Key**
- `contract_id`

**Relationship**
- One Instrument → Many Contracts

**Why this table exists**
- Futures and Options are contract-driven
- Separates contract attributes from daily trading activity

---

### 4. Fact_Trade
Stores daily trading activity at the **lowest analytical grain**.

**Grain**
- `contract_id`
- `exchange_id`
- `trade_date`

**Metrics**
- Open, High, Low, Close
- Volume
- Open Interest
- Change in Open Interest
- Turnover

**Why this table exists**
- Contains only measurable facts
- Enables fast aggregation and window-based analytics

---

## Data Integration Logic

Raw data is mapped into the normalized schema using a **composite join strategy** to avoid missing or mismatching contracts.

```sql
FROM raw_data r
JOIN instruments i
  ON r.SYMBOL = i.symbol
 AND r.INSTRUMENT = i.instrument_type
JOIN contracts c
  ON c.instrument_id = i.instrument_id
 AND STRPTIME(r.EXPIRY_DT, '%d-%b-%Y')::DATE = c.expiry_date
 AND COALESCE(r.STRIKE_PR, 0) = COALESCE(c.strike_price, 0)
 AND COALESCE(r.OPTION_TYP, 'NA') = COALESCE(c.option_type, 'NA');



---

### Optimization Objective
Identify the **highest trading volume day per symbol** over a rolling **30-day window** with minimal scan cost and maximum efficiency. Have added this SQL query shared

---

### Baseline Query Characteristics
- Large fact table (`fact_trade`)
- Time-based filtering (`trade_date`)
- Multiple joins across normalized tables
- Aggregation and ranking using window functions

Without optimization, the query requires a **full scan of the fact table**, increasing execution time.

---

### Optimization Strategy

#### 1. Indexing on High-Selectivity Columns
An index is created on the `trade_date` column in the `fact_trade` table:

```sql
CREATE INDEX idx_fact_trade_date
ON fact_trade(trade_date);
