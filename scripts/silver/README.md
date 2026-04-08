# 🥈 Silver Layer: Data Cleansing & Transformation (The Quality Zone)

### 📌 Project Overview
The **Silver Layer** is where raw data from the Bronze Layer is refined into a "Golden Standard" for business intelligence. This stage is not just about moving data; it is about **Data Governance**. We ingest data from two primary sources—**CRM** and **ERP**—and align them into a unified relational model.

---

## 🔗 Data Integration & Relationships
To enable cross-system insights, the transformation logic ensures that disparate sources can be joined seamlessly.


* **Customer Integration:** `crm_cust_info` (CRM) joins with `erp_cust_az12` and `erp_loc_a101` (ERP) via a standardized Customer ID.
* **Product Integration:** `crm_prd_info` (CRM) links to `erp_px_cat_g1v2` (ERP) to provide categorical hierarchies.
* **Sales Context:** Transactional records in `crm_sales_details` are enriched by these customer and product dimensions.

---

## 🛠️ The Silver Transformation Framework
For every table in this layer, we follow a rigorous 3-step engineering process:

1.  **Analyze:** Detect "dirty" data, duplicates, and structural issues in the Bronze layer.
2.  **Transform:** Apply SQL functions (Windowing, Case logic, String manipulation) to fix the identified problems.
3.  **Verify:** Execute Quality Assurance (QA) scripts to ensure the table meets the target standard.

---

## 🗂️ Table 1: crm_cust_info (CRM System)

### 1. Analyze (Identified Problems)
* **Duplicates:** Multiple system entries for the same customer ID.
* **Noisy Strings:** Inconsistent leading and trailing spaces in name fields.
* **Coded Values:** Obscure abbreviations for Marital Status ('M', 'S') and Gender ('M', 'F').
* **Invalid Records:** Presence of `NULL` customer IDs which break relational integrity.

### 2. Transform (The SQL Solution)
* **Deduplication:** Utilized `RANK() OVER (PARTITION BY cst_id ORDER BY cst_create_date DESC)` to isolate only the most recent customer profile.
* **Standardization:** Implemented `CASE` statements to normalize codes into human-readable business terms ('Male', 'Female', 'Married', 'Single').
* **Sanitization:** Applied `TRIM()` to all string-based name fields.

### 3. Verify (QA Checks)
* **Uniqueness Check**: `SELECT cst_id, COUNT(*) FROM silver.crm_cust_info GROUP BY cst_id HAVING COUNT(*) > 1` (Target: 0).
* **Standardization Check**: Verified `DISTINCT cst_gndr` only returns 'Male', 'Female', or 'n/a'.
---
## 🗂️ Table 2: crm_prd_info (CRM System)

### 1. Analyze (Identified Problems)
* **Relationship Mismatches**: Identified that the first 5 characters of `prd_key` should map to `erp_px_cat_g1v2`, but some categories exist in CRM that are missing from the ERP reference table.
* **Integration Gaps**: Discovered certain products in the CRM have no corresponding sales records in `crm_sales_details`.
* **Complex Key Structures**: The `prd_key` is a composite string, making direct joins impossible without parsing.
* **Data Quality Gaps**: Found `NULL` or negative values in `prd_cost` and inconsistent spaces in `prd_nm`.
* **Timeline Integrity**: Found instances where `prd_start_dt` was greater than `prd_end_dt`.

### 2. Transform (The SQL Solution)
* **Key Decoupling**: Extracted `cat_id` using `REPLACE(SUBSTRING(prd_key, 1, 5),'-','_')` and parsed the actual `prd_key` starting from the 7th character.
* **Data Imputation**: Handled `NULL` costs by defaulting them to `0` using `ISNULL()`.
* **Normalization**: Standardized `prd_line` using `UPPER(TRIM())` and `CASE` logic.
* **Timeline Reconstruction (SCD Logic)**: Rebuilt the product history using the `LEAD()` window function to define `prd_end_dt` as the next record's `start_dt` minus 1 day.

### 3. Verify (QA Checks)
* **Join Readiness**: Checked for `prd_key` values missing from the Sales table to flag unsold inventory.
* **Cost Integrity**: Verified no negative or `NULL` costs remain.
* **Chronological Check**: Validated that `prd_start_dt` is always less than `prd_end_dt` post-transformation.
---
## 🗂️ Table 3: crm_sales_details (CRM System)

### 1. Analyze (Identified Problems)
* **Invalid Date Formats**: Order, Ship, and Due dates were stored as integers (e.g., 0 or incorrect lengths), preventing proper date calculations.
* **Calculation Inconsistencies**: Instances where `Sales` did not equal `Quantity * Price`, or values were negative/NULL.
* **Data Anomalies**: Negative or zero values for quantity and price, which are physically impossible in a standard sales transaction.

### 2. Transform (The SQL Solution)
* **Date Standardization**: Performed a double-cast (Integer ➡️ VARCHAR ➡️ DATE) after validating that the length is exactly 8 characters and non-zero.
* **Financial Data Recovery**:
    * **Sales**: If invalid, recalculated as `Quantity * ABS(Price)`.
    * **Price**: If invalid, derived using `Sales / NULLIF(Quantity, 0)` to prevent division-by-zero errors.
* **Consistency Enforcement**: Used `ABS()` on price to ensure all derived sales totals are positive and logically sound.

### 3. Verify (QA Checks)
* **Date Integrity**: Checked for any `sls_order_dt` values that remained as `NULL` or zero.
* **Arithmetic Validation**: Executed a check where `sls_sales != sls_quantity * sls_price` to confirm all transformations are mathematically perfect.
* **Value Range Check**: Verified `DISTINCT` values to ensure no zero or negative values exist for sales, price, or quantity.

---
## 🗂️ Table 4: erp_loc_a101 (ERP System)

### 1. Analyze (Identified Problems)
* **Key Inconsistency**: The `cid` (Customer ID) contained hyphens (e.g., 'CUST-123'), whereas the CRM system used clean IDs (e.g., 'CUST123'), preventing successful joins.
* **Coded Country Names**: Use of ISO-2 codes ('DE', 'US') and mixed formats ('USA'), which makes filtering difficult for end-users.
* **Data Gaps**: Presence of empty strings or `NULL` values in the country field.

### 2. Transform (The SQL Solution)
* **Key Alignment**: Used `REPLACE(cid, '-', '')` to strip hyphens and match the `cst_key` format used in the `silver.crm_cust_info` table.
* **Geographic Normalization**: Implemented a `CASE` statement to map 'DE' to 'Germany' and consolidate 'US'/'USA' into 'United States'.
* **Defaulting**: Categorized empty or `NULL` country values as 'n/a'.

### 3. Verify (QA Checks)
* **Referential Integrity Check**: Ran a `NOT IN` query to identify any ERP Customer IDs that do not exist in the CRM system (Target: High alignment).
* **Standardization Check**: Verified `DISTINCT cntry` to ensure all codes were successfully converted to full country names.
* **Format Check**: Confirmed no IDs in the Silver table contain hyphens.
---
### 🛠️ Technology Stack
* **Database:** SQL Server (T-SQL)
* **Framework:** Medallion Architecture
* **Tools:** SQL Server Management Studio (SSMS), GitHub
