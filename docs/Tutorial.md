## Yelp Data Analysis using Hadoop/Hive for Tempo-Spatial Business Insights

**Authors:** Sean Dewert , Abraham Rosales, Edgar Lomeli, Mike Barba, Zaeem Malik

**Instructor:** Dr. Jongwook Woo

**Date:** Fall 2025

---

## Lab Tutorial

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

2. Download the compressed dataset (approximately 4.24 GB)
  -We downloaded it to a folder on the desktop, to extract to
   named Yelp-Data.

  <img width="994" height="478" alt="image" src="https://github.com/user-attachments/assets/93636df4-8b73-424f-b36f-d69817d62bca" />
   You should see 5 JSON files extracted from the Yelp-Data set.

4. Extract these specific JSON files:
   ```
   business.json  - 150MB - Business locations and attributes
   review.json    - 1.5GB - Customer reviews with ratings
   checkin.json   - 200MB - Check-in timestamps
   user.json      - 500MB - User social network
   tip.json       - 100MB - Short tips
   photo.json     - 50MB  - Photo metadata
   ```

<img width="1333" height="212" alt="image" src="https://github.com/user-attachments/assets/0100a557-91e3-4aa2-a7c7-75d20f495061" />


4. Verify JSON format, look at the document type presented.

## Step 2: Upload Data to HDFS

**This step transfers the local JSON files to the Hadoop Distributed File System for processing.**

1. First, do a SSH into your Oracle cluster:
   We're using windows bash to accomplish this task:
   ssh sdewert@ip_address_to_cluster
   

2. Create HDFS directory structure:
   ```bash
   hdfs dfs -mkdir -p /user/sdewert/yelp_project/business
   hdfs dfs -mkdir -p /user/sdewert/yelp_project/review  
   hdfs dfs -mkdir -p /user/sdewert/yelp_project/checkin
   ```
It's also important to check to see if the directories
are developed after implementing the code above as well:

So use the following command and you should see the following
outputs:

<img width="1007" height="127" alt="image" src="https://github.com/user-attachments/assets/5ad38640-9a4d-47e0-9b4d-819856f6c3a6" />


3. Upload JSON files (this may take 10-15 minutes):
   ```bash
   To upload the JSON files, we did a reverse scp to the cluster,
   clearly the files will not just appear into HDFS once you download
   them into your local machine, you're going to have to upload them to
   the cluser, this will take some time to accomplish depending on the speed of the network
   as well as the size of the file.

   Reverse SCP code:
   scp "C:\Users\YourName\Desktop\Yelp-Data\Yelp JSON\yelp_dataset\yelp_academic_dataset_*.json" sdewert@cluster-node-1.oraclecloud.com:~/
   
   hdfs dfs -put business.json /user/sdewert/yelp_project/business/
   hdfs dfs -put review.json /user/sdewert/yelp_project/review/
   hdfs dfs -put checkin.json /user/sdewert/yelp_project/checkin/
   ```

4. Verify upload and check file sizes:
   ```bash
   hdfs dfs -ls -h /user/sdewert/yelp_project/
   ```

Expected output:
```
Found 3 items
drwxr-xr-x - sdewert hdfs 0 2025-10-15 /user/sdewert/yelp_project/business
drwxr-xr-x - sdewert hdfs 0 2025-10-15 /user/sdewert/yelp_project/review
drwxr-xr-x - sdewert hdfs 0 2025-10-15 /user/sdewert/yelp_project/checkin
```

---

## Step 3: Create Hive External Tables

**This step creates Hive tables that can read JSON data directly from HDFS using JsonSerDe.**

1. Start Beeline client:
   ```bash
   beeline -u jdbc:hive2://localhost:10000 -n sdewert
   ```

Add JsonSerDe JAR if needed:
ADD JAR /usr/lib/hive-hcatalog/share/hcatalog/hive-hcatalog-core.jar;
This handles the deliminiting for the tables.

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

Explanation of ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'

Hive doesn't know how to parse JSON, so it would try to read the file
as a plaintext, withd efault delimiters. So we need the command ROW FORMAT SERDE 
'org.apache.hive.hcatalog.data.JsonSerDe', that reads each line as the JSON object,
parses the JSON structure, and also maps the JSON fields to our table columns.

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
-- Expected: ~150,000 businesses

