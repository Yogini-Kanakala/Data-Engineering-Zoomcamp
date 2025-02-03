## Question 1: Understanding Docker First Run

**Task**: Run Docker with the `python:3.12.8` image in interactive mode and check the `pip` version.

**Steps**:
1. Pull the Docker image:
   ```bash
   docker pull python:3.12.8
   ```
2. Run the container:
   ```bash
   docker run -it --entrypoint bash python:3.12.8
   ```
3. Inside the container, check the `pip` version:
   ```bash
   pip --version
   ```

**Output**:
    ```bash
root@3e7cf660548f:/# pip --version
pip 24.3.1 from /usr/local/lib/python3.12/site-packages/pip (python 3.12)
    ```

**Answer**: The version of pip is `24.3.1`.



## Question 2: Docker Networking and Postgres Setup

### Task
Given the `docker-compose.yaml` file, determine the hostname and port for `pgadmin` to connect to the `postgres` database.

```yaml
services:
  db:
    container_name: postgres
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: 'postgres'
      POSTGRES_PASSWORD: 'postgres'
      POSTGRES_DB: 'ny_taxi'
    ports:
      - '5433:5432'
    volumes:
      - vol-pgdata:/var/lib/postgresql/data

  pgadmin:
    container_name: pgadmin
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: "pgadmin@pgadmin.com"
      PGADMIN_DEFAULT_PASSWORD: "pgadmin"
    ports:
      - "8080:80"
    volumes:
      - vol-pgadmin_data:/var/lib/pgadmin  

volumes:
  vol-pgdata:
    name: vol-pgdata
  vol-pgadmin_data:
    name: vol-pgadmin_data
```
In Docker Compose, all services in the same docker-compose.yaml are automatically part of the same network.
Services communicate internally using their service names as hostnames.
For postgres, the hostname is db and for the pgadmin, the hostname is pgadmin.

Inside the Docker Compose network, Postgres uses port 5432 

Ports defined in the ports section (5433:5432) map the container's internal port 5432 to the host machine's port 5433.

### Answer
- **Hostname**: `db`
- **Port**: `5432`

### Steps to Set Up Postgres and Load Data
1. Create and start services in detach mode:
   ```bash
   docker-compose up -d
   ```
2. Check for runnning docker containers
   ```bash
   docker ps
   ```

## Prepare Postgres

Run Postgres and load data  of green_tripdata
```wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-09.csv.gz```

The dataset with zones:

```wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv```


To load the dataset of green
```bash
python3 1_Containerization-and-Infrastructure-as-Code/homework/ingest_tripdata.py \
    --user=postgres \
    --password=postgres \
    --host=localhost \
    --port=5433 \
    --db=ny_taxi \
    --table_name=green_tripdata_2019-10 \
    --url="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-10.csv.gz"

```
To load the dataset of taxi_zones

```bash 
    python3 1_Containerization-and-Infrastructure-as-Code/homework/ingest_zones.py \
    --user=postgres \
    --password=postgres \
    --host=localhost \
    --port=5433 \
    --db=ny_taxi \
    --table_name=taxi_zone_lookup \
    --url="wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv"

```



## Question 3. Trip Segmentation Count

During the period of October 1st 2019 (inclusive) and November 1st 2019 (exclusive), how many trips, **respectively**, happened:
1. Up to 1 mile
2. In between 1 (exclusive) and 3 miles (inclusive),
3. In between 3 (exclusive) and 7 miles (inclusive),
4. In between 7 (exclusive) and 10 miles (inclusive),
5. Over 10 miles 

**Answers:** 104,802;  198,924;  109,603;  27,678;  35,189



### SQL Query
```sql
SELECT 
    SUM(CASE WHEN trip_distance <= 1 THEN 1 ELSE 0 END) AS "upto_1_mile",
    SUM(CASE WHEN trip_distance > 1 AND trip_distance <= 3 THEN 1 ELSE 0 END) AS "1_to_3_mile",
    SUM(CASE WHEN trip_distance > 3 AND trip_distance <= 7 THEN 1 ELSE 0 END) AS "3_to_7_miles",
    SUM(CASE WHEN trip_distance > 7 AND trip_distance <= 10 THEN 1 ELSE 0 END) AS "7_to_10 miles",
    SUM(CASE WHEN trip_distance > 10 THEN 1 ELSE 0 END) AS "morethen_10_miles"
FROM 
    public."green_tripdata_2019-10"
WHERE 
    lpep_pickup_datetime >= '2019-10-01' 
    AND lpep_pickup_datetime < '2019-11-01';
```

 104802	198924	109603	27678	35189




## Question 4: Longest Trip for Each Day

**Task**: Identify the pick-up day with the longest trip distance. 

### SQL Query to Find the Longest Trip for Each Day
```sql
   select lpep_pickup_datetime 
   from  public."green_tripdata_2019-10" 
   order by trip_distance desc
   limit 1;
```
**Answer** 2019-10-31

```bash  
"lpep_pickup_datetime"
"2019-10-31 23:23:41"
```





## Question 5. Three biggest pickup zones

Which were the top pickup locations with over 13,000 in
`total_amount` (across all trips) for 2019-10-18?

Consider only `lpep_pickup_datetime` when filtering by date.
 

### **SQL Query**
```sql
   SELECT  tz."Zone" , SUM(gtp.total_amount) AS total_sum
   FROM 
   public."green_tripdata_2019-10" as gtp inner join
   public."taxi_zone_lookup"  as tz 
   on gtp."PULocationID"=  tz."LocationID"
   where  date(gtp.lpep_pickup_datetime) ='2019-10-18'
   group by tz."Zone"
   having SUM(gtp.total_amount) > 13000
   order by total_sum desc
   limit 3;
```

"Zone"	"total_sum"
"East Harlem North"	18686.68000000008
"East Harlem South"	16797.260000000068
"Morningside Heights"	13029.790000000035

**Answer** - East Harlem North, East Harlem South, Morningside Heights




## Question 6. Largest tip

For the passengers picked up in October 2019 in the zone
named "East Harlem North" which was the drop off zone that had
the largest tip?

Note: it's `tip` , not `trip`

We need the name of the zone, not the ID.


- JFK Airport

```sql
   SELECT 
      tz_drop."Zone" AS dropoff_zone,
      MAX(gtp.tip_amount) AS largest_tip
   FROM 
      public."green_tripdata_2019-10" AS gtp
   inner join
      public."taxi_zone_lookup" AS tz_pickup 
      on gtp."PULocationID" = tz_pickup."LocationID"
   inner join
      public."taxi_zone_lookup" AS tz_drop 
      on gtp."DOLocationID" = tz_drop."LocationID"
   where 
      tz_pickup."Zone" = 'East Harlem North'
   group by tz_drop."Zone"
   order by  largest_tip desc
   LIMIT 1;

```
"dropoff_zone"	"largest_tip"
"JFK Airport"	87.3


**Answer**  - JFK Airport





## Question 7. Terraform Workflow

Which of the following sequences, **respectively**, describes the workflow for: 
1. Downloading the provider plugins and setting up backend,
2. Generating proposed changes and auto-executing the plan
3. Remove all resources managed by terraform`

**Answer**
- terraform init, terraform apply -auto-approve, terraform destroy



