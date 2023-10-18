-- 1. How many users are there?
select count(DISTINCT user_id) as total_users from users;

-- 2. How many cookies does each user have on average?
with cte as (
    SELECT user_id, count(DISTINCT cookie_id) as cookie from users GROUP BY user_id
)
SELECT round(avg(cookie)) from cte;

-- 3. What is the unique number of visits by all users per month?
select date_part('month', event_time) as visit_month, count(DISTINCT visit_id) as visits
from events GROUP BY visit_month ORDER BY visit_month;

-- 4. What is the number of events for each event type?
SELECT ei.event_name, count(*) as total
from events e join event_identifier ei USING(event_type) GROUP BY ei.event_name;

-- 5. What is the percentage of visits which have a purchase event?
SELECT FLOOR(sum(case WHEN event_type = 3 then 1 else 0 end)::FLOAT / count(DISTINCT visit_id) * 100) as purchase from events;

-- 6. How many customers visited the checkout page but did not purchase?
with cte as (
    select sum(case when event_type = 1 and page_id = 12 then 1 else 0 end) as checkout,
    sum(case when event_type = 3 THEN 1 else 0 end) as purchase from events
)
SELECT round((1 - purchase::NUMERIC / checkout) * 100, 2) from cte;

-- 7. What are the top 3 pages by the number of views?
SELECT ph.page_name, count(e.visit_id) as page_visit from 
events e join page_hierarchy ph USING(page_id) GROUP BY ph.page_name ORDER BY page_visit DESC LIMIT 3;

-- 8. What is the number of views and cart adds for each product category?
SELECT ph.product_category,
sum(case when event_type = 1 then 1 else 0 end) as views,
sum(case when event_type = 2 then 1 else 0 end) as carts
from events e join page_hierarchy ph USING(page_id) WHERE ph.product_category != '' GROUP BY ph.product_category;

-- 9. What are the top 3 products by purchases?
with cte as (
    select ph.product_id, count(*) as total from events e join page_hierarchy ph USING(page_id) WHERE event_type = 2 and ph.product_id is not null and e.visit_id in (SELECT visit_id FROM events WHERE event_type = 3) 
    GROUP BY ph.product_id
)
SELECT * from cte ORDER BY total DESC LIMIT 3;

-- 10. How many times was each product viewed, added to the cart, added to a cart but not purchased (abandoned), and purchased?
SELECT ph.product_id,
sum(case when e.event_type = 1 then 1 else 0 end) as views,
sum(case when e.event_type = 2 then 1 else 0 end) as addtocart,
sum(case when e.event_type = 2 and e.visit_id in (SELECT visit_id from events WHERE event_type = 3) then 1 else 0 end) as purchase,
sum(case when e.event_type = 2 and e.visit_id not in (SELECT visit_id from events WHERE event_type = 3) then 1 else 0 end) as not_purchased
from events e join page_hierarchy ph USING(page_id) WHERE ph.product_id IS not null
GROUP BY ph.product_id ORDER BY purchase DESC;

-- Campign Analysis

-- 11. Campign Analysis
SELECT u.user_id, e.visit_id, min(e.event_time) as visit_start_time,
count(e.page_id) as page_views,
sum(case when e.event_type = 2 then 1 else 0 end) as cart_adds,
max(case when event_type = 3 then 1 else 0 end) as purchase,
COALESCE((SELECT ci.campaign_name from campaign_identifier ci WHERE min(e.event_time) >= ci.start_date and min(e.event_time) <= ci.end_date), '') as campignname,
sum(case when e.event_type = 4 then 1 else 0 end) as impressions,
sum(case when e.event_type = 5 then 1 else 0 end) as ad_click,
array_remove(array_agg((SELECT product_id from page_hierarchy ph WHERE product_id is not null and ph.page_id = e.page_id) ORDER BY e.sequence_number), null) as cart_products
from events e INNER join users u USING(cookie_id)
GROUP BY u.user_id, e.visit_id
order by u.user_id;
