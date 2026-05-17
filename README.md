# Rides & Bites: Analyzing User Trends and Peak-Time Demand With SQL
## Project Details
The objective of the analysis is to derive key insights into user behavior, peak-time patterns, and operational efficiency using advanced SQL techniques such as window functions, partitioning, and aggregations. By exploring these facets, the project aims to provide valuable business insights that can help optimize operational strategies, identify top-performing users and drivers, and improve service delivery.

The dataset includes information about rides (ride status, fare amounts, etc.) and food orders (order status, item details, delivery times, etc.). The analysis answers critical business questions, such as identifying peak demand hours, understanding user spending behaviors, and determining the most efficient drivers and restaurants.

## Project Objectives:

  * To explore user behavior by analyzing their spending patterns across rides and food deliveries.
  * To identify peak hours and days for both rides and food orders, enabling better resource allocation.
  * To evaluate the operational efficiency of drivers and restaurants through various metrics, including completed vs. canceled activities.
  * To derive business-relevant insights for operational and strategic decision-making.

## Technical Details:

This project leverages PostgreSQL to perform complex data manipulations and analyses. Advanced SQL techniques such as window functions, partitioning, and rolling window aggregations were used to answer business-relevant questions. The analysis is broken down into three main categories:

### User Behavior Analysis:
 * Spending Patterns: Ranking users by total spending across rides and food orders.
 * Frequent Order Categories: Identifying whether users prefer rides or food orders.
 * Average Spending: Calculating average spending per user and identifying top spenders.

### Peak-Time Analysis:
 * Peak Hours for Rides and Food: Identifying the busiest hours of the day for both rides and food orders.
 * Busiest Days: Analyzing which days of the week have the highest combined activity (rides + orders).
 * Average Processing Time: Calculating average order processing times during peak days.

 ### Operational Efficiency:
  * Driver Efficiency: Ranking drivers based on completed vs. canceled rides.
  * Restaurant Efficiency: Measuring the efficiency of restaurants based on completed vs. canceled orders.
  * Average Delivery Time: Calculating the average time between order placement and delivery for each restaurant.


### Business Insights Derived:

  #### User Spending Insights:
  *  Identifying high-value users and understanding their preferences between rides and food orders, enabling personalized offers or loyalty programs.
  #### Peak-Time Demand:
  *  Understanding peak hours and days for both rides and food deliveries, helping to optimize resource allocation and improve service efficiency.
  #### Operational Efficiency:
  *  Identifying drivers and restaurants with the highest efficiency ratios, which can inform performance reviews, training programs, and strategic partnerships.
    
### Challenges Faced:

#### Data Integrity:
 *  Ensuring that data was accurate and consistent across both the rides and food orders tables.
 #### Performance Optimization:
 * Writing efficient queries for large datasets and optimizing queries involving window functions and rolling window aggregations.
