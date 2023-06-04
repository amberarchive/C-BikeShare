# Cyclistic-Bike-Share
Finding the hidden insight of data to increase customer commitment
By Amber Le.


## Overview
### About Cyclistic
Cyclistic is a bike-share company in Chicago. 

This document explores a dataset with information about individual rides covering Chicago within 1 year from March 2022 to Feb 2023.

Link to data on [BigQuery](console.cloud.google.com/bigquery?ws=!1m4!1m3!3m2!1samberlearchive!2sCBikeShare).

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

The other explorations can be related to:

3. `rideable_type`, which type of bike that casual riders prefer

The main features of interest would be: `rideable_type`, `started_at`, `ended_at`, `member_casual`

## Explorations & Findings: 
### Size & data type:
There are 5,829,084 rows, with 13 variables accordingly as below: 

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

### Data Values:

Some variables have their own unique value (`ride_id`, `started_at`, `ended_at`), the other variables only fall into given values, such as the `member_casual` column, the type of user is only member or casual.

To find the value range of each field, I continue to use DISTINCT. It also finds if there are any errors such as typos, repeated spaces,...

```sql
SELECT  
   DISTINCT(member_casual)
FROM 
   `amberlearchive.CBikeShare.tripsdata`
```
```sql
SELECT  
   DISTINCT(rideable_type)
FROM 
   `amberlearchive.CBikeShare.tripsdata`
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
FROM  `amberlearchive.CBikeShare.tripsdata`
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
| `start_station_name` | 344511 | 5.9102081905%
| `start_station_id`   | 344643 | 5.9124726972%
| `end_station_name ` | 366592 |6.2890155640%
| `end_station_id` | 366733 |6.2914344689%
| `start_lat`| 0 |0%
| `start_lng ` | 0 |0%
| `end_lat` | 5938 |0.1018684925%
| `end_lng `| 5938 |0.1018684925%
| `member_casual` | 0 | 0%

#### Duplicate: 

```sql
# using DISTINCT to remove duplicate ride then using COUNT
SELECT
  COUNT(DISTINCT (ride_id))
FROM   `amberlearchive.CBikeShare.tripsdata`
```

Each ride has a unique ride_id, by using DISTINCT, I find out there are 5,829,084 rides in a total of 5,829,084 rides on the original dataset, no duplicate.

## Cleaning
Take 1: 
1. I only chose variables that are irrelevant to my analysis and drop the rest.
2. Remove duplicate: using DISTINCT to remove duplicate in column `ride_id`
3. Converting data time zone: Cyclistic is Chicago based, but the columns `started_at` and `ended_at` are recorded by UTC +0 time zone. So I convert to the Chicago time zone, which is UTC -5. The 2 new columns `adjusted_started_at` and `adjusted_ended_at` are the new time. 

Take 2:
Back to the business task, to find the pattern of the duration of the trip between the member and casual rider and also The pattern of renting hour/date (day of the week, time of the day) between the member and casual rider, first of all, I am going to: 
1. Calculate `duration_sec` - each ride's duration: using end time minus start time. Remind that I am using adjusted_started_at and adjusted_ended_at. This variable will be shown on the type number of seconds for each ride, for example a ride which lasts for 5 minutes and 32 seconds will be shown as (5*60+32) = 332 seconds
2. Extract `start_hour` - the hour that each ride started by extracting the hour in `adjusted_started_at` using the EXTRACT function.
3. Extract `day_no` - the day of the week that each ride started by extracting the day of the week in `adjusted_started_at` using EXTRACT function. The DAYOFWEEK in BigQuery returns the day in number, 1 equal to Sunday, 2 equal to Monday and so on, so I also need another step (in Take 3) to assign the name for each day. 

Take 3: 
- Assign name for time of day: Separated a day into 3 times: Morning is between 05:01 to 12:00; Afternoon is between 12:01 to 18:00, Night is between 18:01 to 05:00 
- Assign a name for the day of the week: As above, I just assign the name of the day of the week to accordingly number.

```sql
### TAKE 3
SELECT
  member_casual,
  duration_sec,
  start_hour,

  CASE
    WHEN day_no = 1 THEN "Sunday"
    WHEN day_no = 2 then "Monday"
    WHEN day_no = 3 then "Tuesday"
    WHEN day_no = 4 then "Wednesday"
    WHEN day_no = 5 then "Thursday"
    WHEN day_no = 6 then "Friday"
    ELSE "Saturday"
  END AS day_of_week,

  CASE
    WHEN start_hour > 5 AND start_hour <= 12 THEN "Morning"
    WHEN start_hour > 12 AND start_hour <=18 THEN "Afternoon"
    ELSE "Night"
  end as time_of_day,
 
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
SELECT
  DISTINCT (ride_id) # Each ride has a unique ride_id, using DISTINCT to remove duplicate ride
  started_at,
  ended_at,

  # adjust time zone from UTC +0 to Chicago -5
  DATETIME(started_at, "America/Chicago") as adjusted_started_at,
  DATETIME(ended_at, "America/Chicago") as adjusted_ended_at,
 
  member_casual
FROM `amberlearchive.CBikeShare.tripsdata`
ORDER BY
  started_at
  )
  ORDER BY
    adjusted_started_at
  )
