

#### **1. Create an External Table in BigQuery**
We will create an **External Table** in BigQuery that directly references the Parquet files in **Google Cloud Storage (GCS)**.

1. **Go to BigQuery Console** 
2. **Select your project** and navigate to your dataset.
3. Click **Create dataset** and set the following configurations:

   - **Source:**
     - Format: `Parquet`
     - Select **Google Cloud Storage (GCS)**
     - **GCS Path**: `gs://dezoomcamp_hw3_2025/yellow_tripdata_2024-*`
     - Check **"Auto detect schema"**
   - **Destination:**
     - Dataset: `my_hwtaxi_dataset`
     - Table name: `external_yellow_taxi`
   - **Table Type:** `External Table`
   - Click **Create Table**



#### **2. Create a Materialized Table in BigQuery**

1. Run the  SQL query:

```sql
CREATE OR REPLACE TABLE my_hwtaxi_dataset.materialized_yellow_taxi AS
SELECT * FROM my_hwtaxi_dataset.external_yellow_taxi;
```

This will **load the data into BigQuery storage** instead of referencing GCS.


#### **3. Verify the Tables**
After creating both tables, check:

- **External Table:** Queries will **not** use BigQuery storage but read directly from GCS.
- **Materialized Table:** Queries will be **faster** since data is stored in BigQuery.


## Question 1:
Question 1: What is count of records for the 2024 Yellow Taxi Data?
- 65,623
- 840,402
- 20,332,093
- 85,431,289

**ANSWER:  "total_records": "20332093"**
```sql
SELECT COUNT(*) AS total_records
FROM my-project-kestra.yellow_taxi_data_2024.yellow_taxi_external;
```


## Question 2:
Write a query to count the distinct number of PULocationIDs for the entire dataset on both the tables.</br> 
What is the **estimated amount** of data that will be read when this query is executed on the External Table and the Table?

- 18.82 MB for the External Table and 47.60 MB for the Materialized Table
- 0 MB for the External Table and 155.12 MB for the Materialized Table
- 2.14 GB for the External Table and 0MB for the Materialized Table
- 0 MB for the External Table and 0MB for the Materialized Table


**ANSWER: 0 MB for the External Table and 155.12 MB for the Materialized Table**
```sql
select count (distinct PULocationID) from `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_materialized`;
```
//This query will process 155.12 MB when run.
```sql
select count (distinct PULocationID) from `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_external`; 
```
//This query will process 0 B when run.


## **Question 3:** Why Are Estimated Bytes Different?
Run the following queries separately:

```sql
SELECT PULocationID
FROM `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_materialized`;
```

```sql
SELECT PULocationID, DOLocationID
FROM `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_materialized`;
```

**answer:**  
**BigQuery is a columnar database, and it only scans the specific columns requested in the query. Querying two columns (PULocationID, DOLocationID) requires reading more data than querying one column (PULocationID), leading to a higher estimated number of bytes processed.**


## **Question 4:** Count Records with `fare_amount = 0`
Run this query:

```sql
select count (*) AS zero_fare_trips from `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_materialized` where fare_amount=0;
```

**Answer:**  
**8333**

## Question 5:
What is the best strategy to make an optimized table in Big Query if your query will always filter based on tpep_dropoff_datetime and order the results by VendorID (Create a new table with this strategy)
- Partition by tpep_dropoff_datetime and Cluster on VendorID
- Cluster on by tpep_dropoff_datetime and Cluster on VendorID
- Cluster on tpep_dropoff_datetime Partition by VendorID
- Partition by tpep_dropoff_datetime and Partition by VendorID

Since we will **filter by `tpep_dropoff_datetime`** and **order by `VendorID`**, the best strategy is:

**Answer: Partition by `tpep_dropoff_datetime` and Cluster on `VendorID`**  

To create the table with this strategy:

```sql

CREATE OR REPLACE TABLE `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_optimized`
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID AS
SELECT * FROM `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_materialized`;

```


## **Question 6:**

Write a query to retrieve the distinct VendorIDs between tpep_dropoff_datetime
2024-03-01 and 2024-03-15 (inclusive)</br>

Use the materialized table you created earlier in your from clause and note the estimated bytes. Now change the table in the from clause to the partitioned table you created for question 5 and note the estimated bytes processed. What are these values? </br>

Choose the answer which most closely matches.</br> 

- 12.47 MB for non-partitioned table and 326.42 MB for the partitioned table
- 310.24 MB for non-partitioned table and 26.84 MB for the partitioned table
- 5.87 MB for non-partitioned table and 0 MB for the partitioned table
- 310.31 MB for non-partitioned table and 285.64 MB for the partitioned table

**materialized table** This query will process 310.24 MB when run.
```sql
SELECT DISTINCT VendorID
FROM `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_materialized`
WHERE DATE(tpep_dropoff_datetime) BETWEEN '2024-03-01' AND '2024-03-15';
```

**partitioned table**:  This query will process 26.84 MB when run.
```sql
SELECT DISTINCT VendorID
from `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_optimized`
WHERE DATE(tpep_dropoff_datetime) BETWEEN '2024-03-01' AND '2024-03-15';
```

**Answer**
**310.24 MB for non-partitioned table and 26.84 MB for the partitioned table**



## Question 7: 
Where is the data stored in the External Table you created?

- Big Query
- Container Registry
- GCP Bucket
- Big Table

**Answer:** **GCP Bucket**  
(External tables reference files stored in **Google Cloud Storage**)


## Question 8:
It is best practice in Big Query to always cluster your data:
- True
- False

**Answer:** **False**  
(Clustering is useful, but not always required. It depends on query patterns.)

## (Bonus: Not worth points) Question 9:
No Points: Write a `SELECT count(*)` query FROM the materialized table you created. How many bytes does it estimate will be read? Why?

```sql
SELECT count(*)
FROM `my-project-kestra.yellow_taxi_data_2024.yellow_taxi_materialized`
```
**Answer: This query will process 0 B when run.**

BigQuery automatically optimizes COUNT(*) queries on materialized tables. It does this by:

Using metadata stored in the table to count rows without scanning the entire dataset.
Avoiding a full table scan, making the operation instant and free.