### Tables Create 
```sql
CREATE SCHEMA IF NOT EXISTS project1
    AUTHORIZATION postgres;
	
CREATE TABLE food_orders (
    order_id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    restaurant_id INT NOT NULL,
    order_date_time TIMESTAMP NOT NULL,
    order_status VARCHAR(50) CHECK (order_status IN ('Completed', 'Canceled', 'Refunded')) NOT NULL,
    total_price DECIMAL(10, 2) NOT NULL,
    delivery_fee DECIMAL(10, 2) NOT NULL
);
CREATE TABLE drivers (
    driver_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    rating DECIMAL(2, 1) CHECK (rating BETWEEN 1 AND 5),
    vehicle_type VARCHAR(50) CHECK (vehicle_type IN ('Car', 'Bike')) NOT NULL
);
CREATE TABLE rides (
    ride_id SERIAL PRIMARY KEY,
    driver_id INT NOT NULL,
    user_id INT NOT NULL,
    pickup_location VARCHAR(100) NOT NULL,
    dropoff_location VARCHAR(100) NOT NULL,
    distance_km DECIMAL(5, 2) NOT NULL,
    fare_amount DECIMAL(10, 2) NOT NULL,
    ride_status VARCHAR(50) CHECK (ride_status IN ('Completed', 'Canceled')) NOT NULL,
    ride_date_time TIMESTAMP NOT NULL
);
CREATE TABLE restaurants (
    restaurant_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    cuisine_type VARCHAR(50) NOT NULL,
    location VARCHAR(100) NOT NULL,
    rating DECIMAL(3, 1) CHECK (rating >= 0 AND rating <= 5) NOT NULL
);
```
## Basic Queries
### 1. Total Revenue by Restaurant
```sql
SELECT restaurant_id, 
       SUM(total_price + delivery_fee) AS total_revenue
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY restaurant_id;
```
### 2. Total Completed Rides for Each User
```sql
SELECT user_id, COUNT(*) AS completed_rides
FROM rides
WHERE ride_status = 'Completed'
GROUP BY user_id;
```
### 3. Average Order Value by User
```sql
SELECT user_id, 
       AVG(total_price + delivery_fee) AS avg_order_value
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY user_id;
```
### 4. Top 5 Users with the Most Completed Orders
```sql
SELECT user_id, COUNT(*) AS completed_orders
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY user_id
ORDER BY completed_orders DESC
LIMIT 5;
```
### 5. Users Who Have Made Both Food Orders and Completed Rides
```sql
SELECT DISTINCT food_orders.user_id
FROM food_orders
JOIN rides ON food_orders.user_id = rides.user_id
WHERE food_orders.order_status = 'Completed' 
  AND rides.ride_status = 'Completed';
```
### 6. Peak Order Times for Each Restaurant
```sql

SELECT restaurant_id, 
       EXTRACT(HOUR FROM order_date_time) AS order_hour, 
       COUNT(*) AS order_count
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY restaurant_id, order_hour
ORDER BY order_count DESC;
```
### 7. Top 3 Restaurants with the Highest Revenue in a Given Month
```sql
SELECT restaurant_id, 
       SUM(total_price + delivery_fee) AS total_revenue
FROM food_orders
WHERE order_status = 'Completed'
  AND EXTRACT(MONTH FROM order_date_time) = 1
  AND EXTRACT(YEAR FROM order_date_time) = 2025
GROUP BY restaurant_id
ORDER BY total_revenue DESC
LIMIT 3;
```
### 8. Order Frequency on Weekdays vs. Weekends
```sql
SELECT restaurant_id,
       CASE 
           WHEN EXTRACT(DOW FROM order_date_time) IN (0, 6) THEN 'Weekend'
           ELSE 'Weekday'
       END AS order_type,
       COUNT(*) AS order_count
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY restaurant_id, order_type;
```
### 9. Daily Revenue for Each Restaurant
```sql
SELECT restaurant_id, 
       DATE(order_date_time) AS order_date, 
       SUM(total_price + delivery_fee) AS daily_revenue
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY restaurant_id, order_date
ORDER BY order_date DESC;
```
### 10. Average Time Between Consecutive Orders for Each User
```sql
WITH order_times AS (
    SELECT user_id,
           EXTRACT(MINUTE FROM (order_date_time - LAG(order_date_time) OVER (PARTITION BY user_id ORDER BY order_date_time))) AS time_diff
    FROM food_orders
    WHERE order_status = 'Completed'
)
SELECT user_id, round(AVG(time_diff),1) AS avg_time_between_orders
FROM order_times
GROUP BY user_id;
```
### 11. Top 3 Users Who Have Spent the Most on Rides and Food Orders
```sql
SELECT rides.user_id, 
       SUM(fare_amount) + SUM(total_price + delivery_fee) AS total_spent
FROM rides
JOIN food_orders ON rides.user_id = food_orders.user_id
WHERE ride_status = 'Completed' AND order_status = 'Completed'
GROUP BY rides.user_id
ORDER BY total_spent DESC
LIMIT 3;
```
### 12. Users Who Have Placed Orders More Than 3 Times in a Week
```sql
SELECT user_id, 
       COUNT(*) AS order_count,
       EXTRACT(WEEK FROM order_date_time) AS order_week
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY user_id, order_week
HAVING COUNT(*) > 3;
```
### 13. Day of Week with the Highest Average Ride Spending
```sql
SELECT EXTRACT(DOW FROM ride_date_time) AS day_of_week,
       AVG(fare_amount) AS avg_ride_spending
FROM rides
WHERE ride_status = 'Completed'
GROUP BY day_of_week
ORDER BY avg_ride_spending DESC
LIMIT 1;
```
## Rides Dataset Queries
### 1. Retrieve the total revenue generated from completed rides.
```sql
SELECT SUM(fare_amount) AS total_revenue
FROM rides
WHERE ride_status = 'Completed';
```
### 2. Find the top 3 drivers with the highest number of rides.
```sql
SELECT driver_id, COUNT(*) AS total_rides
FROM rides
GROUP BY driver_id
ORDER BY total_rides DESC
LIMIT 3;
```
### 3. Calculate the average fare amount for rides longer than 5 kilometers.
```sql
SELECT AVG(fare_amount) AS avg_fare
FROM rides
WHERE distance_km > 5;
```
### 4. List the rides that were canceled and their total distance.
```sql
SELECT ride_id, distance_km
FROM rides
WHERE ride_status = 'Canceled';
```
### 5. Identify the driver with the highest total revenue.
```sql
SELECT driver_id, SUM(fare_amount) AS total_revenue
FROM rides
WHERE ride_status = 'Completed'
GROUP BY driver_id
ORDER BY total_revenue DESC
LIMIT 1;
```
### 6. Count the number of rides completed for each day in January 2025.
```sql
SELECT DATE(ride_date_time) AS ride_date, COUNT(*) AS completed_rides
FROM rides
WHERE ride_status = 'Completed'
GROUP BY DATE(ride_date_time)
ORDER BY ride_date;
```
### 7. Find the most frequent pickup and drop-off location pairs.
```sql
SELECT pickup_location, dropoff_location, COUNT(*) AS trip_count
FROM rides
GROUP BY pickup_location, dropoff_location
ORDER BY trip_count DESC
LIMIT 1;
```
### 8. List all drivers who have completed more than 10 rides.
```sql
SELECT driver_id, COUNT(*) AS ride_count
FROM rides
WHERE ride_status = 'Completed'
GROUP BY driver_id
HAVING COUNT(*) > 10;

```
### 9. Calculate the average driver rating grouped by vehicle type.
```sql
SELECT vehicle_type, AVG(rating) AS avg_rating
FROM drivers
GROUP BY vehicle_type;
```

