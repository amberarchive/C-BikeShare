# Cyclistic-Bike-Share
Finding the hidden insight of data to increase customer commitment
By Amber Le.


## Overview
### About Cyclistic
Cyclistic is a bike-share company in Chicago. 

This document explores a dataset with information about individual rides covering Chicago within 1 year from March 2022 to Feb 2023.

Link to data on [BigQuery](https://console.cloud.google.com/bigquery?pli=1&project=bos-very-first-project&ws=!1m0).

### Disclaimer

Cyclistic is a fictional company. For the purposes of this case study, the datasets are appropriate and enable me to answer the business questions. 
The data has been made available by Motivate International Inc

[Data source](https://divvy-tripdata.s3.amazonaws.com/index.html)

[Lisence](https://ride.divvybikes.com/data-license-agreement)

### Business Task: 
The director of marketing believes the company’s future success depends on maximizing the number of annual memberships. Therefore, by understand how casual riders and annual members use Cyclistic bikes differently, Marketing team will build a strategy to convert casual riders into annual members.

Focusing on the objective, I am most interested in figuring out:
1. The pattern of the duration of the trip between the member and casual rider
2. The pattern of renting hour/date (day of the week, time of the day) between the member and casual rider 

The main features of interest would be the `started_at`, `ended_at` and `member_casual`

## Explorations & Findings: 
### Size & data type:
Looking at the Storage info, I easily see there are 5,663,942 rows, with 13 variables accordingly as below: 

| Field name    | Type |
| ------------- | ------------- |
| `ride_id`    | INTEGER  |
| `rideable_type` | STRING  |
| `started_at`    | DATETIME
| `ended_at`     | DATETIME
| `start_station_name`  | STRING
| `start_station_id`   | INTEGER
| `end_station_name` | STRING
| `end_station_id` | INTEGER
| `start_lat`| FLOAT
| `start_lng`  | FLOAT
| `end_lat` | FLOAT
| `end_lng` | FLOAT
| `member_casual` | STRING

All type of variables is right, so we don't need any step of transfer data type.

### Data Values:

Some variables have their own unique value (`ride_id`, `started_at`, `ended_at`), the other variables only fall into given values, such as the `member_casual` column, the type of user is only member or casual.

To find the value range of each field, I continue to use DISTINCT. It also finds if there are any errors such as typos, repeated spaces,...

```sql
SELECT  
   DISTINCT(member_casual)
FROM 
   `bos-very-first-project.cyclistic_bike_share.tripdata_raw_12months`
```
```sql
SELECT  
   DISTINCT(rideable_type)
FROM 
   `bos-very-first-project.cyclistic_bike_share.tripdata_raw_12months`
```

Findings: 
| Field name    | Description |
| ------------- | ------------- |
| `ride_id`    | ID attached to each ride taken  |
| `rideable_type` | Value falls into 3 types: electric_bike; classic_bike; docked_bike  |
| `started_at`    | day and time trip started, in CST
| `ended_at`     | day and time trip ended, in CST
| `start_station_name`  | name of station where trip originated
| `start_station_id`   | ID of station where trip originated
| `end_station_name` | name of station where trip terminated
| `end_station_id` | End Station ID
| `start_lat`| Start Station Latitude
| `start_lng`  | Start Station Longitude
| `end_lat` | End Station Latitude	
| `end_lng` | End Station Longitude
| `member_casual` | member or casual

### Data errors: 
#### Missing value: 

```sql
# count null value using IS NULL for each variable
SELECT 
  COUNT(*)
FROM `bos-very-first-project.cyclistic_bike_share.tripdata_raw_12months`
WHERE
  ride_id IS NULL
```
Finding: Lucky that the defined variables that relevant to my analysis are all integrity, there is no missing value found in columns `started_at`, `ended_at` and `member_casual`

| Field name    | Number of null | Percentage
| ------------- | ----------: |-------------:|
| `ride_id`   | 0  | 0%
| `rideable_type` | 0  |0%
| `started_at`    | 0  |0%
| `ended_at `     | 0 |0%
| `start_station_name` | 251567 | 4,44%
| `start_station_id`   | 251567 | 4,44%
| `end_station_name ` | 266440 |4,70%
| `end_station_id` | 266440 |4,71%
| `start_lat`| 0 |0%
| `start_lng ` | 0 |0%
| `end_lat` | 5666 |0,10%
| `end_lng `| 5666 |0,10%
| `member_casual` | 0 | 0%

#### Duplicate: 

```sql
# using DISTINCT to remove duplicate ride then using COUNT
SELECT
  COUNT(DISTINCT (ride_id))
FROM `bos-very-first-project.cyclistic_bike_share.tripdata_raw_12months`
```

Each ride has a unique ride_id, by using DISTINCT, I find out there are 5.429.084 rides in a total of 5.663.942 rides on the original dataset. The duplication may be caused by typo, copy or merge data,...

## Cleaning
Take 1: 
1. I only chose variables that are irrelevant to my analysis and drop the rest.
2. Remove duplicate: As my finding above,  each ride has a unique `ride_id`, using DISTINCT to remove duplicate
3. Converting data time zone: Cyclistic is Chicago based, but the columns `started_at` and `ended_at` are recorded by UTC +0 time zone. So I need to convert to the Chicago time zone, which is UTC -5. The 2 new column `adjusted_started_at` and `adjusted_ended_at` is the new time. 

Take 2:
Back to the business task, to find the pattern of the duration of the trip between the member and casual rider and also The pattern of renting hour/date (day of the week, time of the day) between the member and casual rider, first of all, I am going to: 
1. Calculate `duration_sec` - each ride's duration: using end time minus start time. Remind that I am using adjusted_started_at and adjusted_ended_at. This variable will be shown on the type number of seconds for each ride, for example a ride which lasts for 5 minutes and 32 seconds will be shown as (5*60+32) = 332 seconds
2. Extract `start_hour` - the hour that each ride started by extracting the hour in `adjusted_started_at` using the EXTRACT function.
3. Extract `day_no` - the day of the week that each ride started by extracting the day of the week in `adjusted_started_at` using EXTRACT function. The DAYOFWEEK trong BigQuery return the day in number, 1 equal to Sunday, 2 equal to Monday and so on, so I also need another step (in Take 3) to assign the name for each day. 

Take 3: 
- Assign name for time of day: I separated a day into 3 times: Morning is between 05:01 to 12:00; Afternoon is between 12:01 to 18:00, Night is between 18:01 to 05:00 
- Assign a name for the day of the week: As above, I just assign the name of the day of the week to accordingly number.

```sql
### TAKE 3
SELECT
  member_casual,
  duration_sec,
  start_hour,

  # assign name for day of week
  CASE
    WHEN day_no = 1 THEN "Sunday"
    WHEN day_no = 2 then "Monday"
    WHEN day_no = 3 then "Tuesday"
    WHEN day_no = 4 then "Wednesday"
    WHEN day_no = 5 then "Thursday"
    WHEN day_no = 6 then "Friday"
    ELSE "Saturday"
  END AS day_of_week,

  # assign name for time of day
  CASE
    WHEN start_hour > 5 AND start_hour <= 12 THEN "Morning"
    WHEN start_hour > 12 AND start_hour <=18 THEN "Afternoon"
    ELSE "Night"
  end as time_of_day

FROM
  ( ### TAKE 2
  SELECT  
    adjusted_started_at,
    adjusted_ended_at,
    member_casual,

    #calculate duration time in second
    abs ( extract( second from (adjusted_ended_at - adjusted_started_at) ) 
        + extract( minute from (adjusted_ended_at - adjusted_started_at) ) * 60 
        + extract( hour from (adjusted_ended_at - adjusted_started_at) ) * 60 * 60 ) AS duration_sec,

    # extract time of the day
    EXTRACT(HOUR FROM adjusted_started_at) as start_hour,

    # extract day of week
    EXTRACT(DAYOFWEEK FROM adjusted_started_at) AS day_no, --- Sunday is 1
  
  FROM 
  ( ### TAKE 1
    ## Clean data using DISTINCT, DATETIME 
SELECT
  # only chose variables that are rrelevant to my analysis
  DISTINCT (ride_id) # Each ride has a unique ride_id, using DISTINCT to remove duplicate ride
  started_at,
  ended_at,

  # adjust time zone from UTC +0 to Chicago -5
  DATETIME(started_at, "America/Chicago") as adjusted_started_at,
  DATETIME(ended_at, "America/Chicago") as adjusted_ended_at,

  member_casual
FROM `bos-very-first-project.cyclistic_bike_share.tripdata_raw_12months`
ORDER BY
  started_at
  )
  ORDER BY
    adjusted_started_at
  )
```
The cleaned data found at [here](https://console.cloud.google.com/bigquery?pli=1&project=bos-very-first-project&ws=!1m5!1m4!1m3!1sbos-very-first-project!2sbquxjob_1d9e6189_187028858b1!3sUS).

## Analizing

### Descriptive Statistics
| Variable | N | Mean | Max | Min | SD |
| ----- | ----- | ---- | ----- |----- |----- |
| `duration_sec` | 5.429.084 | 965,0313 | 86395 | 0 | 18903,6068 |
```sql
SELECT 
  MAX(duration_sec) as maximum,
  MIN(duration_sec) as minimum,
  AVG(duration_sec) as mean,
  STDDEV(DISTINCT duration_sec) as sd
FROM `bos-very-first-project.cyclistic_bike_share.cleaned`
```
