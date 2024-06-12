# Creating a K-means Model to Cluster London Bicycle Hires with BigQuery ML

In this blog post, I'll walk you through creating a k-means clustering model using Google Cloud's BigQuery ML. We'll use the London Bicycle Hires dataset to identify patterns and make data-driven decisions. Follow along as I detail each step, and feel free to refer to the screenshots I took during this process for a visual guide.

## Objectives
- Create a k-means clustering model.
- Analyze clusters to make data-driven decisions.

## Prerequisites
Before we begin, ensure you have:
1. A Google Cloud project with billing enabled.
2. BigQuery API activated.

## Step 1: Create a Dataset
First, we need to create a dataset in BigQuery to store our ML model.

1. Navigate to the BigQuery page in the Google Cloud Console.
2. In the Explorer pane, click your project name.
3. Click **View actions > Create dataset**.
4. Fill in the details:
   - **Dataset ID**: `bicycle_hires`
   - **Location**: Multi-region, select `EU (European Union)`

## Step 2: Examine Your Training Data
Next, we need to examine the data to train our model. We’ll be clustering bike stations based on the duration of rentals, number of trips per day, and distance from the city center.

Run the following GoogleSQL query to inspect the data:

```sql
#standardSQL
WITH
  hs AS (
    SELECT
      h.start_station_name AS station_name,
      IF(EXTRACT(DAYOFWEEK FROM h.start_date) = 1 OR EXTRACT(DAYOFWEEK FROM h.start_date) = 7, "weekend", "weekday") AS isweekday,
      h.duration,
      ST_DISTANCE(ST_GEOGPOINT(s.longitude, s.latitude), ST_GEOGPOINT(-0.1, 51.5))/1000 AS distance_from_city_center
    FROM
      `bigquery-public-data.london_bicycles.cycle_hire` AS h
    JOIN
      `bigquery-public-data.london_bicycles.cycle_stations` AS s
    ON
      h.start_station_id = s.id
    WHERE
      h.start_date BETWEEN TIMESTAMP '2015-01-01 00:00:00' AND TIMESTAMP '2016-01-01 00:00:00'
  ),
  stationstats AS (
    SELECT
      station_name,
      isweekday,
      AVG(duration) AS avg_duration,
      COUNT(duration) AS num_trips,
      MAX(distance_from_city_center) AS distance_from_city_center
    FROM
      hs
    GROUP BY
      station_name, isweekday
  )
SELECT
  *
FROM
  stationstats
ORDER BY
  distance_from_city_center ASC;
```

This query calculates the average duration of rentals, the number of trips, and the distance of each station from the city center.

## Step 3: Create a K-means Model
Now, let's create the k-means model using BigQuery ML.

Run the following SQL command:

```sql
CREATE OR REPLACE MODEL `bicycle_hires.kmeans_model`
OPTIONS(model_type='kmeans', num_clusters=5)
AS
SELECT
  station_name,
  avg_duration,
  num_trips,
  distance_from_city_center
FROM
  `your-project-id.bicycle_hires.stationstats`;
```

This command creates a k-means model clustering the data into five clusters based on the station attributes.

## Step 4: Predict Cluster Assignments
After creating the model, we can predict which cluster each station belongs to.

Execute this SQL command:

```sql
SELECT
  station_name,
  predicted_cluster
FROM
  ML.PREDICT(MODEL `bicycle_hires.kmeans_model`,
  (SELECT station_name, avg_duration, num_trips, distance_from_city_center FROM `your-project-id.bicycle_hires.stationstats`));
```

This will give us a list of stations with their corresponding clusters.

## Step 5: Make Data-Driven Decisions
With the clusters identified, we can make informed decisions. For instance, we can determine which bike stations might need additional bikes or better maintenance.

## Adding Visuals
To help you visualize the process, I’ve taken screenshots at each step. These images will guide you through the console navigation and show the results of each query.

---

By following these steps, you can leverage BigQuery ML to create k-means models for clustering your data, enabling better decision-making through data-driven insights. Stay tuned for more tutorials on advanced data analytics and machine learning with Google Cloud!
