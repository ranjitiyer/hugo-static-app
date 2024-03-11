+++
title = 'SQL questions on DataLemur (medium)'
date = 2024-03-11T11:59:43-07:00
+++

**Question**

> Users's third transaction - [link](https://datalemur.com/questions/sql-third-transaction)

```sql
SELECT user_id, spend, transaction_date
from 
  (select user_id, spend, transaction_date,
    rank() over (partition by user_id order by transaction_date) as rnk
    from transactions) as r
where r.rnk = 3
```

**Question**

> Sending vs opening snaps - [link](https://datalemur.com/questions/time-spent-snaps)

```sql
SELECT age_bucket,
  round(100.0 * 
  CAST(sum(case when activity_type = 'send' then time_spent else 0 end)
    as decimal)
    /
  cast(sum(case when activity_type in ('send','open') then time_spent else 0 end)
    as decimal),2) 
  as send_perc,
  round(100.0 * 
  CAST(sum(case when activity_type = 'open' then time_spent else 0 end)
    as decimal)
    /
  cast(sum(case when activity_type in ('send','open') then time_spent else 0 end)
    as decimal),2) 
  as open_perc
  from activities a inner join age_breakdown b
    on a.user_id = b.user_id
  group by age_bucket
  
  
```

**Question**

> Tweets' rolling average - [link](https://datalemur.com/questions/rolling-average-tweets)

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

> Highest grossing items - [link](https://datalemur.com/questions/sql-highest-grossing)

```sql
;with MaxSpends as
(select 
  distinct 
  category,
  product,
  sum(spend) over (partition by category, product) as total_spend
from product_spend
where extract('year' from transaction_date) = 2022),
Ranks as
(
  select category, product, total_spend,
    rank() over (partition by category order by total_spend desc) as rnk
  from MaxSpends
)
select category, product, total_spend
from Ranks
where rnk in (1,2)
```

**Question**

> Signup activation rate - [link](https://datalemur.com/questions/signup-confirmation-rate)

```sql
select 
  round(cast(sum(case when t.signup_action='Confirmed' 
      then 1 else 0 end) as decimal) / 
  cast(count(distinct e.email_id) as decimal),2)
  from emails e left join texts t on e.email_id = 
    t.email_id
```

**Question**

> Fill missing client data - [link](https://datalemur.com/questions/fill-missing-product)

```sql
with Numbering as 
(select 
  product_id,
  category,
  name,
  count(category) over (order by product_id) as numbered_category
from products),
CategoryToNumber as
(select numbered_category, category
  from Numbering
  where category is not null)
select product_id, r.category as category,
  name 
  from Numbering l inner join CategoryToNumber r
    on l.numbered_category = r.numbered_category
```

**Question**

> Mean, median, mode - [link](https://datalemur.com/questions/mean-median-mode)

```sql
with MostFrequent AS
(
  select 
    0 as mean,
    0 as median,
    email_count as mode
  from inbox_stats
  group by email_count
  order by count(*) DESC
  limit 1 -- give me the top one
),
MeanMedian as
(
  select 
    sum(email_count) / count(*) as mean,
    percentile_cont(0.5) within group ( order by email_count asc ) as median,
    0 as mode
  from inbox_stats
),
Combined As
(
  select * from MostFrequent
  UNION ALL
  select * from MeanMedian
)
select sum(mean) as mean,
  sum(median) as median,
  sum(mode) as mode
from Combined

-- -- simpler version (after learning about mode() function)
-- select 
--   round(avg(email_count)) as mean,
--   percentil_cont(0.5) within group (order by email_count) as median,
--   mode() within group (order by email_count) as mode
-- from inbox_stats;
```

**Question**

> Pharmacy analytics 4 - [link](https://datalemur.com/questions/top-drugs-sold)

```sql
with UnitsSold as
(select 
  distinct 
    manufacturer,
    drug,
    sum(units_sold) over (partition by manufacturer, drug)
      as total_sold
  from pharmacy_sales
),
Ranked AS
(
  select 
    manufacturer,
    drug,
    rank() over (partition by manufacturer order by total_sold desc) 
      as rnk
  from UnitsSold
)
select 
  manufacturer,
  drug as top_drugs
from Ranked
where rnk < 3
order by manufacturer
```

**Question**

> Frequently purchased pairs - [link](https://datalemur.com/questions/frequently-purchased-pairs)

```sql
select 
  distinct 
  array_agg(product_id::text order by product_id) as combination
from transactions
group by transaction_id
having count(*) > 1
order by combination
```

**Question**

> Super cloud customer - [link](https://datalemur.com/questions/supercloud-customer)

```sql
select distinct 
  c.customer_id
  from customer_contracts c inner join products p
    on c.product_id = p.product_id
    group by c.customer_id
  having count(distinct product_category) 
    = (select count(distinct product_category) from products)
  ORDER BY c.customer_id
```

**Question**

> Odd even measurements - [link](https://datalemur.com/questions/odd-even-measurements)

```sql
With MeasurementNumber as
(
  select measurement_id,
    measurement_value,
    date_trunc('day', measurement_time) as measurement_day,
    row_number() over (partition by date_part('day', measurement_time)
      order by measurement_time) as rn
    from measurements
)
select measurement_day,
  sum(case when rn % 2 = 1 then measurement_value else 0 end)
    as odd_sum,
  sum(case when rn % 2 = 0 then measurement_value else 0 end)
    as even_sum
  from MeasurementNumber
group by measurement_day
order by measurement_day
```

**Question**

> Booking ref source - [link](https://datalemur.com/questions/booking-referral-source)

```sql
with FirstBooking AS
(
  select 
    distinct 
      user_id,
      booking_id,
      rank() over (partition by user_id order by booking_date) as rnk
      from bookings
)
select 
  distinct
  channel,
  round(100.0 * count(*) over (partition by channel)::decimal / 
    count(*) over ()::decimal) as first_booking_pct
  from FirstBooking f inner join booking_attribution a
    on f.booking_id = a.booking_id
  where rnk = 1
  order by first_booking_pct DESC
  limit 1
```

**Question**

> User shopping spree - [link](https://datalemur.com/questions/amazon-shopping-spree)

```sql
With ConsecutiveDays as
(
  select user_id,
    transaction_date, 
    case when 
      lead(transaction_date, 1) over (partition by user_id
          order by transaction_date)::date
        - transaction_date::date = 1
        then 1 else 0 end as ShoppedOnDay2,
    case when
      lead(transaction_date, 2) over (partition by user_id
          order by transaction_date)::date
        - transaction_date::date = 2
        then 1 else 0 end as ShoppedOnDay3
    from transactions
)
select user_id from
   ConsecutiveDays
 where ShoppedOnDay2 = 1 and ShoppedOnDay3 = 1
```

**Question**

> Second ride delay - [link](https://datalemur.com/questions/2nd-ride-delay)

```sql
Ranked as 
(
  select distinct 
    user_id, 
    ride_date,
  row_number() over (PARTITION BY user_id order by ride_date) as rnk
  from rides
  where user_id in (select user_id from InTheMoment)
),
FirstRide as
(
  select user_id,
  ride_date
  from Ranked where rnk = 1
),
SecondRide as
(
  select user_id,
  ride_date
  from Ranked where rnk = 2
)
select 
  round(avg(s.ride_date - f.ride_date),2) as average_delay-- in days
  from FirstRide f inner join SecondRide s
  on f.user_id = s.user_id
```

**Question**

> Histogram of users and purchasew - [link](https://datalemur.com/questions/histogram-users-purchases)

```sql
with LatestDateForEachUser AS
(
select 
  distinct 
  user_id,
  max(transaction_date) over (partition by user_id) 
    as max_transaction_date
  from user_transactions
)
select distinct  
  mt.max_transaction_date as transaction_date,
  mt.user_id,
  count(*) 
    over (partition by mt.user_id, u.transaction_date)
      as purchase_count
from LatestDateForEachUser mt inner join 
  user_transactions u on mt.user_id = u.user_id
  and u.transaction_date = max_transaction_date
  order by max_transaction_date
  
```

**Question**

> Off-topic UGC posts on Google - [link](https://datalemur.com/questions/off-topic-maps-ugc)

```sql
with places 
as (
select 
  place_category,
  count(*) as content_count
from place_info p inner join maps_ugc_review r
  on r.place_id = p.place_id 
    and content_tag is not null
    and content_tag ilike '%Off-topic%'
group by place_category
),
top_places AS
(
  select place_category,
  content_count,
  rank() over (order by content_count desc) as top_place
from places
)
select place_category as off_topic_places
  from top_places
  where top_place = 1
```

**Question**

> Alibaba compressed mean - [link](https://datalemur.com/questions/alibaba-compressed-mode)

```sql
with MaxFreq as
(select max(order_occurrences) as max_occurrences
  from items_per_order
)
select 
  item_count as mode
from MaxFreq mf inner join items_per_order i
  on mf.max_occurrences = i.order_occurrences
```

**Question**

> Card launch success - [link](https://datalemur.com/questions/card-launch-success)

```sql
With IssueMonthRanked AS
(
  select 
    card_name,
    issued_amount,
    issue_year,
    issue_month,
    row_number() over (partition by card_name
      order by issue_year, issue_month 
      ) rn
    from monthly_cards_issued
)
select card_name, issued_amount
  rn
  from IssueMonthRanked
where rn = 1
order by issued_amount desc
```

**Question**

> International call percentage - [link](https://datalemur.com/questions/international-call-percentage)

```sql
with CallerCountry
AS
(
  select 
    p.caller_id, 
    p.receiver_id,
    country_id as caller_country
  from phone_calls p inner join phone_info i
    on p.caller_id = i.caller_id
),
CallerAndReceiverCountry as
(
  select c.caller_id,
  c.receiver_id,
  c.caller_country,
  i.country_id as receiver_country
  from CallerCountry c inner join phone_info i
    on c.receiver_id = i.caller_id
)
select 
  round(100.0 * 
    cast(sum(case when caller_country <> receiver_country 
    then 1 else 0 end) as decimal) 
    /
  cast(count(*) as decimal), 1)
from CallerAndReceiverCountry
```

**Question**

> LinkedIn power creators (2) - [link](https://datalemur.com/questions/linkedin-power-creators-part2)

```sql
with CompanyFollowers as 
(SELECT
    employees.personal_profile_id,
	MAX(pages.followers) AS max_company_followers
  FROM employee_company AS employees 
  LEFT JOIN company_pages AS pages
    ON employees.company_id = pages.company_id  
GROUP BY employees.personal_profile_id
)
select 
  profile_id
from CompanyFollowers c
  inner join personal_profiles p
    on c.personal_profile_id = p.profile_id
  where p.followers > c.max_company_followers
```

**Question**

> Money transfer relationship - [link](https://datalemur.com/questions/money-transfer-relationships)

```sql
with Pairs as
(
select
  distinct 
    p.payer_id as payer_id, 
    p.recipient_id as recipient_id,
    r.payer_id right_payerid
    , r.recipient_id right_recipient_id
FROM
payments p inner join payments r
 on p.payer_id = r.recipient_id
  and p.recipient_id = r.payer_id
)
select count(*) as unique_relationships
from Pairs p
where payer_id < recipient_id

```

**Question**

> User session activity - [link](https://datalemur.com/questions/user-session-activity)

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

> First transaction on Etsy - [link](https://datalemur.com/questions/sql-first-transaction)

```sql
with Ranked AS
(
  select distinct 
    user_id,
    spend,
    transaction_date,
    row_number() over (partition by user_id order by 
    transaction_date) as rnk
  from user_transactions 
)
select count(*) from Ranked
where spend >= 50
and rnk = 1

```

**Question**

> Email table transformation - [link](https://datalemur.com/questions/email-table-transformation)

```sql
with Pivoted as
(select 
  user_id,
  case when email_type = 'personal'
    then email else NULL END as personal,
  case when email_type = 'business'
    then email else NULL end as business,
  case when email_type = 'recovery'
    then email else NULL end as recovery
from users)
, Arrays as
(select user_id,
array_agg(personal order by personal) as personal,
array_agg(business order by business) as business,
array_agg(recovery order by recovery) as recovery
from Pivoted
group by user_id)
select user_id,
  personal[1] as personal,
  business[1] as business,
  recovery[1] as recovery
from Arrays
  
```

**Question**

> Photoshop revenue analysis - [link](https://datalemur.com/questions/photoshop-revenue-analysis)

```sql
select 
    customer_id,
    sum(case when product = 'Photoshop' then 0 else revenue end) as revenue
  from adobe_transactions
  where customer_id -- only those that purchased Photoshop
    in (
      select distinct customer_id
        from adobe_transactions
        where product = 'Photoshop'
    )
  group by customer_id
```

**Question**

> Consulting bench time - [link](https://datalemur.com/questions/consulting-bench-time)

```sql
with Durations AS
(
  select 
      s.employee_id,
      e.job_id,
      (end_date - start_date + 1) as duration_days
    from staffing s
    inner join consulting_engagements e on
    s.job_id = e.job_id
  where s.is_consultant = 'true'
  and date_part('year', e.start_date) = 2021
)
select employee_id,
  365 - sum(duration_days) as bench_days
from Durations
group by employee_id
order by employee_id
```

**Question**

> Sales team compensation - [link](https://datalemur.com/questions/sales-team-compensation)

```sql
with TotalDeals
AS
(
  select employee_id,
    sum(deal_size) as total_sales
  from deals
  group by employee_id
),
Temp as
(
  select d.employee_id, 
    d.base,
    d.commission, 
    d.quota, 
    d.accelerator,
    t.total_sales,
    case when (t.total_sales - quota) > 0
      then (t.total_sales - quota) else 0 end as excess
  from TotalDeals t 
    inner join employee_contract d
    on t.employee_id = d.employee_id
)
select 
  employee_id,
  base + 
    case when 
      excess > 0 then 
        (commission * quota)
        + 
        commission * excess * accelerator
      ELSE
        (commission * total_sales)
      end as total_compensation
from Temp
order by total_compensation desc, employee_id 
```

**Question**

> Invalid search pct - [link](https://datalemur.com/questions/invalid-search-pct)

```sql
SELECT
  country,
  sum(num_search) as total_search,
  round(100.0 * 
    (sum(num_search * invalid_result_pct::decimal / 100.0)::decimal
    /sum(num_search)::decimal),2) as invalid_searches_pct
from search_category
where 
(num_search is not NULL and invalid_result_pct is not NULL)
group by country
order by country, total_search, invalid_searches_pct
```
