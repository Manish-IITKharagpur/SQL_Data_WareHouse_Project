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
---
### 🗂️ Data Dictionary

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| **customer_key** | `INT` | **Surrogate Key**: A system-generated unique identifier used as the Primary Key for the Gold layer. Decouples the warehouse from source system changes. |
| **customer_id** | `INT` | Unique numerical identifier assigned to each customer from the CRM system. |
| **customer_number** | `NVARCHAR(50)`| Alphanumeric identifier representing the customer, used for cross-system tracking and integration. |
| **first_name** | `NVARCHAR(50)`| The customer's first name, as recorded in the system. |
| **last_name** | `NVARCHAR(50)`| The customer's last name or family name. |
| **country** | `NVARCHAR(50)`| The standardized country of residence (e.g., 'Australia', 'Germany'). |
| **marital_status** | `NVARCHAR(50)`| The current marital status of the customer (e.g., 'Married', 'Single'). |
| **gender** | `NVARCHAR(50)`| The consolidated gender of the customer (e.g., 'Male', 'Female', 'n/a') using fallback logic. |
| **birthdate** | `DATE` | The date of birth of the customer, formatted as YYYY-MM-DD. |
| **create_date** | `DATE` | The date and time when the customer record was originally created in the system. |

---
## 🗂️ Dimension Table: dim_products

### 1. Business Logic
The `dim_products` table provides a unified view of the company's product catalog. It enriches basic CRM product data with detailed category hierarchies from the ERP system.

### 2. Key Transformations
* **Hierarchical Integration**: Merged CRM product records with ERP category and subcategory data to enable multi-level reporting.
* **SCD Support**: Included `start_date` and `end_date` to support historical analysis and Slowly Changing Dimensions (SCD).
* **Surrogate Key**: Generated `product_key` to ensure each unique state of a product is uniquely identified.

#### **Data Dictionary**

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| **product_key** | `INT` | **Surrogate Key**: Unique identifier used to uniquely identify each product record in the dimension table. |
| **product_id** | `INT` | A unique identifier assigned to the product for internal tracking and referencing. |
| **product_number** | `NVARCHAR(50)` | A structured alphanumeric code representing the product, often used for categorization or inventory. |
| **product_name** | `NVARCHAR(50)` | Descriptive name of the product, including key details such as type, color, and size. |
| **category_id** | `NVARCHAR(50)` | A unique identifier for the product's category, linking to its high-level classification. |
| **category** | `NVARCHAR(50)` | The broader classification of the product (e.g., 'Bikes', 'Components') to group related items. |
| **subcategory** | `NVARCHAR(50)` | A more detailed classification of the product within the category, such as product type. |
| **maintenance_required**| `NVARCHAR(50)` | Indicates whether the product requires maintenance (e.g., 'Yes', 'No'). |
| **cost** | `INT` | The cost or base price of the product, measured in monetary units. |
| **product_line** | `NVARCHAR(50)` | The specific product line or series to which the product belongs (e.g., 'Road', 'Mountain'). |
| **start_date** | `DATE` | The date when the product became available for sale or use. |

---

## 🗂️ Fact Table: facts_sales

### 1. Business Logic
The `facts_sales` table is the central hub of the analytical model. It records every sales transaction and links them to relevant business entities (Customers and Products).

### 2. Key Transformations
* **Key Mapping**: Replaced source system IDs with Gold-layer Surrogate Keys (`product_key`, `customer_key`) to ensure referential integrity.
* **Schema Alignment**: Standardized all column names to business-friendly terms for easier reporting.
* **Data Consolidation**: Brought together order dates, shipping dates, and financial metrics into a single, flat structure ready for aggregation.

### 3. Data Dictionary
| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| **order_number** | `NVARCHAR(50)` | The unique identifier for each sales order, used for tracking individual transactions. |
| **product_key** | `INT` | **Foreign Key**: Connects each sale to a specific record in `gold.dim_products` via its Surrogate Key. |
| **customer_key** | `INT` | **Foreign Key**: Connects each sale to a specific record in `gold.dim_customers` via its Surrogate Key. |
| **order_date** | `DATE` | The date the order was placed, used for time-series analysis and trend reporting. |
| **ship_date** | `DATE` | The date the order was shipped from the warehouse to the customer. |
| **due_date** | `DATE` | The date by which the order payment or delivery was expected. |
| **sales_amount** | `INT`| The total monetary value of the line item (Quantity × Price). |
| **quantity** | `INT` | The total number of units sold for this specific transaction. |
| **price** | `INT`| The unit price of the product at the time of the transaction. |