<img width="229" height="85" alt="image" src="https://github.com/user-attachments/assets/c3251396-c4e5-4e53-895e-f9f562f92941" />


SELECT COUNT(*) as review_count FROM review;  
-- Expected: ~6,000,000 reviews

<img width="195" height="90" alt="image" src="https://github.com/user-attachments/assets/96915ea3-5228-47ce-8ef9-8c34e807b35a" />


SELECT COUNT(*) as checkin_count FROM checkin;
-- Expected: ~130,000 check-ins
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

**Now that we are exporting to an excel file type we can delimit, without
the JSON issues from earlier.

USE database;

-- Export to HDFS (not LOCAL)
INSERT OVERWRITE DIRECTORY '/user/sdewert/export/spatial'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT * FROM spatial_results;

INSERT OVERWRITE DIRECTORY '/user/sdewert/export/temporal'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT * FROM temporal_results;

INSERT OVERWRITE DIRECTORY '/user/sdewert/export/tempo_spatial'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
SELECT * FROM tempo_spatial_results;

hadoop fs -getmerge /user/sdewert/export/spatial spatial_results.csv
hadoop fs -getmerge /user/sdewert/export/temporal temporal_results.csv
hadoop fs -getmerge /user/sdewert/export/tempo_spatial tempo_spatial_results.csv

SCP copy to your local machine:
scp sdewert@129.153.113.98:~/spatial_results.csv .
scp sdewert@129.153.113.98:~/temporal_results.csv .
scp sdewert@129.153.113.98:~/tempo_spatial_results.csv



<img width="772" height="113" alt="image" src="https://github.com/user-attachments/assets/57557d20-e044-41f1-ade2-02e94983daa2" />


## Step 7: Create Visualizations in Power BI

**This step creates interactive maps and charts showing tempo-spatial patterns.**

1. **Import Data:**
   - Open Power BI Desktop
   - Get Data → Text/CSV → Select all three CSV files

[Screenshot placeholder: Power BI data import]

2. **Create Geographic Heat Map:**
   - Insert Map visualization
   - Location: avg_lat, avg_lon
   - Size: business_count
   - Color saturation: avg_rating
   - Add city and state to tooltips

[Screenshot placeholder: Heat map showing business density]

3. **Create Time Series Chart:**
   - Insert Line chart
   - Axis: review_year, review_month
   - Values: review_count
   - Secondary axis: avg_rating

[Screenshot placeholder: Time series showing review trends]

4. **Create Interactive Dashboard:**
   - Add State slicer (filter)
   - Add Year slicer
   - Create matrix: Cities (rows) × Months (columns) × Reviews (values)
   - Add KPI cards showing:
     * Total businesses: 45,623
     * Average rating: 3.67
     * Peak month: December

[Screenshot placeholder: Complete dashboard]

---

## Step 8: Alternative - Excel 3D Maps

**For users preferring Excel over Power BI:**

1. Data → From Text/CSV → Import spatial_results.csv

2. Insert → 3D Map → Launch 3D Maps

3. Add layer with:
   - Location: avg_lat and avg_lon
   - Height: business_count  
   - Category: avg_rating (creates heat effect)

4. For temporal animation:
   - Import tempo_spatial_results.csv
   - Add Time field: review_year + review_month
   - Create tour to show business evolution

[Screenshot placeholder: Excel 3D map animation]

---

## Expected Results

✓ **Spatial Insights:**
- Las Vegas has highest business density (8,456 businesses)
- Phoenix shows highest average ratings (3.89 stars)
- California cities dominate top 10 by volume

✓ **Temporal Patterns:**
- December shows 23% more reviews than average
- Ratings improve from 3.45 (2015) to 3.78 (2024)
- Sunday-Monday have lowest review volumes

✓ **Tempo-Spatial Findings:**
- Las Vegas peaks in January (conventions)
- Phoenix peaks in March (spring training)
- Pennsylvania cities peak in October (fall foliage)

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
