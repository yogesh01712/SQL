# 📘 Slowly Changing Dimensions (SCD) Implementation Guide

This document provides detailed implementations of Slowly Changing Dimensions (SCD) Types 1, 2, and 3 using SQL, MERGE statements, and PySpark. 
It serves as a comprehensive reference for managing dimensional data changes in data warehousing.

📌 Use MERGE statement for atomic update/insert logic where supported (SQL Server, Snowflake, Delta).

	•	SCD Type 1 = Overwrite 
	•	SCD Type 2 = Insert new row + expire old
	•	SCD Type 3 = Add previous value columns

---

## 📌 SCD Type 1 — Overwrite on Change (No History)

### SQL Implementation

```sql
-- Update existing records with new data
UPDATE dim_customer
SET 
    FullName = s.FullName,
    Address = s.Address,
    City = s.City,
    Country = s.Country
FROM staging_customer s
WHERE dim_customer.Customer_ID = s.Customer_ID
  AND (
      dim_customer.FullName != s.FullName OR
      dim_customer.Address  != s.Address OR
      dim_customer.City     != s.City OR
      dim_customer.Country  != s.Country
  );

-- Insert new records
INSERT INTO dim_customer (Customer_ID, FullName, Address, City, Country)
SELECT 
    s.Customer_ID, s.FullName, s.Address, s.City, s.Country
FROM staging_customer s
LEFT JOIN dim_customer d ON s.Customer_ID = d.Customer_ID
WHERE d.Customer_ID IS NULL;
```

### Using MERGE  Statement

```sql
MERGE INTO dim_customer AS target
USING staging_customer AS source
ON target.Customer_ID = source.Customer_ID
WHEN MATCHED AND (
     target.FullName != source.FullName OR
     target.Address  != source.Address OR
     target.City     != source.City OR
     target.Country  != source.Country
)
THEN UPDATE SET
     FullName = source.FullName,
     Address  = source.Address,
     City     = source.City,
     Country  = source.Country
WHEN NOT MATCHED THEN
INSERT (Customer_ID, FullName, Address, City, Country)
VALUES (source.Customer_ID, source.FullName, source.Address, source.City, source.Country);
```

### PySpark 
```python

from pyspark.sql.functions import col

# Join staging and dimension data
joined_df = staging_df.alias("s").join(dim_df.alias("d"), on="Customer_ID", how="left")

# Identify records to update
updates_df = joined_df.filter(
    (col("d.Customer_ID").isNotNull()) &
    (
        (col("s.FullName") != col("d.FullName")) |
        (col("s.Address") != col("d.Address")) |
        (col("s.City") != col("d.City")) |
        (col("s.Country") != col("d.Country"))
    )
).select("s.*")

# Identify new records
new_records_df = joined_df.filter(col("d.Customer_ID").isNull()).select("s.*")

# Combine updates and new records
final_df = updates_df.unionByName(new_records_df)

# Write the final DataFrame to the target location
final_df.write.mode("overwrite").parquet("path_to_dim_customer")
```

---------

## 📌 SCD Type 2 — Track Full History with New Rows

SQL Implementation

```sql
-- Step 1: Expire old records
UPDATE dim_customer
SET End_Date = CURRENT_DATE,
    Active = 0
WHERE Customer_ID IN (
    SELECT s.Customer_ID
    FROM staging_customer s
    JOIN dim_customer d ON s.Customer_ID = d.Customer_ID
    WHERE d.Active = 1
      AND (
          s.FullName != d.FullName OR
          s.Address  != d.Address  OR
          s.City     != d.City     OR
          s.Country  != d.Country
      )
)
AND Active = 1;

-- Step 2: Insert new version for changed or new customers
INSERT INTO dim_customer (Customer_ID, FullName, Address, City, Country, Start_Date, End_Date, Active)
SELECT 
    s.Customer_ID,
    s.FullName,
    s.Address,
    s.City,
    s.Country,
    CURRENT_DATE AS Start_Date,
    NULL AS End_Date,
    1 AS Active
FROM staging_customer s
LEFT JOIN dim_customer d 
    ON s.Customer_ID = d.Customer_ID AND d.Active = 1
WHERE d.Customer_ID IS NULL
   OR (
        s.FullName != d.FullName OR
        s.Address  != d.Address  OR
        s.City     != d.City     OR
        s.Country  != d.Country
      );
```

### MERGE STATEMENT

```sql
MERGE INTO dim_customer AS target
USING staging_customer AS source
ON target.Customer_ID = source.Customer_ID AND target.Active = 1
WHEN MATCHED AND (
     target.FullName != source.FullName OR
     target.Address  != source.Address OR
     target.City     != source.City OR
     target.Country  != source.Country
)
THEN UPDATE SET
    End_Date = CURRENT_DATE,
    Active = 0
WHEN NOT MATCHED THEN
INSERT (Customer_ID, FullName, Address, City, Country, Start_Date, End_Date, Active)
VALUES (source.Customer_ID, source.FullName, source.Address, source.City, source.Country, CURRENT_DATE, NULL, 1);
```

## Merge Data from Staging Table

Finally, we use the MERGE statement to update the main employee table with changes from the staging table.

```sql

INSERT INTO employee (id, name, salary, start_date, end_date, is_active)
SELECT id, name, salary, start_date, end_date, is_active 
FROM (
    MERGE INTO employee AS target
    USING stg_employee AS source
    ON target.id = source.id AND target.is_active = 'Y'
    WHEN MATCHED THEN
        UPDATE SET 
            target.is_active = 'N',
            target.end_date = GETDATE()
    WHEN NOT MATCHED THEN
        INSERT (id, name, salary, start_date, end_date, is_active)
        VALUES (source.id, source.name, source.salary, GETDATE(), NULL, 'Y')
    OUTPUT $action,
        source.id,
        source.name,
        source.salary,
        GETDATE(),
        NULL,
        'Y'
) AS [changes] (action, id, name, salary, start_date, end_date, is_active)
WHERE action = 'UPDATE';
```

### PySpark Implementation
```python
from pyspark.sql.functions import current_date, lit

# Identify records to expire
records_to_expire = dim_df.alias("d").join(staging_df.alias("s"), on="Customer_ID") \
    .filter(
        (col("d.Active") == 1) &
        (
            (col("s.FullName") != col("d.FullName")) |
            (col("s.Address") != col("d.Address")) |
            (col("s.City") != col("d.City")) |
            (col("s.Country") != col("d.Country"))
        )
    ).select("d.*")

# Expire old records
expired_df = records_to_expire.withColumn("End_Date", current_date()).withColumn("Active", lit(0))

# Identify new records
new_records_df = staging_df.alias("s").join(dim_df.alias("d"), on="Customer_ID", how="left") \
    .filter(
        (col("d.Customer_ID").isNull()) |
        (
            (col("s.FullName") != col("d.FullName")) |
            (col("s.Address") != col("d.Address")) |
            (col("s.City") != col("d.City")) |
            (col("s.Country") != col("d.Country"))
        )
    ).select("s.*") \
    .withColumn("Start_Date", current_date()) \
    .withColumn("End_Date", lit(None).cast("date")) \
    .withColumn("Active", lit(1))

# Combine unchanged, expired, and new records
unchanged_df = dim_df.filter(col("Active") == 1).subtract(records_to_expire)
final_df = unchanged_df.unionByName(expired_df).unionByName(new_records_df)

# Write the final DataFrame to the target location
final_df.write.mode("overwrite").parquet("path_to_dim_customer")
```
