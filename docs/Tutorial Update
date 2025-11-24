# CIS4560-02 Term Project Tutorial

**Authors:** Sean Dewert, Abraham Rosales, Edgar Lomeli, Mike Barba, Zaeem Malik

**Instructor:** Dr. Jongwook Woo

**Date:** Fall 2025

---

## Lab Tutorial For Team 3

### Objectives

In this hands-on lab, we will go over how to:
- Download and prepare 4+ GB Yelp Academic Dataset  
- Upload JSON data to HDFS on Oracle Cloud Infrastructure
- Create and clean Hive tables using JsonSerDe
- Perform tempo-spatial analysis combining location and time dimensions
- Export results and create interactive visualizations
- Generate business success metrics using composite scoring

### Platform Specifications

- **Cluster:** Oracle Cloud Infrastructure (5 nodes)
- **CPU Speed:** 2445.406 MHz (AMD EPYC 7763)
- **# of CPU cores:** 6
- **# of nodes:** 5 (2 Master, 3 Worker)  
- **Total Memory Size:** 31 GB
- **HDFS Capacity:** 390.17 GB

---

## Step 1: Download Yelp Academic Dataset

**This step obtains the 4+ GB dataset containing business, review, and check-in data spanning multiple years and locations.**

1. Navigate to https://www.yelp.com/dataset and complete the registration form

2. Download the compressed dataset (approximately 4.24 GB) - We downloaded it to a folder on the desktop, to extract to named Yelp-Data.

   <img width="994" height="478" alt="image" src="https://github.com/user-attachments/assets/93636df4-8b73-424f-b36f-d69817d62bca" />
   
   You should see 5 JSON files extracted from the Yelp-Data set.

3. Extract these specific JSON files:
   ```
   business.json  - 150MB - Business locations and attributes
   review.json    - 1.5GB - Customer reviews with ratings
   checkin.json   - 200MB - Check-in timestamps
   user.json      - 500MB - User social network
   tip.json       - 100MB - Short tips
   photo.json     - 50MB  - Photo metadata
   ```

4. Verify JSON format, look at the document type presented.

---

## Step 2: Upload Data to HDFS

**This step transfers the local JSON files to the Hadoop Distributed File System for processing.**

1. First, do a SSH into your Oracle cluster. We're using Windows bash to accomplish this task:
   ```bash
   ssh sdewert@ip_address_to_cluster
   ```

2. Create HDFS directory structure:
   ```bash
   hdfs dfs -mkdir -p /user/sdewert/yelp_project/business
   hdfs dfs -mkdir -p /user/sdewert/yelp_project/review  
   hdfs dfs -mkdir -p /user/sdewert/yelp_project/checkin
   ```

   The `-p` part of the code creates a parent directory automatically, making the yelp_project parent directory. We do not have to individually make it - we can make it during this process.

   It's also important to check to see if the directories are developed after implementing the code above. Use the following command and you should see the following outputs:

   ```bash
   hdfs dfs -ls /user/username/yelp_project/
   ```

   <img width="1007" height="127" alt="image" src="https://github.com/user-attachments/assets/5ad38640-9a4d-47e0-9b4d-819856f6c3a6" />

3. **Upload JSON files (this may take 10-15 minutes):**

   To upload the JSON files, we did a reverse scp to the cluster. Clearly the files will not just appear into HDFS once you download them into your local machine - you're going to have to upload them to the cluster. This will take some time to accomplish depending on the speed of the network as well as the size of the file.

   Reverse SCP code:
   ```bash
   scp "/c/Users/smdew/OneDrive/Desktop/Yelp-Data/Yelp JSON/"yelp_academic_dataset_*.json sdewert@129.153.113.98:~/
   ```

   The wildcard in the previous code will allow the download of all the following JSON files after.

   ```bash
   hdfs dfs -put business.json /user/sdewert/yelp_project/business/
   hdfs dfs -put review.json /user/sdewert/yelp_project/review/
   hdfs dfs -put checkin.json /user/sdewert/yelp_project/checkin/
   ```

   <img width="1333" height="212" alt="image" src="https://github.com/user-attachments/assets/0100a557-91e3-4aa2-a7c7-75d20f495061" />

4. Verify upload and check file sizes:
   ```bash
   hdfs dfs -ls -h /user/sdewert/yelp_project/
   ```

   Expected output:

   <img width="1007" height="127" alt="image" src="https://github.com/user-attachments/assets/5ad38640-9a4d-47e0-9b4d-819856f6c3a6" />

---

## Step 3: Create Hive External Tables

**This step creates Hive tables that can read JSON data directly from HDFS using JsonSerDe.**

