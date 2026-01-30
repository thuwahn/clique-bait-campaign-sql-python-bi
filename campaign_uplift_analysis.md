# A. EVENT PROCESSING & USER SEGMENTATION

Create a view that map relevant events to users:

```sql
CREATE VIEW user_events AS
SELECT 	u.user_id,
	    ei.event_name,
	    e.event_time
FROM events e
JOIN event_identifier ei ON ei.event_type = e.event_type
JOIN users u ON u.cookie_id = e.cookie_id
WHERE ei.event_name IN ('Ad Impression', 'Ad Click', 'Purchase');
```

Create a view that maps each user event to the campaign active at the time of the event:

```sql
CREATE VIEW campaign_events AS
SELECT 	u.user_id,
		u.event_name,
	    u.event_time,	    
	    c.campaign_id,
	    c.campaign_name
FROM user_events u
JOIN campaign_identifier c ON u.event_time 
	BETWEEN c.start_date AND c.end_date;
```

Users were segmented into mutually exclusive exposure groups ('Non-exposed', 'Impression-only', 'Clickers'):

```sql
CREATE VIEW user_segmentation AS
SELECT	user_id,
		MIN(event_time) AS event_time,
		campaign_id,
		campaign_name,

		MAX(CASE WHEN event_name = 'Ad Impression' THEN 1 ELSE 0 END) AS has_impression,
		MAX(CASE WHEN event_name = 'Ad Click' THEN 1 ELSE 0 END) AS has_click,
		MAX(CASE WHEN event_name = 'Purchase' THEN 1 ELSE 0 END) AS has_purchase,

		CASE
			WHEN MAX(CASE WHEN event_name = 'Ad Impression' THEN 1 ELSE 0 END) = 0
				THEN 'Non-exposed'
	        WHEN MAX(CASE WHEN event_name = 'Ad Impression' THEN 1 ELSE 0 END) = 1
	            AND MAX(CASE WHEN event_name = 'Ad Click' THEN 1 ELSE 0 END) = 0
	            THEN 'Impression-only'
	        WHEN MAX(CASE WHEN event_name = 'Ad Click' THEN 1 ELSE 0 END) = 1
	            THEN 'Clicker'
	     END AS user_segmentation 
			
FROM campaign_events 
GROUP BY user_id, campaign_id, campaign_name;

SELECT *
FROM user_segmentation;
```

| user_id | event_time  | campaign_id | campaign_name                          | has_impression | has_click | has_purchase | user_segmentation |
|--------:|------------:|------------:|----------------------------------------|---------------:|----------:|-------------:|-------------------|
| 445     | 2020-02-03  | 3           | Half Off - Treat Your Shelf(...)       | 1              | 0         | 1            | Impression-only   |
| 448     | 2020-01-25  | 2           | 25% Off - Living The Lux Life          | 1              | 0         | 1            | Impression-only   |
| 388     | 2020-01-17  | 2           | 25% Off - Living The Lux Life          | 1              | 1         | 1            | Clicker           |
| 443     | 2020-02-24  | 3           | Half Off - Treat Your Shelf(...)       | 1              | 1         | 1            | Clicker           |
| 6       | 2020-01-25  | 2           | 25% Off - Living The Lux Life          | 0              | 0         | 1            | Non-exposed       |
| 384     | 2020-01-03  | 1           | BOGOF - Fishing For Compliments        | 0              | 0         | 1            | Non-exposed       |

Create a view that transforms event into product-level user events:

```sql
CREATE VIEW product_events AS
SELECT 	u.user_id,
		e.visit_id,
	  	p.page_name AS product_name,
	 	ei.event_name,
	  	p.product_id,
	  	DATE_TRUNC('month', e.event_time)::DATE AS month
FROM events e
JOIN page_hierarchy p ON e.page_id = p.page_id
JOIN event_identifier ei ON ei.event_type = e.event_type
JOIN users u ON u.cookie_id = e.cookie_id
WHERE ei.event_name IN ('Page View', 'Add to Cart', 'Purchase');
```

# B. FUNNEL METRICS

Funnel metrics are calculated using user-level flags to avoid double counting and to measure conversion from impression → click → purchase accurately.
DAX measures are used to compute the distinct number of users at each stage of the funnel:

