# CIS4250-02 Term Project Tutorial

**Authors:** Sean Dewert, Abraham Rosales, Edgar Lomeli, Mike Barba, Zaeem Malik

**Instructor:** Dr. Jongwook Woo

**Date:** Fall 2025

---

## Lab Tutorial For Team 3

## Objectives

In this hands-on lab, we will go over how to:

- Download and prepare 4+ GB Yelp Academic Dataset
- Upload JSON data to HDFS on Oracle Cloud Infrastructure
- Create and clean Hive tables using JsonSerDe
- Perform tempo-spatial analysis combining location and time dimensions
- Export results and create interactive visualizations
- Generate business success metrics using composite scoring

---

## Platform Specifications

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

> ![Yelp Dataset Extraction](media/image1.png)
>
> You should see 5 JSON files extracted from the Yelp-Data set.

4. Extract these specific JSON files

- yelp_academic_dataset_business.json -- 116 MB -- Business locations and attributes
- yelp_academic_dataset_review.json -- 5.2 GB -- Customer reviews with ratings
- yelp_academic_dataset_checkin.json -- 280 MB -- Check-in timestamps
- yelp_academic_dataset_user.json -- 3.2 GB -- User social network
- yelp_academic_dataset_tip.json -- 176 MB -- Short tips

Verify JSON format, look at the document type presented.

---

## Step 2: Upload Data to HDFS

**This step transfers the local JSON files to the Hadoop Distributed File System for processing.**

1. First, do a SSH into your Oracle cluster: We're using windows bash to accomplish this task:

```bash
ssh sdewert@ip_address_to_cluster
```

2. Create HDFS directory structure:

```bash
hdfs dfs -mkdir -p /user/sdewert/yelp_project/business
hdfs dfs -mkdir -p /user/sdewert/yelp_project/review
hdfs dfs -mkdir -p /user/sdewert/yelp_project/checkin
```

Then `-p` part of the code create a parent directory automatically, making the yelp_project parent directory, we do not have to individually make it we can make it during this process.

It's also important to check to see if the directories are developed after implementing the code above as well:

So, use the following command and you should see the following outputs:

```bash
hdfs dfs -ls /user/username/yelp_project/
```

![HDFS Directory Listing](media/image2.png)

**Upload JSON files (this may take 10-15 minutes):**

To upload the JSON files, we did a reverse scp to the cluster, clearly the files will not just appear into HDFS once you download them into your local machine, you're going to have to upload them to the cluster, this will take some time to accomplish depending on the speed of the network as well as the size of the file.

Reverse SCP code:

```bash
scp "/c/Users/smdew/OneDrive/Desktop/Yelp-Data/Yelp JSON/"yelp_academic_dataset_*.json sdewert@129.153.113.98:~/
```

The wildcard in the previous code will allow the download of all the following JSON files after.

```bash
hdfs dfs -put yelp_academic_dataset_business.json /user/sdewert/yelp_project/business/
hdfs dfs -put yelp_academic_dataset_review.json /user/sdewert/yelp_project/review/
hdfs dfs -put yelp_academic_dataset_checkin.json /user/sdewert/yelp_project/checkin/
hdfs dfs -put yelp_academic_dataset_tip.json /user/sdewert/yelp_project/tip/
hdfs dfs -put yelp_academic_dataset_user.json /user/sdewert/yelp_project/user/
```

![HDFS Put Commands](media/image3.png)

4. Verify upload and check file sizes:

```bash
hdfs dfs -ls -h /user/sdewert/yelp_project/
```

Expected output:

![HDFS File Sizes](media/image4.png)

---

## Step 3: Create Hive External Tables

**This step creates Hive tables that can read JSON data directly from HDFS using JsonSerDe.**

1. Start Beeline client:

```bash
beeline -u jdbc:hive2://localhost:10000 -n sdewert
```

Enter the following code FIRST, to allow for delimiting with JSON files:

```sql
ADD JAR /usr/lib/hive-hcatalog/share/hcatalog/hive-hcatalog-core.jar;
```

The code above will allow for parsing in JSON files, meaning being able to interpret braces, colon, and quotes in the files.

This handles the delimiting for the tables.

### Business Table:

```sql
DROP TABLE IF EXISTS business;

CREATE EXTERNAL TABLE business (
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
    categories STRING -- Reverted to STRING to match your file
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/user/sdewert/yelp_project/business/';
```

Hive doesn't know how to parse JSON, so it would try to read the file as a plaintext, with default delimiters. So, we need the command `ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'`, that reads each line as the JSON object, parses the JSON structure, and maps the JSON fields to our table columns.

![Business Table Creation](media/image5.png)

### Review Table:

```sql
DROP TABLE IF EXISTS review;

CREATE EXTERNAL TABLE review (
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

![Review Table Creation](media/image6.png)

### Check-in Table:

```sql
DROP TABLE IF EXISTS checkin;

CREATE EXTERNAL TABLE checkin (
    business_id STRING,
    `date` STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/user/sdewert/yelp_project/checkin/';
```

### User_yelp Table:

```sql
DROP TABLE IF EXISTS user_yelp;

CREATE EXTERNAL TABLE user_yelp (
    user_id STRING,
    name STRING,
    review_count INT,
    yelping_since STRING,
    useful INT,
    funny INT,
    cool INT,
    elite ARRAY<STRING>,
    friends ARRAY<STRING>,
    fans INT,
    average_stars DOUBLE,
    compliment_hot INT,
    compliment_more INT,
    compliment_profile INT,
    compliment_cute INT,
    compliment_list INT,
    compliment_note INT,
    compliment_plain INT,
    compliment_cool INT,
    compliment_funny INT,
    compliment_writer INT,
    compliment_photos INT
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/user/sdewert/yelp_project/user/';
```

### Create Tip Table:

```sql
DROP TABLE IF EXISTS tip;

CREATE EXTERNAL TABLE tip (
    text STRING,
    `date` STRING,
    compliment_count INT,
    business_id STRING,
    user_id STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/user/sdewert/yelp_project/tip/';
```

![Tip Table Creation](media/image7.png)

4. Verify record counts:

```sql
SELECT COUNT(*) as business_count FROM business;
```

![Business Count](media/image8.png)

```sql
SELECT COUNT(*) as review_count FROM review;
```

![Review Count](media/image9.png)

```sql
SELECT COUNT(*) as checkin_count FROM checkin;
```

![Checkin Count](media/image10.png)

---

## Step 4: Data Cleaning and Preparation

**This step removes invalid data and extracts temporal features for analysis.**

### Clean Business Data (Spatial Focus):

1. Create a Business cleaned table:

```sql
DROP TABLE IF EXISTS business_clean;

CREATE TABLE business_clean AS
SELECT
    business_id,
    regexp_replace(name, ',', '') as name,
    regexp_replace(address, ',', '') as address,
    regexp_replace(city, ',', '') as city,
    state,
    postal_code,
    latitude,
    longitude,
    stars,
    review_count,
    is_open,
    categories
FROM business
WHERE latitude IS NOT NULL AND longitude IS NOT NULL;
```

![Business Clean Table](media/image11.png)

### Clean Review Data (Temporal Focus):

```sql
DROP TABLE IF EXISTS review_clean;

CREATE TABLE review_clean AS
SELECT
    review_id,
    business_id,
    user_id,
    stars,
    `date`,
    `date` as review_date, -- Creating the alias used in analysis
    regexp_replace(text, ',', '') as text, -- Kept (was dropped in tutorial)
    useful, -- Kept (was dropped in tutorial)
    funny, -- Kept (was dropped in tutorial)
    cool, -- Kept (was dropped in tutorial)
    YEAR(`date`) as review_year,
    MONTH(`date`) as review_month
FROM review
WHERE
    `date` IS NOT NULL
    AND `date` >= '2010-01-01';
```

![Review Clean Table](media/image12.png)

### Clean Check-in Data:

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

![Checkin Clean Table](media/image13.png)

### Clean the Tip Table:

```sql
DROP TABLE IF EXISTS tip_clean;

CREATE TABLE tip_clean AS
SELECT
    regexp_replace(text, ',', '') as text,
    `date`,
    `date` as tip_date,
    YEAR(`date`) as tip_year,
    MONTH(`date`) as tip_month,
    compliment_count,
    business_id,
    user_id
FROM tip
WHERE `date` IS NOT NULL;
```

### Clean the User Table:

```sql
DROP TABLE IF EXISTS user_clean;

CREATE TABLE user_clean AS
SELECT
    user_id,
    regexp_replace(name, ',', '') as name,
    review_count,
    yelping_since,
    YEAR(yelping_since) as join_year, -- Useful for "User Growth" charts
    useful,
    funny,
    cool,
    elite,
    friends,
    fans,
    average_stars,
    compliment_hot,
    compliment_more,
    compliment_profile,
    compliment_cute,
    compliment_list,
    compliment_note,
    compliment_plain,
    compliment_cool,
    compliment_funny,
    compliment_writer,
    compliment_photos
FROM user_yelp
WHERE user_id IS NOT NULL;
```

---

## Step 5: Tempo-Spatial and Analysis Queries

### 1. SPATIAL ANALYSIS ("High-Engagement Zones")

```sql
DROP TABLE IF EXISTS geo_patterns;

CREATE TABLE geo_patterns AS
SELECT
    regexp_replace(city, ',', '') as city,
    regexp_replace(state, ',', '') as state,
    AVG(latitude) as avg_lat,
    AVG(longitude) as avg_lon,
    COUNT(*) as engagement_volume,
    AVG(stars) as avg_quality
FROM business_clean
WHERE review_count > 0
GROUP BY city, state
ORDER BY engagement_volume DESC;
```

### 2. TEMPORAL ANALYSIS ("Seasonal Rush Hours")

```sql
DROP TABLE IF EXISTS seasonal_trends;

CREATE TABLE seasonal_trends AS
SELECT
    MONTH(to_date(review_date)) as season_month,
    COUNT(*) as seasonal_volume,
    AVG(stars) as seasonal_sentiment
FROM review_clean
WHERE review_date IS NOT NULL
GROUP BY MONTH(to_date(review_date))
ORDER BY season_month ASC;
```

### 3. ANIMATED MAP DATA

```sql
DROP TABLE IF EXISTS map_animated_success;

CREATE TABLE map_animated_success AS
SELECT
    regexp_replace(b.city, ',', '') as city,
    regexp_replace(b.state, ',', '') as state,
    b.latitude,
    b.longitude,
    r.review_date,
    r.review_month, -- <== ADDED THIS COLUMN
    r.stars
FROM review_clean r
JOIN business_clean b ON r.business_id = b.business_id;
```

### 4. CATEGORY ANALYSIS ("Top Business Types")

```sql
DROP TABLE IF EXISTS category_results;

CREATE TABLE category_results AS
SELECT
    TRIM(category) as category_name,
    COUNT(*) as business_count,
    AVG(stars) as avg_rating,
    SUM(review_count) as total_reviews
FROM business_clean
LATERAL VIEW EXPLODE(SPLIT(categories, ',')) table_alias AS category
GROUP BY TRIM(category)
ORDER BY business_count DESC;
```

### 5. EXPORT HELPER TABLE (Formats Map Data for CSV Export)

This table exists solely to bypass permission errors during export

```sql
DROP TABLE IF EXISTS export_map_data;

CREATE TABLE export_map_data
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/sdewert/export_map_data'
AS
SELECT
    city,
    state,
    latitude,
    longitude,
    review_date,
    review_month, -- <== CRITICAL ADDITION
    stars
FROM map_animated_success;
```

### Table for Sentiment Distribution

```sql
DROP TABLE IF EXISTS sentiment_dist;

CREATE TABLE sentiment_dist AS
SELECT
    stars as star_rating,
    COUNT(*) as count_dist
FROM review_clean
GROUP BY stars
ORDER BY stars DESC;
```

### Seasonal Categories Table

```sql
DROP TABLE IF EXISTS seasonal_categories;

CREATE TABLE seasonal_categories AS
SELECT
    r.review_month,
    TRIM(b_exploded.category_name) as category_name,
    COUNT(*) as seasonal_count
FROM review_clean r
JOIN (
    SELECT business_id, category as category_name
    FROM business_clean
    LATERAL VIEW EXPLODE(SPLIT(categories, ',')) sub_table AS category
) b_exploded ON r.business_id = b_exploded.business_id
-- Filter removed so ALL months are included
GROUP BY r.review_month, TRIM(b_exploded.category_name)
ORDER BY r.review_month, seasonal_count DESC;
```

---

## Step 6: Export Results to CSV

**This step exports analysis results for visualization in Power BI or Excel.**

OBTAIN your JDBC line from BEELINE:

**IMPORTANT: MUST RUN THESE AS A SINGLE LINE!**

### BEESHELL LINE COMMANDS FOR .CSV EXPORT

```bash
beeline -u "jdbc:hive2://bigdaiun0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaimn0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaiwn0.sub03291929060.trainingvcn.oraclevcn.com:2181/default;password=sdewert;serviceDiscoveryMode=zooKeeper;user=sdewert;zooKeeperNamespace=hiveserver2" --outputformat=csv2 --showHeader=true -e "USE sdewert; SELECT * FROM sentiment_dist" > ~/sentiment_dist.csv
```

```bash
beeline -u "jdbc:hive2://bigdaiun0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaimn0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaiwn0.sub03291929060.trainingvcn.oraclevcn.com:2181/default;password=sdewert;serviceDiscoveryMode=zooKeeper;user=sdewert;zooKeeperNamespace=hiveserver2" --outputformat=csv2 --showHeader=true -e "USE sdewert; SELECT * FROM category_results" > ~/category_results.csv
```

```bash
beeline -u "jdbc:hive2://bigdaiun0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaimn0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaiwn0.sub03291929060.trainingvcn.oraclevcn.com:2181/default;password=sdewert;serviceDiscoveryMode=zooKeeper;user=sdewert;zooKeeperNamespace=hiveserver2" --outputformat=csv2 --showHeader=true -e "USE sdewert; SELECT * FROM geo_patterns" > ~/geo_patterns_final.csv
```

```bash
beeline -u "jdbc:hive2://bigdaiun0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaimn0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaiwn0.sub03291929060.trainingvcn.oraclevcn.com:2181/default;password=sdewert;serviceDiscoveryMode=zooKeeper;user=sdewert;zooKeeperNamespace=hiveserver2" --outputformat=csv2 --showHeader=true -e "USE sdewert; SELECT * FROM seasonal_trends" > ~/seasonal_trends_final.csv
```

```bash
beeline -u "jdbc:hive2://bigdaiun0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaimn0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaiwn0.sub03291929060.trainingvcn.oraclevcn.com:2181/default;password=sdewert;serviceDiscoveryMode=zooKeeper;user=sdewert;zooKeeperNamespace=hiveserver2" --outputformat=csv2 --showHeader=true -e "USE sdewert; SELECT * FROM map_animated_success" > ~/map_data_final.csv
```

```bash
beeline -u "jdbc:hive2://bigdaiun0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaimn0.sub03291929060.trainingvcn.oraclevcn.com:2181,bigdaiwn0.sub03291929060.trainingvcn.oraclevcn.com:2181/default;password=sdewert;serviceDiscoveryMode=zooKeeper;user=sdewert;zooKeeperNamespace=hiveserver2" --outputformat=csv2 --showHeader=true -e "USE sdewert; SELECT * FROM seasonal_categories" > ~/seasonal_categories.csv
```

5. SCP copy to your local machine:

```bash
scp sdewert@129.153.113.98:/tmp/sentiment_dist.csv .
scp sdewert@129.153.113.98:/tmp/category_results.csv .
scp sdewert@129.153.113.98:/tmp/geo_patterns_final.csv .
scp sdewert@129.153.113.98:/tmp/seasonal_trends_final.csv .
scp sdewert@129.153.113.98:/tmp/map_data_final.csv .
scp sdewert@129.153.113.98:~/seasonal_categories.csv .
```

My files are downloaded into the following path but might be different for you:

---

## Step 7: Create Visualizations in Power BI

### 1. Create the 3D Map Visualization

Using map_data_final.csv → convert the .csv to .xlsx file first before doing the following operations.

**Open the Data in Excel:**

- Open a blank Excel workbook.
- Go to **Data** > **Get Data** > **From Text/CSV**.
- Select your **map_data_final.csv** file.
- Click **Load**.

**Launch 3D Maps:**

- Highlight all the data (Ctrl+A).
- Go to the **Insert** tab.
- Click on **3D Map** (sometimes labeled as **3D Maps** or **Open 3D Maps**).
- Select **New Tour**.

**Configure the Map Settings:**

A "Layer Pane" will appear on the right. Configure it using the settings defined in your Tutorial:

- **Location:**
  - Check the boxes for latitude and longitude.
  - Ensure Excel identifies them as "Latitude" and "Longitude" in the dropdowns.

- **Height (Volume):**
  - Drag the **rating** (or name) field into the **Height** box.
  - **Important:** Change the aggregation to **Count** (not Sum). This makes the bar height represent the *number of reviews* (Engagement Volume).

- **Category (Color):**
  - Drag the **rating** field into the **Category** box.
  - This will split the bars into different colors based on the star rating (1-5).

- **Time (Animation):**
  - Drag the **review_date** field into the **Time** box.
  - This enables the "Play" button at the bottom to show the data accumulating over time.

### 2. Creation for the Seasonal Trends Line Chart Process:

1. **Open** seasonal_trends_final.csv.
2. **Ctrl + A** to select all data.
3. **Insert > Line Chart**.
4. **Rename Title** to "Seasonal Trends".

### 3. Create a Heat Map for Comparisons between July and November:

Steps for creating the heat map comparisons for the project:

**Setup**

- Open map_data_final.csv.
- Go to **Insert > 3D Map > Open 3D Maps**.
- Click **Themes** (top bar) > Select **Dark Gray**.
- Change Chart Type to **Heat Map** (Gradient Icon).

**2. Configure Data**

- **Value:** Drag **stars** to the **Value** box.
- **Aggregation:** Change dropdown from "Sum" to **Count**.

**3. Capture July (Peak)**

- In Layer Pane, click **Filters > Add Filter**.
- Select **review_month**.
- Set slider/checkbox to **7**.
- Add Text Box: **July - Seasonal Engagement Density**.
- **Take Screenshot.**

**4. Capture November (Drop)**

- Change Filter to **11**.
- Change Text Box to: **November - Seasonal Engagement Density**.
- **Take Screenshot.**

![July Heat Map - Seasonal Engagement Density](media/image14.png)

July Map above:

November Map Below:

![November Heat Map - Seasonal Engagement Density](media/image15.png)

### Create the Pie Chart for Summer and Fall Sentiment:

**1. Setup Pivot Table**

- Open map_data_final.csv.
- **Insert > PivotTable**.
- **Rows:** Drag stars.
- **Values:** Drag stars (Change setting to **Count**).
- **Columns:** Drag review_month.

**2. Create July Chart (Figure 6A)**

- Click **Column Labels** dropdown (Cell B3).
- Uncheck (Select All), check **7**.
- **Insert > Pie Chart**.
- **Title:** July Sentiment (Summer Rush).
- **Color Foundation:**
  - 1 Star: **Red**
  - 2 Star: **Orange**
  - 3 Star: **Yellow**
  - 4 Star: **Light Green**
  - 5 Star: **Dark Green**
- **Capture Screenshot.**

**3. Create November Chart (Figure 6B)**

- Click **Column Labels** dropdown (Cell B3).
- Uncheck 7, check **11**.
- **Title:** November Sentiment (Seasonal Drop).
- **Color Foundation:** Verify colors match the list above (1=Red, 5=Green).
- **Capture Screenshot.**

![July Sentiment Pie Chart (Summer Rush)](media/image16.png)

Above July, Below November:

![November Sentiment Pie Chart (Seasonal Drop)](media/image17.png)

### Create a Categories Map for the Five Most Popular Business in July and November:

**Part 1: The Donut Charts (Seasonal Categories)**

Matches Tutorial Figure 8.A and 8.B

**1. Prepare the Data**

- **Source File:** Open seasonal_categories.csv in Excel.
- **Filter for July:**
  - Click the header row. Go to **Data > Filter**.
  - Filter the review_month column to show only **7** (July).
  - Sort the seasonal_count column from **Largest to Smallest**.
- **Select Data:** Highlight the top 5 Category Names and their Counts (e.g., Restaurants, Food, Nightlife, etc.).

**2. Create July Donut (Figure 8A)**

- **Insert Chart:** Go to **Insert** > **Pie Chart** icon > Select **Doughnut**.
- **Format:**
  - **Title:** Change chart title to **"TOP CATEGORIES: JULY"**.
  - **Labels:** Right-click the ring > **Add Data Labels**. Right-click labels > **Format Data Labels** > Check **Category Name** and **Percentage**.
- **Save:** Take a screenshot.

**3. Create November Donut (Figure 8B)**

- **Filter for November:** Change the review_month filter to **11** (November).
- **Select Data:** Highlight the top 5 Category Names and Counts.
- **Insert Chart:** Go to **Insert** > **Pie Chart** > **Doughnut**.
- **Format:**
  - **Title:** Change chart title to **"TOP CATEGORIES: NOVEMBER"**.
  - **Labels:** Add Category Name and Percentage.
- **Save:** Take a screenshot.

![Top Categories July Donut Chart](media/image18.png)

July above, November Below:

![Top Categories November Donut Chart](media/image19.png)

### Create Spatial Visualization for 5-Star and 1-Star Businesses:

- **Source File:** Open map_data_final.csv in Excel.
- **Launch:** Go to **Insert** > **3D Map** > **Open 3D Maps**.
- **Layer Settings (Layer Pane):**
  - **Location:** Check latitude and longitude.
  - **Height:** Drag stars into the **Height** box.
  - **CRITICAL STEP:** Click the arrow next to Sum of stars and change it to **Count**. *(This makes the bars represent volume/density).*
  - **Category:** Click **X** to remove any field from the Category box (to keep the color solid).
  - **Time:** Click **X** to remove any field from the Time box. *(Removing time aggregates all data to create the massive towers shown in the tutorial images).*

**2. Create High-Quality Map (Figure 9A)**

- **Filter:** Click **Filters** > Add Filter > Select stars.
  - Uncheck all boxes except **5**.
- **Color:** Go to **Layer Options** > Change color to **Green**.
- **Label:** Rename the Layer to **"High Quality (5-Star)"**.
- **Save:** Take a screenshot of the green clusters.

**3. Create Low-Quality Map (Figure 9B)**

- **Filter:** Change the stars filter to uncheck 5 and check **1**.
- **Color:** Go to **Layer Options** > Change color to **Red**.
- **Label:** Rename the Layer to **"Low Quality (1-Star)"**.
- **Save:** Take a screenshot of the red clusters.

![5-Star Business Spatial Visualization](media/image20.png)

5-Star Businesses Above and 1-Star Business Below:

![1-Star Business Spatial Visualization](media/image21.png)

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