### 10. Retrieve the top 3 longest rides in terms of distance.
```sql
SELECT ride_id, driver_id, user_id, distance_km
FROM rides
ORDER BY distance_km DESC
LIMIT 3;
```
### 11. Identify drivers with average ride fares above 300.
```sql
SELECT driver_id, AVG(fare_amount) AS avg_fare
FROM rides
WHERE ride_status = 'Completed'
GROUP BY driver_id
HAVING AVG(fare_amount) > 300;

```
### 12. Determine the percentage of canceled rides.
```sql
SELECT 
    (COUNT(*) FILTER (WHERE ride_status = 'Canceled') * 100.0 / COUNT(*)) AS canceled_percentage
FROM rides;
```
### 13. Find the driver who has traveled the most distance overall.
```sql
SELECT driver_id, SUM(distance_km) AS total_distance
FROM rides
GROUP BY driver_id
ORDER BY total_distance DESC
LIMIT 1;
```
### 14. Retrieve drivers whose rating is below the average rating of all drivers.
```sql
SELECT driver_id, name, rating
FROM drivers
WHERE rating < (SELECT AVG(rating) FROM drivers);
```
## Food Orders Dataset Queries
### 15. Calculate the total revenue generated from completed orders.
```sql
SELECT SUM(COALESCE(total_price, 0) + COALESCE(delivery_fee, 0)) AS total_revenue
FROM food_orders
WHERE order_status = 'Completed';

```
### 16. Find the top 5 users who placed the most orders.
```sql

SELECT user_id, COUNT(*) AS total_orders
FROM food_orders
GROUP BY user_id
ORDER BY total_orders DESC
LIMIT 5;

```
### 17. List the restaurants with the highest average order value.
```sql
SELECT restaurant_id, AVG(total_price) AS avg_order_value
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY restaurant_id
ORDER BY avg_order_value DESC
LIMIT 3;
```
### 18. Identify the user who paid the most delivery fees.
```sql
SELECT user_id, SUM(delivery_fee) AS total_delivery_fee
FROM food_orders
GROUP BY user_id
ORDER BY total_delivery_fee DESC
LIMIT 1;
```
### 19. Calculate the refund rate (percentage of refunded orders).
```sql
SELECT 
    (COUNT(*) FILTER (WHERE order_status = 'Refunded') * 100.0 / COUNT(*)) AS refund_rate
FROM food_orders;
```
### 20. Find the restaurant with the most canceled orders.
```sql
SELECT restaurant_id, COUNT(*) AS canceled_orders
FROM food_orders
WHERE order_status = 'Canceled'
GROUP BY restaurant_id
ORDER BY canceled_orders DESC
LIMIT 1;
```
### 21. Retrieve the most frequently ordered cuisine type.
```sql
SELECT r.cuisine_type, COUNT(o.order_id) AS order_count
FROM food_orders o
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
GROUP BY r.cuisine_type
ORDER BY order_count DESC
LIMIT 1;
```
### 22. Calculate the average total order price (including delivery fees).
```sql
SELECT AVG(total_price + delivery_fee) AS avg_order_price
FROM food_orders
WHERE order_status = 'Completed';
```
### 23. Identify users with total spending above 2000.
```sql
SELECT user_id, SUM(total_price + delivery_fee) AS total_spending
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY user_id
HAVING total_spending > 2000;
```
### 24. List the top 5 restaurants by total revenue.
```sql
SELECT r.restaurant_id, r.name, SUM(o.total_price + o.delivery_fee) AS total_revenue
FROM food_orders o
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE o.order_status = 'Completed'
GROUP BY r.restaurant_id, r.name
ORDER BY total_revenue DESC
LIMIT 5;
```
### 25. Find the average rating for restaurants grouped by cuisine type.
```sql
SELECT cuisine_type, AVG(rating) AS avg_rating
FROM restaurants
GROUP BY cuisine_type;
```

