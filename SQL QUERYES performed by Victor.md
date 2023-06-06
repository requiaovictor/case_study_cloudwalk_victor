SQL QUERYES
CREATED BY VICTOR REICHENBACH REQUIÃO - REQUIAOVICTOR@GMAIL.COM
CLOUDWALK CASE STUDY

## CREATING cloudwalk TABLE IN MY SQL SERVER(POSTGRES)

CREATE TABLE cloudwalk (
    id SERIAL,
    status VARCHAR(100),
    provider VARCHAR(100),
    state VARCHAR(100),
    city VARCHAR(100),
    sales_order_created_at TIMESTAMP,
    device_order_created_at TIMESTAMP,
    processing_at TIMESTAMP,
    in_transit_to_local_distribution_at TIMESTAMP,
    local_distribution_at TIMESTAMP,
    in_transit_to_deliver_at TIMESTAMP,
    delivered_at TIMESTAMP,
    delivery_estimate_date TIMESTAMP,
    supply_name VARCHAR(100),
    shipment_cost NUMERIC
)

## INSERTED INFORMATION O MY TABLE USING THE 'INSERTS' FILE PROVIDED

## CREATING NEW COLUMNS WITH INFORMATION (delivery_time, OTD and difference_otd, region)

ALTER TABLE cloudwalk
ADD delivery_time INT;

UPDATE cloudwalk
SET delivery_time = DATE_PART('day', delivered_at - device_order_created_at)::INTEGER;

ALTER TABLE cloudwalk
ADD otd VARCHAR(10);

UPDATE cloudwalk
SET otd = CASE WHEN delivered_at <= delivery_estimate_date THEN 'On Time' ELSE 'Delayed' END;

ALTER TABLE cloudwalk
ADD Difference_otd INT;

UPDATE cloudwalk
SET difference_otd = DATE_PART('day', delivered_at - delivery_estimate_date)::INTEGER;

ALTER TABLE cloudwalk
ADD COLUMN region VARCHAR(100);

UPDATE cloudwalk
SET region = 
  CASE 
    WHEN state IN ('BA', 'SE', 'AL', 'PE', 'PB', 'RN', 'CE', 'PI', 'MA') THEN 'Nordeste'
    WHEN state IN ('SP', 'RJ', 'ES', 'MG') THEN 'Sudeste'
    WHEN state IN ('RS', 'SC', 'PR') THEN 'Sul'
    WHEN state IN ('GO', 'MT', 'MS', 'DF') THEN 'Centro-Oeste'
    WHEN state IN ('AM', 'RR', 'AP', 'PA', 'TO', 'RO', 'AC') THEN 'Norte'
    ELSE 'Outro'
END;

## SUBQUERY TO CALCULATE THE % OF ON TIME DELIVERY(94,18%)

WITH ontime AS (
  SELECT otd, COUNT(otd) AS count_ontime
  FROM cloudwalk
  WHERE otd = 'On Time'
  GROUP BY otd
)
SELECT (count_ontime::numeric / 42710) * 100 AS percent
FROM ontime;

## OTD PER REGION AND PERCENTS

with count_otd_regio as (select region,count(otd) AS total_delivery_per_region from cloudwalk where status ILIKE 'delivered' group by region),
otd_ontime_per_region as (select region,count(otd) AS on_time_deliverys from cloudwalk where otd ILIKE 'on time' group by region)
select a.region,a.total_delivery_per_region,b.on_time_deliverys,
	(b.on_time_deliverys::numeric/a.total_delivery_per_region::numeric)*100 AS percent,
	AVG((b.on_time_deliverys::numeric/a.total_delivery_per_region::numeric)*100) OVER() AS media_total_percent
from count_otd_regio a INNER JOIN otd_ontime_per_region b ON  a.region = b.region


## CALCULATING OVERALL STATUS OF THE DATABASE(RETURNED, CANCELLED AND DELIVERED)

SELECT status, COUNT(*) as count, 
    ROUND(CAST(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS numeric), 2) AS percentage
FROM cloudwalk
GROUP BY status;

445 returned - 1.04%
45 cancelled - 0.11%
42220 delivered - 98.85%

## NUMBER OF ORDERS DELIVERED/CANCELLED/RETURNED PER PROVIDER

SELECT provider,status,COUNT(status) 
FROM cloudwalk
group by provider,status;

## NUMBER OF ORDERS PER REGION AND STATE

