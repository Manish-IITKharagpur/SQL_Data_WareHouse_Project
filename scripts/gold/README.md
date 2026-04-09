# 🏆 Gold Layer: The Analytical Data Model (Star Schema)

### 📌 Project Strategy
The Gold Layer is the final stage of the Medallion Architecture. Here, we transform our normalized Silver tables into a **Star Schema**—a specialized structure designed for high-performance reporting and ease of use for business users.

---

## 🌟 Understanding the Star Schema
The Star Schema consists of two types of tables:

1.  **Fact Tables**: The center of the star. These store quantitative measurements (e.g., Sales, Quantity) and foreign keys to dimensions.
2.  **Dimension Tables**: The "points" of the star. These provide descriptive context (Who, What, Where, When) for the numbers in the fact table.



**Why use a Star Schema?**
* **Performance**: Minimizes complex joins, making queries run significantly faster.
* **Usability**: Renames technical columns into "Friendly Names" (e.g., `cst_id` becomes `customer_id`).
* **Decoupling**: Uses **Surrogate Keys** (e.g., `customer_key`) to protect the data warehouse from changes in source system IDs.

---

## 🗂️ Dimension Table: dim_customer

### 1. Business Logic
The `dim_customer` table is the "Golden Record" of our customer base. It integrates three separate sources:
* **CRM (`crm_cust_info`)**: Primary source for names, status, and account creation dates.
* **ERP Demographics (`erp_cust_az12`)**: Secondary source for birthdates and gender validation.
* **ERP Location (`erp_loc_a101`)**: Source for geographical reporting.

### 2. Key Transformations
* **Surrogate Key Generation**: Implemented a system-generated `customer_key` using `ROW_NUMBER()`.
* **Gender Consolidation**: Applied a fallback logic—if the CRM gender is unknown ('n/a'), the model automatically pulls gender data from the ERP demographic records.
* **Full Join Integrity**: Utilized `LEFT JOIN` to ensure that every customer in our CRM is represented, even if supplementary ERP data is missing.

### 3. Data Dictionary
| Column Name | Type | Description |
| :--- | :--- | :--- |
| `customer_key` | INT | Surrogate Key (Primary Key for Gold Layer) |
| `customer_id` | INT | Original System ID from CRM |
| `gender` | VARCHAR | Consolidated gender from CRM and ERP sources |
| `country` | VARCHAR | Standardized country name for regional analysis |

---

## 🚀 Upcoming Gold Objects
* [ ] **dim_product**: Integrating product details and categories.
* [ ] **fact_sales**: The central table for financial metrics and KPIs.