## Combined Queries Across Rides and Food Datasets
### 26. Calculate total revenue from both rides and food orders.
```sql
SELECT 
    (SELECT SUM(fare_amount) FROM rides WHERE ride_status = 'Completed') +
    (SELECT SUM(total_price + delivery_fee)
FROM food_orders
WHERE order_status = 'Completed') AS total_revenue;
```
### 27. Identify the top 3 users with the highest combined spending on rides and food orders.
```sql
SELECT user_id, SUM(total_spent) AS combined_spending
FROM (
    SELECT user_id, SUM(fare_amount) AS total_spent
    FROM rides
    WHERE ride_status = 'Completed'
    GROUP BY user_id
    UNION ALL
    SELECT user_id, SUM(total_price + delivery_fee) AS total_spent
    FROM food_orders
    WHERE order_status = 'Completed'
    GROUP BY user_id
) t
GROUP BY user_id
ORDER BY combined_spending DESC
LIMIT 3;
```
### 28. List users who have both taken rides and ordered food.
```sql
SELECT DISTINCT r.user_id
FROM rides r
JOIN food_orders o ON r.user_id = o.user_id;
```
### 29. Retrieve the restaurant and driver with the highest combined revenue.
```sql
SELECT 
    (SELECT driver_id 
     FROM rides 
     WHERE ride_status = 'Completed'
     GROUP BY driver_id 
     ORDER BY SUM(fare_amount) DESC 
     LIMIT 1) AS top_driver,
    (SELECT restaurant_id 
     FROM food_orders 
     WHERE order_status = 'Completed'
     GROUP BY restaurant_id 
     ORDER BY SUM(total_price + delivery_fee) DESC 
     LIMIT 1) AS top_restaurant;
```
## User Behavior Queries
### 1. Rank users by total spending (rides + orders) in descending order using window functions.
```sql
WITH user_spending AS (
    SELECT user_id, SUM(fare_amount) AS total_spent
    FROM rides
    WHERE ride_status = 'Completed'
    GROUP BY user_id
    UNION ALL
    SELECT user_id, SUM(total_price + delivery_fee) AS total_spent
    FROM food_orders
    WHERE order_status = 'Completed'
    GROUP BY user_id
)
SELECT user_id, total_spent,
       RANK() OVER (ORDER BY total_spent DESC) AS spending_rank
FROM user_spending;
```
### 2. Identify the average spending per user and find users who are top 20% highest spenders.
```sql
WITH user_avg_spending AS (
    SELECT rides.user_id, 
           AVG(rides.fare_amount) AS avg_ride_spend, 
           AVG(food_orders.total_price + food_orders.delivery_fee) AS avg_food_spend
    FROM rides
    LEFT JOIN food_orders ON rides.user_id = food_orders.user_id
    WHERE rides.ride_status = 'Completed' AND food_orders.order_status = 'Completed'
    GROUP BY rides.user_id
),
percentile_threshold AS (
    SELECT PERCENTILE_CONT(0.8) WITHIN GROUP (ORDER BY avg_ride_spend + avg_food_spend) AS threshold
    FROM user_avg_spending
)
SELECT user_avg_spending.user_id, 
       (avg_ride_spend + avg_food_spend) AS total_avg_spending
FROM user_avg_spending
JOIN percentile_threshold ON TRUE
WHERE (avg_ride_spend + avg_food_spend) > percentile_threshold.threshold;

```
### 3. Track the most frequent order category (e.g., rides or food) for each user.
```sql
WITH user_order_types AS (
    SELECT user_id, 'Ride' AS order_type, COUNT(*) AS order_count
    FROM rides
    WHERE ride_status = 'Completed'
    GROUP BY user_id
    UNION ALL
    SELECT user_id, 'Food' AS order_type, COUNT(*) AS order_count
    FROM food_orders
    WHERE order_status = 'Completed'
    GROUP BY user_id
),
ranked_orders AS (
    SELECT user_id, order_type, order_count,
           RANK() OVER (PARTITION BY user_id ORDER BY order_count DESC) AS frequency_rank
    FROM user_order_types
)
SELECT user_id, order_type, order_count, frequency_rank
FROM ranked_orders
WHERE frequency_rank = 1;

```
### 4. Calculate the cumulative total spending by users over time.
```sql
WITH cumulative_spending AS (
    SELECT rides.user_id, (fare_amount + total_price + delivery_fee) AS total_spending, ride_date_time
    FROM rides
    LEFT JOIN food_orders ON rides.user_id = food_orders.user_id
    WHERE ride_status = 'Completed' AND order_status = 'Completed'
)
SELECT user_id, ride_date_time, 
       SUM(total_spending) OVER (PARTITION BY user_id ORDER BY ride_date_time) AS cumulative_spending
FROM cumulative_spending;
```
## Peak-Time Analysis Queries
### 5. Find peak hours for rides by determining the hour with the most completed rides.
```sql
SELECT EXTRACT(HOUR FROM ride_date_time) AS hour_of_day, COUNT(*) AS completed_rides
FROM rides
WHERE ride_status = 'Completed'
GROUP BY hour_of_day
ORDER BY completed_rides DESC
LIMIT 1;
```
### 6. Calculate the average order value (food and delivery) per hour to analyze peak ordering times.
```sql
SELECT EXTRACT(HOUR FROM order_date_time) AS hour_of_day, 
       AVG(total_price + delivery_fee) AS avg_order_value
FROM food_orders
WHERE order_status = 'Completed'
GROUP BY hour_of_day
ORDER BY hour_of_day;
```
### 7. Find the busiest days for rides and food orders combined by counting completed activities (rides and orders) per day.
```sql
WITH combined_activity AS (
    SELECT DATE(ride_date_time) AS activity_date, COUNT(*) AS completed_rides, 0 AS completed_orders
    FROM rides
    WHERE ride_status = 'Completed'
    GROUP BY activity_date
    UNION ALL
    SELECT DATE(order_date_time) AS activity_date, 0 AS completed_rides, COUNT(*) AS completed_orders
    FROM food_orders
    WHERE order_status = 'Completed'
    GROUP BY activity_date
)
SELECT activity_date, SUM(completed_rides + completed_orders) AS total_activities
FROM combined_activity
GROUP BY activity_date
ORDER BY total_activities DESC
LIMIT 1;

```
### 8. Analyze the average time spent per ride during peak hours (top 3 busiest hours).
```sql
WITH peak_hours AS (
    SELECT EXTRACT(HOUR FROM ride_date_time) AS hour_of_day, COUNT(*) AS completed_rides
    FROM rides
    WHERE ride_status = 'Completed'
    GROUP BY hour_of_day
    ORDER BY completed_rides DESC
    LIMIT 3
)
SELECT EXTRACT(HOUR FROM ride_date_time) AS hour_of_day, AVG(EXTRACT(MINUTE FROM ride_date_time)) AS avg_minutes_per_ride
FROM rides
WHERE EXTRACT(HOUR FROM ride_date_time) IN (SELECT hour_of_day FROM peak_hours)
GROUP BY hour_of_day;
```
### 9. Find the average order processing time during peak days (top 3 busiest days).
```sql
WITH peak_days AS (
    SELECT DATE(order_date_time) AS activity_date, COUNT(*) AS completed_orders
    FROM food_orders
    WHERE order_status = 'Completed'
    GROUP BY activity_date
    ORDER BY completed_orders DESC
    LIMIT 3
),
order_times AS (
    SELECT user_id,
           DATE(order_date_time) AS activity_date,
           EXTRACT(MINUTE FROM order_date_time - LAG(order_date_time) OVER (PARTITION BY user_id ORDER BY order_date_time)) AS processing_time
    FROM food_orders
    WHERE DATE(order_date_time) IN (SELECT activity_date FROM peak_days)
)
SELECT activity_date, AVG(processing_time) AS avg_order_processing_time
FROM order_times
GROUP BY activity_date;

```
### 10. Track the number of rides and orders in each month (rolling window of 3 months).
```sql
SELECT activity_month, 
       SUM(ride_count) OVER (ORDER BY activity_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_rides,
       SUM(order_count) OVER (ORDER BY activity_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_orders
FROM (
    SELECT TO_CHAR(ride_date_time, 'YYYY-MM') AS activity_month, COUNT(*) AS ride_count, 0 AS order_count
    FROM rides
    WHERE ride_status = 'Completed'
    GROUP BY activity_month
    UNION ALL
    SELECT TO_CHAR(order_date_time, 'YYYY-MM') AS activity_month, 0 AS ride_count, COUNT(*) AS order_count
    FROM food_orders
    WHERE order_status = 'Completed'
    GROUP BY activity_month
) activity_data
ORDER BY activity_month;
```
## Operational Efficiency Queries
### 11. Determine the most efficient drivers based on the ratio of completed rides to canceled rides.
```sql
WITH driver_efficiency AS (
    SELECT driver_id, 
           COUNT(CASE WHEN ride_status = 'Completed' THEN 1 END) AS completed_rides,
           COUNT(CASE WHEN ride_status = 'Canceled' THEN 1 END) AS canceled_rides
    FROM rides
    GROUP BY driver_id
)
SELECT driver_id, 
       completed_rides / NULLIF(canceled_rides, 0) AS efficiency_ratio
FROM driver_efficiency
ORDER BY efficiency_ratio DESC
LIMIT 5;
```
### 12. Find the average time between order placement and delivery for each restaurant.
```sql
WITH order_times AS (
    SELECT restaurant_id,
           EXTRACT(MINUTE FROM (order_date_time - LAG(order_date_time) OVER (PARTITION BY restaurant_id ORDER BY order_date_time))) AS delivery_time
    FROM food_orders
    WHERE order_status = 'Completed'
)
SELECT restaurant_id, AVG(delivery_time) AS avg_delivery_time
FROM order_times
GROUP BY restaurant_id;
```
### 13. Analyze the number of orders completed and canceled by restaurant over a rolling 30-day window.
```sql
SELECT restaurant_id, activity_date,
       SUM(CASE WHEN order_status = 'Completed' THEN 1 ELSE 0 END) OVER (PARTITION BY restaurant_id ORDER BY activity_date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS completed_orders,
       SUM(CASE WHEN order_status = 'Canceled' THEN 1 ELSE 0 END) OVER (PARTITION BY restaurant_id ORDER BY activity_date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS canceled_orders
FROM (
    SELECT restaurant_id, DATE(order_date_time) AS activity_date, order_status
    FROM food_orders
) order_activity
ORDER BY restaurant_id, activity_date;
```
### 14. Find the top 3 most efficient restaurants based on the ratio of completed orders to canceled orders.
```sql
WITH restaurant_efficiency AS (
    SELECT restaurant_id,
           COUNT(CASE WHEN order_status = 'Completed' THEN 1 END) AS completed_orders,
           COUNT(CASE WHEN order_status = 'Canceled' THEN 1 END) AS canceled_orders
    FROM food_orders
    GROUP BY restaurant_id
)
SELECT restaurant_id, 
       completed_orders / NULLIF(canceled_orders, 0) AS efficiency_ratio
FROM restaurant_efficiency
ORDER BY efficiency_ratio DESC
LIMIT 3;

```
