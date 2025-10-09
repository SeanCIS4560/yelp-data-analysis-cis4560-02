--Creation of the Checkin table

-- Create Checkin table
CREATE EXTERNAL TABLE IF NOT EXISTS checkin (
    business_id STRING,
    `date` STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/user/sdewert/yelp_project/checkin/';

-- Verify
SELECT COUNT(*) FROM checkin;
