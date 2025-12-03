# Data-Modeling
A structured collection of core concepts, patterns, and best practices in modern data modeling and analytics architecture.

**🏗️ 1. Architecture Overview**


The project follows the Medallion Architecture


                +----------------+
                |    Source      |
                +----------------+
                         |
                         v
                +----------------+
                |    Bronze      |  (Raw → Incremental Load)
                +----------------+
                         |
                         v
                +----------------+
                |    Silver      |  (Cleaning + Standardization)
                +----------------+
                         |
                         v
                +----------------+
                |     Gold       |  (Dimensional Model + Facts + SCD)
                +----------------+



**📥 2. BRONZE LAYER – Ingestion & Incremental Load**


**File: Bronze.py**


Cited: 

Bronze

**✔ Responsibilities**

Read the source system (default.source_data).

Detect last load date dynamically.

Load only incremental rows into bronze_table


**✔ Key Logic**


if spark.catalog.tableExists('datamodeling.bronze.bronze_table'):
    last_load_date = spark.sql("select max(order_date) from datamodeling.bronze.bronze_table").collect()[0][0]
else:
    last_load_date = '1900-01-01'


select * 
from datamodeling.default.source_data 
where order_date > '{last_load_date}'


**📝 Output:**

datamodeling.bronze.bronze_table


**⚙️ 3. SILVER LAYER – Cleaning + Standardization**


File: Silver.py


Cited:Gold


**✔ Responsibilities**


Remove duplicates.

Standardize fields.

Add derived columns.


Example:
Customer_Name_Upper column


select 
    *, upper(customer_name) as Customer_Name_Upper
from datamodeling.bronze.bronze_table


Silver layer ensures the data is:
Cleaned → Deduped → Conformed


**🧱 4. GOLD LAYER – Dimensional Modeling (Star Schema)**

**_File: Gold.py_**

The Gold Layer converts the single large transactional table into the Star Schema:





              +------------------+
              |   DimCustomer    |
              +------------------+
                       |
                       |
 +--------------+      |     +---------------+
 | DimProduct   +------+-----+  DimRegion    |
 +--------------+      |     +---------------+
                       |
                       |
               +------------------+
               |    FactSales     |
               +------------------+



**🌟 5. Dimension Tables (with Surrogate Keys)**

**5.1 DimCustomers**


with rem_dup as (
  select distinct customer_id, customer_name, customer_email, Customer_Name_Upper 
  from datamodeling.silver.silver_table
)
select *, row_number() over(order by customer_id) as DimCustomerKey 
from rem_dup

**Result sample:**


| DimCustomerKey | customer_id | customer_name | email                                     |
| -------------- | ----------- | ------------- | ----------------------------------------- |
| 1              | CUST001     | Raj Kumar     | [r.k@gmail.com](mailto:r.k@gmail.com)     |
| 2              | CUST002     | Priya         | [priya@gmail.com](mailto:priya@gmail.com) |


**5.2 DimProducts**


with rem_dup as (
  select distinct product_id, product_name, product_category 
  from datamodeling.silver.silver_table
)
select *, row_number() over(order by product_id) as DimProductKey 
from rem_dup

**5.3 DimPayments**


with rem_dup as (
  select distinct payment_type 
  from datamodeling.silver.silver_table
)
select *, row_number() over(order by payment_type) as DimPaymentKey 
from rem_dup

**5.4 DimRegions**

with rem_dup as (
  select distinct country 
  from datamodeling.silver.silver_table
)
select *, row_number() over(order by country) as DimRegionKey 
from rem_dup

**5.5 DimSales**


with rem_dup as (
select distinct order_id, order_date, customer_id, product_id, payment_type, country
from datamodeling.silver.silver_table
)
select *, row_number() over(order by order_id) as DimSaleKey
from rem_dup

**📊 6. FACT TABLE Construction**


FactSales = Numeric metrics + All Dim Keys
select 
    c.DimCustomerKey,
    p.DimProductKey,
    r.DimRegionKey,
    py.DimPaymentKey,
    s.DimSaleKey,
    F.quantity,
    F.unit_price
from datamodeling.silver.silver_table F
left join datamodeling.gold.dimcustomers c on F.customer_id = c.customer_id
left join datamodeling.gold.DimProducts p on F.product_id = p.product_id
left join datamodeling.gold.DimRegions r on F.country = r.country
left join datamodeling.gold.DimPayments py on F.payment_type = py.payment_type 
left join datamodeling.gold.DimSales s on F.order_id = s.order_id

**🔄 7. SCD IMPLEMENTATION**


File: SCDs.py
Cited: 

Bronze

**✔ SCD Type 1 (Overwrite History)**


Used when corrections (customer name updated, email typo fix).

merge into dimcustomers tgt
using updates src
on tgt.customer_id = src.customer_id
when matched then update *

**✔ SCD Type 2 (Track History with Versioning)**


Used when we need to maintain full historical accuracy.

Typical fields:

start_date

end_date

is_current

version

when matched and tgt.is_current = true 
and tgt.customer_name <> src.customer_name
then update set end_date = current_date, is_current = false

when not matched then 
insert (cols..., start_date, end_date, is_current)
values (..., current_date, null, true)

**🧪 8. Data Quality Rules**


Check	Layer	Explanation
Duplicate removal	Silver	Ensures no duplicate dimension keys
Null handling	Silver	Removes null customer details
Surrogate keys	Gold	Enables joins without natural key issues
Incremental load	Bronze	Ensures pipeline idempotency


**🎯 9. Why This Design Is Production-Ready**


Requirement	How We Cover It
Idempotency	Bronze incremental load logic
Schema evolution	Bronze→Silver schema explicitly defined
Performance	Partitioning + surrogate keys
Data quality	Deduplication + validation
SCD handling	Type 1 & 2 patterns implemented
Analytics ready	Star Schema model


**💬 10. Prepare for Interview (Key Talking Points)**


If they ask “Why star schema?”

Faster joins

Better BI performance

Supports dimensional modeling

Compatible with SCDs

If they ask “Why use surrogate keys?”

Natural keys change

Joins become faster

SCD tracking requires surrogate keys

If they ask “Why bronze–silver–gold?”

Bronze = raw & traceable

Silver = cleaned, conformed

Gold = analytics optimized

If they ask “Where to implement data quality checks?”

Silver layer (nulls, duplicates)

Gold layer (FK validation)

**🖼️ 11. Final Diagram – End-to-End System**


                                 +-----------------------+
                                 |       Source Data      |
                                 +-----------+-----------+
                                             |
                                             v
                     +-------------------------------------------------+
                     |                    BRONZE                       |
                     | Incremental Load, Raw Ingestion                 |
                     +----------------------+--------------------------+
                                            |
                                            v
                     +-------------------------------------------------+
                     |                     SILVER                      |
                     | Cleaning, Deduping, Standardization             |
                     +----------------------+--------------------------+
                                            |
                                            v
               +----------------------- GOLD -----------------------------+
               |   DimCustomers   |   DimProducts   |   DimPayments       |
               |   DimRegions     |   DimSales      |                     |
               +----------------------------------------------------------+
                                            |
                                            v
                                +------------------------+
                                |      FactSales         |
                                +------------------------+