```DAX
Impression Users =
CALCULATE(
    DISTINCTCOUNT('hqtcsdl user_segmentation'[user_id]),
    'hqtcsdl user_segmentation'[has_impression] = 1
)

Click Users =
CALCULATE(
    DISTINCTCOUNT('hqtcsdl user_segmentation'[user_id]),
    'hqtcsdl user_segmentation'[has_click] = 1
)

Purchase Users =
CALCULATE(
    DISTINCTCOUNT('hqtcsdl user_segmentation'[user_id]),
    'hqtcsdl user_segmentation'[has_click] = 1 &&
    'hqtcsdl user_segmentation'[has_purchase] = 1
)

Impression Abandoned = 
CALCULATE(
    DISTINCTCOUNT('hqtcsdl user_segmentation'[user_id]),
    'hqtcsdl user_segmentation'[has_impression] = 1 &&
    'hqtcsdl user_segmentation'[has_click] = 0
)

Click Abandoned = 
CALCULATE(
    DISTINCTCOUNT('hqtcsdl user_segmentation'[user_id]),
    'hqtcsdl user_segmentation'[has_click] = 1 &&
    'hqtcsdl user_segmentation'[has_purchase] = 0
)
```

Conversion rates are calculated by dividing the distinct user counts between funnel stages:

```DAX
CTR = DIVIDE([Click Users], [Impression Users])
Purchase CR = DIVIDE([Purchase Users], [Click Users])
Overall Funnel CR = DIVIDE([Purchase Users], [Impression Users])
```


# C. PURCHASE UPLIFT ANALYSIS

```sql
CREATE VIEW uplift_metrics AS
WITH event_user AS (
	-- Join events with users to retrieve user_id from cookie_id
    SELECT	u.user_id,	
        	ei.event_name,
        	e.event_time        
    FROM events e
	JOIN event_identifier ei ON ei.event_type = e.event_type
    JOIN users u ON e.cookie_id = u.cookie_id
),
campaign_users AS (
	-- Identify all users who appeared during the campaign period
    SELECT DISTINCT
			c.campaign_id,
	        c.campaign_name,
	        e.user_id
    FROM campaign_identifier c
    JOIN event_user e ON e.event_time 
		BETWEEN c.start_date AND c.end_date
),
user_flags AS (
	-- Identify impression, click, and purchase events for each user within each campaign
    SELECT	cu.campaign_id,
			cu.campaign_name,
        	cu.user_id,
			
	        MAX(CASE WHEN eu.event_name = 'Ad Impression' THEN 1 ELSE 0 END) AS has_impression,
	        MAX(CASE WHEN eu.event_name = 'Ad Click' THEN 1 ELSE 0 END) AS has_click,
	        MAX(CASE WHEN eu.event_name = 'Purchase' THEN 1 ELSE 0 END) AS has_purchase
			
    FROM campaign_users cu
    LEFT JOIN event_user eu ON cu.user_id = eu.user_id
		AND eu.event_time BETWEEN
			(SELECT start_date FROM campaign_identifier c WHERE c.campaign_name = cu.campaign_name)
             AND
            (SELECT end_date FROM campaign_identifier c WHERE c.campaign_name = cu.campaign_name)
    GROUP BY cu.campaign_id, cu.campaign_name, cu.user_id
)
SELECT	campaign_id,
    	campaign_name,

	    -- Total number of users in each group
		COUNT(DISTINCT CASE WHEN has_impression = 1 THEN user_id END) AS impression_user,
    	COUNT(DISTINCT CASE WHEN has_impression = 0 THEN user_id END) AS no_impression_user,
		COUNT(DISTINCT CASE WHEN has_click = 1 THEN user_id END) AS click_users,
    	COUNT(DISTINCT CASE WHEN has_impression = 1 AND has_click = 0 THEN user_id END) AS impression_only_users,
	
	    -- Total number of purchasers in each group
		COUNT(DISTINCT CASE WHEN has_impression = 1 AND has_purchase = 1 THEN user_id END) AS impression_purchasers,
    	COUNT(DISTINCT CASE WHEN has_impression = 0 AND has_purchase = 1 THEN user_id END) AS no_impression_purchasers,
		COUNT(DISTINCT CASE WHEN has_click = 1 AND has_purchase = 1 THEN user_id END) AS click_purchasers,
    	COUNT(DISTINCT CASE WHEN has_impression = 1 AND has_click = 0 AND has_purchase = 1 THEN user_id END) AS impression_only_purchasers
	
FROM user_flags
GROUP BY campaign_id, campaign_name
ORDER BY campaign_id;
```