```
The cleaned data found at [here](console.cloud.google.com/bigquery?ws=!1m5!1m4!4m3!1samberlearchive!2sCBikeShare!3stripsdata_cleaned).

## Analyzing

### Rides proportion: Member vs. Casual

![mc_pie1 (2)](https://github.com/amberarchive/C-BikeShare/assets/132808754/2fe43d63-7c25-4293-ac10-29d786175003)

3/5 of rides (about 59,57%) are taken by the member subscribers with the remaining 40,43% being casual riders. It doesn't mean that the market of casual riders to convert to member is not potential. 

A larger percentage of trips by member subscribers may be described as they are pay for their monthly fees, so they make efficient use of their subscriptions as much as possible. They may use our service as a main transportation - daily transportation. In contrast, casual riders sometimes use our service as a one-time solution: they miss their train, their car is broken or maintenance, or maybe the weather is good that day and they want a bike ride instead,... They priority some other transportation in their daily.

### Duration Trip

#### Descriptive Statistics
|index|duration\_sec|
|---|---|
|count|5829084\.0|
|mean|977\.3973651777878|
|std|1919\.4462659171945|
|25%|344\.0|
|50%|609\.0|
|75%|1094\.0|
|min|0.0|
|max|86395\.0|

The shortest trip is 0 second and the highest is 86395 seconds (equal to ~1440 hours or ~60 days). The standard deviation is 1919 which means the duration values are spread out over a very wide range. The average duration is 977 seconds (~16 minutes), Compared to min, max and the mode (which is 50% = 609), we can easily imagine the distribution of the duration trip will be highly skewed on the right with a really long tail.

Let's break it down into 2 separate user types according to the histogram below

![Duration Trip Distribution](https://github.com/amberarchive/C-BikeShare/assets/132808754/f5652fed-a236-481a-ab82-8980c1997c44)

The bin size is 60 which is equal to 1 minute.

The distribution is highly right-skewed with a really long tail. The histogram above limited the x-axis max to 150 (minute) to make it easier to observe.

The right-skewed is reasonable, our product is for urban moving only, so the duration trip should variaty between few minute to a few hours. There are trips that are too short or too long, it can explain that this is the user's error, they may depart or park the bike while the dock has not recorded the time used. 

![Line Distribution](https://github.com/amberarchive/C-BikeShare/assets/132808754/4c26527a-812d-43b2-8fec-d792fa203fb5)

True to the original hypothesis about the needs of 2 customer objects,
- The histogram of member riders is more right-skewed, they usually have a shorter travel time but higher volume due to the advantage of membership.
- In contrast, the histograms of customers using single rides were less right-skewed indicating that their trip duration tends to be longer. Furthermore, the slope of the histogram of casual riders is more gentle than the member's one.
- Initially, the member's histogram is much higher than the casual one. However, around the 24th bin (24th minute), the casual rider's histogram line crossed above the member's line.

<img width="358" alt="Cy Pricing" src="https://github.com/amberarchive/C-BikeShare/assets/132808754/33e9b180-edab-4da7-b747-c1f691c727e5">

### Day of the week
![day of week](https://github.com/amberarchive/C-BikeShare/assets/132808754/35f399dc-8fe0-45d2-b4b9-95b80ee5223e)

Saturday is the busiest day. As shown, the pattern of each user type is quite different. But let's separate it into 2 single graphs

![day of week by user type](https://github.com/amberarchive/C-BikeShare/assets/132808754/94c76f27-d129-4791-9341-c88cf15ae5a9)

Member subscribers tend to be more active on weekdays, the number of usage increases gradually from the beginning of the week and peaks on Tuesdays, Wednesdays and Thursdays, then continues to decrease on the last days. week.

Absolutely opposite, casual riders have a habit of using our service a lot on weekends and gradually decrease and the lowest is on weekdays: Monday, Tuesday, Wednesday and Thursday

### Time of the day
![222222](https://github.com/amberarchive/C-BikeShare/assets/132808754/360c8a96-6ba9-4ddc-a55b-964c8a81a419)
The x-axis has a max limit of 100% showing the percentage of rides for each time of day: about 44% rides in the morning, 45% rides in the afternoon, and about 11% rides at night.
It is easily explained that Morning and Afternoon are the time that people go to work and go home. After working hours, the demand of using the bike is not high.

The y-axis shows the percentage of rides of member and casual riders.

The percentage inside the body of the graph (pink and blue) is the percentage of rides of each rider type at each time of the day.

- The blue squares (from right to left): 
   - 17.75% rides were made by casual riders in the morning;
   - 19.18% rides were made by casual riders in the afternoon;
   - 3.30% is were made by casual riders in the afternoon;

Casual riders' ride in the morning is just a little bit lower (about 2%) than in the afternoon - the peak time, and then it dramatically drop-down at night  

- The pink squares (from right to left): 
   - 26.59% rides were made by casual riders in the morning;
   - 26.16%  rides were made by casual riders in the afternoon; 
   - 6.83%  rides were made by casual riders in the afternoon;  

Member user's peak time is morning, it just falls down insignificant in the afternoon and then dramatically drop-down at night.

It is not too different between member vs. casual riders on the time of day using the services. 