SELECT region, state, COUNT(*) as order_count
FROM cloudwalk
GROUP BY region, state
ORDER BY order_count DESC;

## NUMBER OF ON TIME DELIVERY PER PROVIDER

 SELECT provider,otd, COUNT(otd) AS count_ontime
  FROM cloudwalk
  WHERE 
	otd = 'On Time'
  GROUP BY provider,otd

## QUERY TO CALCULATE AVERAGE SHIPPING COST PER PROVIDER PER REGION

WITH provider1 as (
	SELECT
		provider,
		region,
		AVG(shipment_cost)
	FROM
		cloudwalk
	WHERE provider = 'provider 1'
	GROUP BY provider,region
	ORDER BY provider,region
), provider2 as (
	SELECT
		provider,
		region,
		AVG(shipment_cost)
	FROM
		cloudwalk
	WHERE provider = 'provider 2'
	GROUP BY provider,region
	ORDER BY provider,region
)
SELECT 
      a.provider,
      a.avg AS avg_provider_1,
      b.provider,
      b.avg AS avg_provider_2,
      a.region
FROM provider1 a
FULL OUTER JOIN provider2 b
    ON a.region = b.region

## SAME QUERY AS ABOVE BUT CALCULATING THE AVERAGE DELIVERY TIME PER PROVIDER PER REGION

WITH provider1 AS (
    SELECT
        provider,
        region,
        AVG(delivery_time) AS avg_delivery_time
    FROM
        cloudwalk
    WHERE provider = 'provider 1'
    GROUP BY provider, region
    ORDER BY provider, region
), provider2 AS (
    SELECT
        provider,
        region,
        AVG(delivery_time) AS avg_delivery_time
    FROM
        cloudwalk
    WHERE provider = 'provider 2'
    GROUP BY provider, region
    ORDER BY provider, region
)
SELECT 
    a.provider AS provider_1,
    a.avg_delivery_time AS avg_delivery_time_provider_1,
    b.provider AS provider_2,
    b.avg_delivery_time AS avg_delivery_time_provider_2,
    a.region
FROM provider1 a
FULL OUTER JOIN provider2 b
    ON a.region = b.region;

## DIFFERENCE CALCULATIONS BETWEEN A SALE AND A DELIVERY

with calculations as (select 
	(delivered_at - processing_at) AS processed_delivered,
	(delivered_at - sales_order_created_at) AS sold_delivered,
	((delivered_at - sales_order_created_at) - (delivered_at - processing_at)) as difference
from cloudwalk
WHERE status ILIKE 'delivered')
select 
	avg(processed_delivered) as avg_processed_delivered, 
	avg(sold_delivered) as avg_sold_delivered,
	avg(difference) as avg_difference
from calculations

## CALCULATIONS OF AVERAGE TIME BETWEEN DELIVERY STAGES (considering when less than one day, the result will be shown in hours, when more than one day the result will be in days)

SELECT 
  AVG(EXTRACT(epoch FROM (device_order_created_at - sales_order_created_at)) / 86400) AS diff_sales_to_order,
  AVG(processing_at - device_order_created_at) AS diff_order_to_processing,
  AVG(in_transit_to_local_distribution_at - processing_at) AS diff_processing_to_local_dist,
  AVG(EXTRACT(epoch FROM (local_distribution_at - in_transit_to_local_distribution_at)) / 86400) AS diff_local_dist_to_local_arrive,
  AVG(EXTRACT(epoch FROM (in_transit_to_deliver_at - local_distribution_at)) / 86400) AS diff_local_arrive_to_transit_deliver,
  AVG(EXTRACT(epoch FROM (delivered_at - in_transit_to_deliver_at)) / 86400) AS diff_delivery_to_transit_deliver
FROM cloudwalk
WHERE status = 'delivered';

## PERCENTAGES OF OTD PER REGION

WITH delivered_region AS (
    SELECT region, COUNT(*) AS delivered
    FROM cloudwalk
    WHERE status ILIKE 'delivered'
    GROUP BY region
),
on_time AS (
    SELECT region, COUNT(*) AS on_time
    FROM cloudwalk
    WHERE status ILIKE 'delivered' AND otd ILIKE 'on time'
    GROUP BY region
)
SELECT
    a.region,
    a.delivered,
    b.on_time,
    (b.on_time * 100.0) / a.delivered AS percentage_on_time
FROM
    delivered_region a
INNER JOIN
    on_time b
ON
    a.region = b.region;