1. Start Beeline client:
   ```bash
   beeline -u jdbc:hive2://localhost:10000 -n sdewert
   ```

2. Enter the following code FIRST, to allow for delimiting with JSON files:
   ```sql
   ADD JAR /usr/lib/hive-hcatalog/share/hcatalog/hive-hcatalog-core.jar;
   ```

   The code above will allow for parsing in JSON files, meaning being able to interpret braces, colons, and quotes in the files. This handles the delimiting for the tables.

3. Create the three main tables:

**Business Table:**
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS business (
    business_id STRING,
    name STRING,
    address STRING,
    city STRING,
    state STRING,
    postal_code STRING,
    latitude DOUBLE,
    longitude DOUBLE,
    stars DOUBLE,
    review_count INT,
    is_open INT,
    attributes STRING,  -- This is usually a nested JSON object
    categories STRING,
    hours STRING        -- Business hours, also often nested JSON
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/user/sdewert/yelp_project/business/';
```

**Explanation of ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe':**

Hive doesn't know how to parse JSON, so it would try to read the file as plaintext with default delimiters. So, we need the command `ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'`, which reads each line as the JSON object, parses the JSON structure, and maps the JSON fields to our table columns.

<img width="566" height="353" alt="image" src="https://github.com/user-attachments/assets/8dc17f96-b4a3-4bc5-9db6-11dea6a0f679" />

**Review Table:**
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS review (
    review_id STRING,
    user_id STRING,
    business_id STRING,
    stars INT,
    `date` STRING,
    text STRING,
    useful INT,
    funny INT,
    cool INT
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/user/sdewert/yelp_project/review/';
```

<img width="570" height="252" alt="image" src="https://github.com/user-attachments/assets/63401020-793a-4098-a0a0-fc79b060a434" />

**Check-in Table:**
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS checkin (
    business_id STRING,
    `date` STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/user/sdewert/yelp_project/checkin/';
```

<img width="560" height="115" alt="image" src="https://github.com/user-attachments/assets/fb14d698-0b3d-4940-b198-b33f42118b48" />

4. Verify record counts:

```sql
SELECT COUNT(*) as business_count FROM business;
```

<img width="229" height="85" alt="image" src="https://github.com/user-attachments/assets/c3251396-c4e5-4e53-895e-f9f562f92941" />

```sql
SELECT COUNT(*) as review_count FROM review;
```

<img width="195" height="90" alt="image" src="https://github.com/user-attachments/assets/96915ea3-5228-47ce-8ef9-8c34e807b35a" />

```sql
SELECT COUNT(*) as checkin_count FROM checkin;
```

<img width="202" height="91" alt="image" src="https://github.com/user-attachments/assets/c7952943-162a-4182-bd02-b63365ae0790" />

---

## Step 4: Data Cleaning and Preparation

**This step removes invalid data and extracts temporal features for analysis.**

**Clean Business Data (Spatial Focus):**
```sql
CREATE TABLE business_clean AS
SELECT 
    business_id,
    name,
    city,
    state,
    latitude,
    longitude,
    stars,
    review_count,
    categories
FROM business
WHERE 
    latitude IS NOT NULL 
    AND longitude IS NOT NULL
    AND latitude BETWEEN -90 AND 90
    AND longitude BETWEEN -180 AND 180
    AND state IN ('CA', 'AZ', 'NV', 'PA')  
    AND is_open = 1;
```

<img width="458" height="250" alt="image" src="https://github.com/user-attachments/assets/3cfa5a47-87fc-4840-87a7-327f8b1e36c1" />

**Clean Review Data (Temporal Focus):**
```sql
DROP TABLE IF EXISTS review_clean;
CREATE TABLE review_clean AS
SELECT 
    review_id,
    business_id,
    user_id,
    stars,
    `date` as review_date,
    YEAR(`date`) as review_year,
    MONTH(`date`) as review_month
FROM review
WHERE 
    `date` IS NOT NULL
    AND `date` >= '2010-01-01';
```

<img width="449" height="226" alt="image" src="https://github.com/user-attachments/assets/ca883a5d-6a19-4084-a1b0-d61e2811c53c" />

**Clean Check-in Data:**
```sql
DROP TABLE IF EXISTS checkin_clean;
CREATE TABLE checkin_clean AS
SELECT 
    business_id,
    `date` as checkin_dates_raw,
    SIZE(SPLIT(`date`, ',')) as checkin_count
FROM checkin
WHERE `date` IS NOT NULL;
```

<img width="511" height="138" alt="image" src="https://github.com/user-attachments/assets/4a95795d-8145-44fe-b308-5b8f5d0fb0c7" />

---

## Step 5: Tempo-Spatial Analysis Queries

**This step performs the core analysis combining spatial (location) and temporal (time) dimensions.**

**1. Spatial Analysis - Business Distribution by City:**
```sql
DROP TABLE IF EXISTS spatial_results;
CREATE TABLE spatial_results AS
SELECT city, state, 
       COUNT(*) as business_count,
       AVG(stars) as avg_rating,
       AVG(latitude) as avg_lat,
       AVG(longitude) as avg_lon
FROM business_clean
GROUP BY city, state
ORDER BY business_count DESC
LIMIT 50;
```

<img width="472" height="192" alt="image" src="https://github.com/user-attachments/assets/9384eb20-3561-4211-8b63-b2b3e7756387" />

**2. Temporal Analysis - Monthly Review Trends:**
```sql
DROP TABLE IF EXISTS temporal_results;
CREATE TABLE temporal_results AS
SELECT review_year, review_month,
       COUNT(*) as review_count,
       AVG(stars) as avg_rating
FROM review_clean
WHERE review_year >= 2015
GROUP BY review_year, review_month
ORDER BY review_year, review_month;
```

<img width="445" height="150" alt="image" src="https://github.com/user-attachments/assets/af71edaa-ea1d-48d0-9531-e7ac7307b771" />

**3. Combined Tempo-Spatial - Peak Times by Location:**
```sql
DROP TABLE IF EXISTS tempo_spatial_results;
CREATE TABLE tempo_spatial_results AS
SELECT b.city, b.state,
       r.review_year,
       r.review_month,
       COUNT(*) as reviews,
       AVG(r.stars) as avg_rating
FROM review_clean r
JOIN business_clean b ON r.business_id = b.business_id
GROUP BY b.city, b.state, r.review_year, r.review_month
ORDER BY reviews DESC
LIMIT 1000;
```

<img width="451" height="190" alt="image" src="https://github.com/user-attachments/assets/c5f6883b-5472-4ad1-9b7d-d3bb5a01c46f" />

---

## Step 6: Export Results to CSV

**This step exports analysis results for visualization in Power BI or Excel.**

**OBTAIN your JDBC line from BEELINE:**

JDBC stands for Java Database Connectivity - it's essentially a URL/address that tells beeline exactly how to connect to your Hive database.

**JDBC Line = Your Personal Connection Address to the Hive Database**

In BASH, type `beeline`, then look for your JDBC line as follows: each user has their own individualized JDBC line when logged into the beeline.

<img width="1887" height="43" alt="image" src="https://github.com/user-attachments/assets/cb1848bb-1e6f-42d9-90e0-6132fdc98ec0" />

**IMPORTANT: MUST RUN THESE AS A SINGLE LINE!**

1. In HDFS we will use the following command to convert the spatial_results into a .csv file:

```bash
beeline -u "jdbc:hive2://bigdaiun0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaimn0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaiwn0.sub03291929060.trainingvcn.oraclevcn.com:2181/default;password=sdewert;serviceDiscoveryMode=zooKeeper;user=sdewert;zooKeeperNamespace=hiveserver2" --outputformat=csv2 --showHeader=true -e "USE sdewert; SELECT * FROM spatial_results" > spatial_results.csv
```

Here are the results of the first 10 lines within the .csv file we developed to make sure it contained the information that was transferred to it:

```bash
head -10 spatial_results.csv
```

<img width="1843" height="106" alt="image" src="https://github.com/user-attachments/assets/6cf54b24-ac9a-4cd9-8de0-a1a6a7d17b4b" />

2. Now we will develop the temporal_results.csv file in the same way we did for the spatial_results.csv file as well:

```bash
beeline -u "jdbc:hive2://bigdaiun0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaimn0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaiwn0.sub03291929060.trainingvcn.oraclevcn.com:2181/default;password=sdewert;serviceDiscoveryMode=zooKeeper;user=sdewert;zooKeeperNamespace=hiveserver2" --outputformat=csv2 --showHeader=true -e "USE sdewert; SELECT * FROM temporal_results" > temporal_results.csv
```

3. Now for the tempo_spatial_results.csv creation, similar to the creation of the previous files:

```bash
beeline -u "jdbc:hive2://bigdaiun0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaimn0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaiwn0.sub03291929060.trainingvcn.oraclevcn.com:2181/default;password=sdewert;serviceDiscoveryMode=zooKeeper;user=sdewert;zooKeeperNamespace=hiveserver2" --outputformat=csv2 --showHeader=true -e "USE sdewert; SELECT * FROM tempo_spatial_results" > tempo_spatial_results.csv
```

**SCP copy to your local machine:**
```bash
scp sdewert@129.153.113.98:/home/sdewert/spatial_results.csv .
scp sdewert@129.153.113.98:/home/sdewert/temporal_results.csv .
scp sdewert@129.153.113.98:/home/sdewert/tempo_spatial_results.csv .
```

My files are downloaded into the following path but might be different for you.

---

## Step 7: Create Visualizations in Power BI

**This step creates interactive maps and charts showing tempo-spatial patterns.**

### 1. Import Data:
- Open Power BI Desktop
- Get Data → Text/CSV → Select all three CSV files

The Files uploaded are the results files only for the visualization process.

You do not need to transform the datasets - they should be ready to go. Once they are all loaded into Power BI, the right side should look like this:

<img width="590" height="59" alt="image" src="https://github.com/user-attachments/assets/0e2dbe2a-3b57-4f1c-bea3-649b4bb602a5" />

### 2. Create Geographic Bubble Map:

Insert Map visualization

- Location: spatial_results.city
- Legend: spatial_results.avg_rating
- Latitude: spatial_results.avg_lat
- Longitude: spatial_results.avg_lon
- Bubble Size: spatial_results.business_count
- Add city and state to tooltips

<img width="1855" height="823" alt="image" src="https://github.com/user-attachments/assets/cb72ff9f-1641-40a1-bc10-d9cbd8527647" />

**Table 1: Map Element Reference**

| Map Element | Data Field | What It Shows |
|-------------|------------|---------------|
| Bubble Location | city, avg_lat, avg_lon | Geographic location of business clusters |
| Bubble Size | Sum of business_count | Number of businesses in each city |
| Legend/Color | avg_rating | Average star rating by city |

**Table 2: Top Cities by Business Count**

| City | State | Business Count | Average Rating |
|------|-------|----------------|----------------|
| Philadelphia | PA | 10,540 | 3.65 |
| Tucson | AZ | 7,533 | 3.62 |
| Reno | NV | 4,759 | 3.79 |
| Santa Barbara | CA | 3,020 | 4.13 |

### 3. Create Time Series Chart:

Insert Line chart

**Configure Fields:**
- **X-axis**: Drag temporal_results.review_year, then drag temporal_results.review_month below it
- **Y-axis**: Drag temporal_results.review_count
- **Secondary Y-axis**: Drag temporal_results.avg_rating
- Click dropdown on avg_rating in Secondary Y-axis → Change from "Sum" to "Average"
- Click three dots (...) on chart → Sort axis → Select "review_year review_month" → Sort ascending

**Graph Element Reference:**

| Visual Element | Axis & Scale | Metric / Meaning |
|----------------|--------------|------------------|
| Light Blue Line | Left Axis (20,000 – 90,000) | Volume: The total number of reviews received each month |
| Dark Blue Line | Right Axis (3.60 – 3.90) | Sentiment: The average star rating (score) for that month |

### 4. Create Interactive Dashboard:

- Add State slicer (filter)
- Add Year slicer
- Create matrix: Cities (rows) × Months (columns) × Reviews (values)
- Add KPI cards showing:
  - Total businesses: 38K
  - Average rating: 3.62
  - Total reviews: 5M

**Dashboard Value Reference:**

| Value | Meaning / Description |
|-------|----------------------|
| 38K | Total business count across all cities in your spatial_results dataset |
| 3.62 | The average star rating across all businesses |
| 5M | Total reviews analyzed (approx. 5 million) spanning the years 2015–2022 |
| Matrix numbers | Specific review counts broken down by city and month (e.g., Philadelphia showing 64,737 reviews in July) |
| State totals | High-level volume summary by state (e.g., PA leads with 66,557 reviews in the filtered view) |

---

## Troubleshooting Common Issues

**JsonSerDe ClassNotFoundException:**
```sql
ADD JAR /usr/lib/hive-hcatalog/share/hcatalog/hive-hcatalog-core.jar;
```

**Out of Memory during JOIN:**
```sql
SET hive.auto.convert.join=false;
SET mapreduce.map.memory.mb=4096;
SET mapreduce.reduce.memory.mb=8192;
```

**Permission Denied on Export:**
```bash
chmod 755 ~/
hdfs dfs -chmod -R 755 /user/$(whoami)/
```

---

## References

1. Yelp Open Dataset: https://www.yelp.com/dataset
2. Apache Hive JsonSerDe: https://cwiki.apache.org/confluence/display/Hive/JsonSerDe
3. GitHub Repository: https://github.com/SeanCIS4560/yelp-data-analysis-cis4560-02
4. Power BI Documentation: https://docs.microsoft.com/en-us/power-bi/
