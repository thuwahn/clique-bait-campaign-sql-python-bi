# A. USER SEGMENTATION

#### Funnel-relevant events mapped to users:
```sql
WITH user_events AS (
    SELECT 	u.user_id,
        	ei.event_name,
        	e.event_time
    FROM events e
    JOIN event_identifier ei ON ei.event_type = e.event_type
    JOIN users u ON u.cookie_id = e.cookie_id
    WHERE ei.event_name IN ('Ad Impression', 'Ad Click', 'Purchase')
)
```

#### Users were segmented into mutually exclusive exposure groups ('Non-exposed', 'Impression-only', 'Clickers'):
```sql
user_segmentation  AS (
	SELECT	u.user_id,
			c.campaign_id,
			c.campaign_name,

			MAX(CASE WHEN u.event_name = 'Ad Impression' THEN 1 ELSE 0 END) AS has_impression,
			MAX(CASE WHEN u.event_name = 'Ad Click' THEN 1 ELSE 0 END) AS has_click,
			MAX(CASE WHEN u.event_name = 'Purchase' THEN 1 ELSE 0 END) AS has_purchase,

			CASE
	            WHEN MAX(CASE WHEN u.event_name = 'Ad Impression' THEN 1 ELSE 0 END) = 0
	                THEN 'Non-exposed'
	            WHEN MAX(CASE WHEN u.event_name = 'Ad Impression' THEN 1 ELSE 0 END) = 1
	             	AND MAX(CASE WHEN u.event_name = 'Ad Click' THEN 1 ELSE 0 END) = 0
	                THEN 'Impression-only'
	            WHEN MAX(CASE WHEN u.event_name = 'Ad Click' THEN 1 ELSE 0 END) = 1
	                THEN 'Clicker'
	        END AS user_segmentation 
			
	FROM user_events u
	JOIN campaign_identifier c ON u.event_time BETWEEN c.start_date AND c.end_date
	GROUP BY u.user_id, c.campaign_id, c.campaign_name
)
```
| user_id | campaign_id | campaign_name                          | has_impression | has_click | has_purchase | user_segmentation |
|--------:|------------:|----------------------------------------|---------------:|----------:|-------------:|-------------------|
| 445     | 3           | Half Off - Treat Your Shelf(...)       | 1              | 0         | 1            | Impression-only   |
| 448     | 2           | 25% Off - Living The Lux Life          | 1              | 0         | 1            | Impression-only   |
| 388     | 2           | 25% Off - Living The Lux Life          | 1              | 1         | 1            | Clicker           |
| 443     | 3           | Half Off - Treat Your Shelf(...)       | 1              | 1         | 1            | Clicker           |
| 6       | 2           | 25% Off - Living The Lux Life          | 0              | 0         | 1            | Non-exposed       |
| 384     | 1           | BOGOF - Fishing For Compliments        | 0              | 0         | 1            | Non-exposed       |

