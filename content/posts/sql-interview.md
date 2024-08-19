+++
title = 'Datalemur SQL interview questions (Easy)'
date = 2024-03-11T11:32:50-07:00
+++

**Question**

> Histogram of tweets - [link](https://datalemur.com/questions/sql-histogram-tweets)

```
Example Output:
tweet_bucket	users_num
1	2
2	1
```

```sql
with tweets_per_user as (
select count(*) as num_tweets, user_id
  from tweets
  where date_part('year', tweet_date) = 2022
  group by user_id
)
select num_tweets as tweet_bucket, 
  count(*) as user_num
  from tweets_per_user 
group by tweet_bucket
```

**Question**

> Rolling average tweets - [link](https://datalemur.com/questions/rolling-average-tweets)

```
Example Output:
user_id	tweet_date	rolling_avg_3d
111	06/01/2022 00:00:00	2.00
111	06/02/2022 00:00:00	1.50
111	06/03/2022 00:00:00	2.00
111	06/04/2022 00:00:00	2.67
111	06/05/2022 00:00:00	4.00
```


```sql
SELECT distinct user_id, tweet_date,
  round(
  avg(tweet_count) over (
    partition by user_id 
    order by tweet_date 
    range between 
      interval '2 days' preceding 
      and
      current row 
  ),2)
  as rolling_avg_3d
  FROM tweets
  order by user_id, tweet_date
```

**Question**

> Twitter user session activity - [link](https://datalemur.com/questions/user-session-activity)

```
Example Output:
user_id	session_type	ranking
333	reply	1
222	reply	2
111	retweet	1
```

```sql
with TotalDuration AS
(
select 
  distinct user_id, session_type,
  sum(duration) over (partition by user_id, 
    session_type 
    ) total_duration
  from sessions
  where start_date BETWEEN '2022-01-01'::timestamp
    and '2022-02-01'::timestamp
),
Ranked as 
(
  select 
    user_id,
    session_type,
    rank() over (partition by session_type 
    order by total_duration desc)
      as ranking
  from TotalDuration
)
select * from Ranked
```

**Question**

> Data science skills - [link](https://datalemur.com/questions/matching-skills)

```sql
with potential_candidates AS
(select candidate_id, 
  count(*) over (partition by candidate_id) as required_skills_count
from candidates
where skill in ('Python', 'Tableau', 'PostgreSQL'))
select distinct candidate_id
 from potential_candidates
 where required_skills_count = 3
order by candidate_id
```

**Question**

> Page with no likes - [link](https://datalemur.com/questions/sql-page-with-no-likes)

```sql
SELECT p.page_id FROM pages p
  LEFT JOIN page_likes pl on p.page_id = pl.page_id
where pl.page_id is null
order by p.page_id
```

**Question**

> Telsa unfinished parts - [link](https://datalemur.com/questions/tesla-unfinished-parts)

```sql
SELECT part, assembly_step 
FROM parts_assembly
where finish_date is null
```

**Question** 

> Laptop vs mobile viewership - [link](https://datalemur.com/questions/laptop-mobile-viewership)

```sql
SELECT 
  sum(case when device_type = 'laptop' then 1 else 0 end) as laptop_views,
  sum(case when device_type in ('phone', 'tablet') then 1 else 0 end) as
    mobile_views
FROM viewership;
```

**Question** 

> Average post hiatus - [link](https://datalemur.com/questions/sql-average-post-hiatus-1)

```sql
-- count of posts > 2 in 2021
-- for every users, get the first and last date
-- then datediff(last and first)
with Gt2PostsIn2021
as (
select count(*), user_id
from posts
  where date_part('year', post_date) = 2021
  group by user_id
  having count(*) > 2
)
SELECT 
  distinct user_id,
  max(post_date::date) over (partition by user_id)
  -
  min(post_date::date) over (partition by user_id) 
  from posts
  where user_id in (select user_id from Gt2PostsIn2021)
  
```

And an overly complicated version - on a day when I forgot about `min` and `max` :)

```sql
with UsersIn2021 as
(
  select user_id,
  count(*) as num_posts
  from posts
  where date_part('year', post_date) = 2021
  group by user_id
  having count(*) >= 2
),
RankedPosts AS
(
  select
    user_id,
    post_date,
    rank() over (partition by user_id order by post_date) asc_rnk,
    rank() over (partition by user_id order by post_date desc) desc_rnk
  from posts
  where user_id in (select user_id from UsersIn2021)
),
FirstAndLast as
(select
  user_id,
  post_date
  from RankedPosts where asc_rnk = 1
UNION ALL
  select user_id,
  post_date
  from RankedPosts where desc_rnk = 1
),
Hiatus as
(select 
  user_id,
  lead(post_date) over (partition by user_id order by post_date) - post_date 
    as days_between
  from FirstAndLast
)
select user_id,
  extract(days from days_between::interval) as days_between
  from Hiatus
where days_between is not null
```

**Question** 

> Teams power users - [link](https://datalemur.com/questions/teams-power-users)

```sql
select sender_id, count(*) as message_count
  from messages
  where 
    date_part('year', sent_date::date) = 2022
    and date_part('month', sent_date::date) = 08
group by sender_id
order by message_count DESC
limit 2
```

**Question** 

> Duplicate job listing - [link](https://datalemur.com/questions/duplicate-job-listings)

```sql
SELECT count(*) as duplicate_companies 
  from 
  (select distinct company_id, 
    count(*) over (partition by company_id, title, description)
      as duplicate_posts
    from job_listings
  ) a 
  where a.duplicate_posts >= 2
```

**Question** 

> Cities with completed trades - [link](https://datalemur.com/questions/completed-trades)

```sql
select 
  distinct city,
  count(*) over (partition by city) as total_orders
  from 
    trades t inner join users u on t.user_id = u.user_id 
      and t.status = 'Completed'
  order by total_orders desc
  limit 3
```

**Question** 

> Average review ratings - [link](https://datalemur.com/questions/sql-avg-review-ratings)

```sql
SELECT 
  distinct 
    date_part('month', submit_date) as mth,
    product_id as product,
    round(avg(stars) over (partition by date_part('month', submit_date), product_id) 
      ,2) as avg_stars
  FROM reviews
  order by mth, product
```

**Question** 

> Final account balance - [link](https://datalemur.com/questions/final-account-balance)

```sql
select account_id,
  sum(case when transaction_type = 'Deposit' 
    then amount else 0 end)
  - 
  sum(case when transaction_type = 'Withdrawal'
    then amount else 0 end) as final_balance
  from transactions
  group by account_id
  
```

**Question** 

> Quickbooks vs Turbotax - [link](https://datalemur.com/questions/quickbooks-vs-turbotax)

```sql
select 
  sum(case when product ilike 'turbotax%' then 1 else 0 end)
    as turbotax_total,
  sum(case when product ilike 'quickbooks%' then 1 else 0 end)
    as quickbooks_total
  from filed_taxes
  
```

**Question** 

> Click through rate - [link](https://datalemur.com/questions/click-through-rate)

```sql
SELECT distinct app_id,
  round(100.0 * (
    cast(sum(case when event_type = 'click' then 1 else 0 end)
      over (partition by app_id) as decimal)
    /
    cast(sum(case when event_type ='impression' then 1 else 0 end)
      over (partition by app_id) as decimal)
  ),2) as ctr
  from events
  where date_part('year', timestamp::date) = 2022
```

**Question** 

> Second day confirmation - [link](https://datalemur.com/questions/second-day-confirmation)

```sql
select user_id
  from texts t inner join emails e on t.email_id = e.email_id
  where t.signup_action = 'Confirmed'
  and t.action_date::date - e.signup_date::date = 1
```

**Question** 

> Cards issued difference - [link](https://datalemur.com/questions/cards-issued-difference)

```sql
SELECT 
  distinct card_name,
  max(issued_amount) over (partition by card_name)
    - min(issued_amount ) over (partition by card_name)
  as difference
  from monthly_cards_issued
  order by difference desc
```

**Question** 

> Alibaba compressed mean - [link](https://datalemur.com/questions/alibaba-compressed-mean)

```sql
SELECT
  round(SUM(item_count * order_occurrences)::decimal
    / sum(order_occurrences)::decimal, 1) as mean
from items_per_order
  
```

**Question** 

>  - [link]()

```sql

```

**Question** 

> Pharmacy analytics 1 - [link](https://datalemur.com/questions/top-profitable-drugs)

```sql
select drug, total_profit
FROM
  (
    select
      product_id, 
      drug,
      (total_sales - cogs) as total_profit,
      rank() over (order by (total_sales - cogs) desc) as rnk
      from pharmacy_sales
    ) alias
 where rnk <=3 
```

**Question** 

> Most expensive purchase - [link](https://datalemur.com/questions/most-expensive-purchase)

```sql
select DISTINCT customer_id,
  max(purchase_amount) over (partition by customer_id 
    order by purchase_amount desc)
    as purchase_amount
from transactions
order by purchase_amount desc
```

**Question** 

> ApplePay volume - [link](https://datalemur.com/questions/apple-pay-volume)

```sql
select merchant_id,
  sum(case when payment_method ilike '%apple pay%' 
    then transaction_amount else 0 end) as total_transaction
  from transactions
group by merchant_id
order by total_transaction desc
```

**Question** 

> Accenture SMEs - [link](https://datalemur.com/questions/subject-matter-experts)

```sql
with experience as 
(SELECT
  distinct employee_id,
  count(*) over (partition by employee_id)
    as domains,
  sum(years_of_experience)
    over (partition by employee_id) as total_experience
from employee_expertise)
-- select * from experience
select employee_id
from experience
where 
  (domains = 1 and total_experience > 8)
  or
  (domains = 2 and total_experience > 12)
order by employee_id
```

**Question** 

> Linked power creators - [link](https://datalemur.com/questions/linkedin-power-creators)

```sql
select distinct 
  profile_id
from personal_profiles p 
  inner join company_pages c 
  on p.employer_id = c.company_id
where p.followers > c.followers
order by profile_id
```

**Question** 

> Highest number of products - [link](https://datalemur.com/questions/sql-highest-products)

```sql
SELECT
  user_id,
  count(*) as num_products
from user_transactions
group by user_id
having sum(spend) > 1000
order by num_products desc, sum(spend) desc
limit 3
```

**Question** 

> Spare server capacity - [link](https://datalemur.com/questions/sql-spare-server-capacity)

```sql
with forecast as
(
  select distinct 
    datacenter_id,
  sum(monthly_demand) over (partition by datacenter_id)
    as total_demand
  from forecasted_demand
)
select d.datacenter_id,
  monthly_capacity - f.total_demand
from datacenters d
inner join forecast f 
    on f.datacenter_id = d.datacenter_id
order by d.datacenter_id
  
```

**Question** 

> Top rated businesses - [link](https://datalemur.com/questions/sql-top-businesses)

```sql
select 
  sum(case when review_stars in (4,5) then 1 else 0 end) 
   as business_count,
  round(
  100.0 * 
    sum(case when review_stars in (4,5) then 1 else 0 end)::decimal 
    / count(*)::decimal) as top_rated_pct
from reviews
  
```

**Question** 

> Ad campaign ROAS - [link](https://datalemur.com/questions/ad-campaign-roas)

```sql
select 
  advertiser_id,
  round(sum(revenue)::decimal / sum(spend)::decimal,2) 
   as roas
from ad_campaigns
group by advertiser_id
order by advertiser_id
```

**Question** 

> Revenue by product line - [link](https://datalemur.com/questions/revenue-by-product-line)

```sql
select 
  distinct product_line,
  sum(amount) over
    (partition by product_line) as total_revenue
  from transactions t 
  inner JOIN product_info as p 
    on t.product_id = p.product_id
  order by total_revenue DESC
```

**Question**

> Trade-in payouts - [link](https://datalemur.com/questions/trade-in-payouts)

```sql
select store_id,
  sum(payout_amount) as payout_total
  from trade_in_transactions t
    inner join trade_in_payouts p
      on t.model_id = p.model_id
  group by store_id
  order by payout_total desc
```

**Question**

> Webinar popularity - [link](https://datalemur.com/questions/snowflake-webinar-popularity)

```sql
select 
  round(
    100.0 * sum(case when event_type = 'webinar' then 1 else 0 end)::decimal
    / 
    count(*)::decimal
  ) as webinar_pct
from marketing_touches
where date_part('month', event_date)=4
and date_part('year', event_date)=2022
```

**Question**

> Oracle sales quota - [link](https://datalemur.com/questions/oracle-sales-quota)

```sql
with TotalSales as 
(
  select employee_id,
    sum(deal_size) as total_sales
    from deals
  GROUP BY employee_id
)
select 
  ts.employee_id,
  case when ts.total_sales > s.quota then 'yes' else 'no' end
    as made_quota
from TotalSales ts inner join sales_quotas s
  on ts.employee_id = s.employee_id
order by ts.employee_id
```