+++
title = 'Sql Interview Hard'
date = 2024-03-22T21:27:03-07:00
+++

**Questions**

> Active user retention [link](https://datalemur.com/questions/user-retention)

```sql
with ActiveUsers as
(select 
  distinct 
  user_id,
  date_part('month', event_date) as Month,
  max(case when event_type in ('sign-in', 'like', 'comment') then 
  1 else 0 end) over (partition by user_id, date_part('month', event_date))
    as IsActive
from user_actions
where date_part('month', event_date) in ('06','07')),
SideBySide as
(select 
  user_id,
  Month,
  lag(Month) over (partition by user_id order by Month) as PrevMonth
  from ActiveUsers)
select Month,
  count(*)
from SideBySide
where Month = 7 and PrevMonth is not null
group by Month
```

> Active User growth [link](https://datalemur.com/questions/yoy-growth-rate)

```sql
With SpendPerYear as
(select 
  distinct 
    product_id,
    date_part('year', transaction_date) as theyear,
    sum(spend) over (partition by product_id, date_part('year', transaction_date))
      as spend
from user_transactions),
YoY as (
select 
    theyear,
    product_id,
    spend as curr_year_spend,
    lag(spend) over (partition by product_id order by theyear) as prev_year_spend
    from SpendPerYear
)
select 
  theyear as year,
  product_id,
  curr_year_spend,
  prev_year_spend,
  round(100.0 * ((curr_year_spend::decimal - prev_year_spend::decimal)
    / prev_year_spend::decimal),2) as yoy_rate
from YoY
```

> Median Google Search Frequency [link](https://datalemur.com/questions/median-search-freq)

```sql
-- make search repeat num_users times for each row
-- 1 | 2 | [1,1]
-- 2 | 2 | [2,2]
-- 3 | 3 | [3,3,3]
with Flattened as
(select 
  searches,
  num_users,
  generate_series(1,num_users)
from search_frequency)
select 
  round(PERCENTILE_CONT(0.5) within group (order by searches)::decimal,1)
from Flattened

```

> Advertiser Status [link](https://datalemur.com/questions/updated-status)

```sql
with CurrentPayment AS
(select 
  coalesce(a.user_id,p.user_id) as user_id,
  a.status,
  p.paid
from 
  advertiser a FULL OUTER JOIN daily_pay p 
  on a.user_id = p.user_id)
select 
  user_id,
  case 
    when status = 'NEW' and paid is not NULL then 'EXISTING'
    when status = 'NEW' and paid is NULL then 'CHURN'
    when status = 'EXISTING' and paid is not NULL then 'EXISTING'
    when status = 'EXISTING' and paid is NULL then 'CHURN'
    when status = 'CHURN' and paid is not null then 'RESURRECT'
    when status = 'CHURN' and paid is null then 'CHURN'
    when status = 'RESURRECT' and paid is not null then 'EXISTING'
    when status = 'RESURRECT' and paid is null then 'CHURN'
    when status is null and paid is not null then 'NEW'
  end as new_status
from CurrentPayment
order by user_id
```

> Consecutive filing years [link](https://datalemur.com/questions/consecutive-filing-years)

```sql
with FiledWithTurbo AS
(
  select distinct user_id,
    date_part('year', filing_date) as theYear
  from filed_taxes
  where product like 'TurboTax%'
),
Lookahead as
(
  select user_id,
  theYear,
  generate_series(theYear::int, (theYear+2)::int) as nextYear
  from FiledWithTurbo  
)
select 
  distinct t.user_id
from FiledWithTurbo t inner join Lookahead l
  on t.user_id = l.user_id and t.theyear = l.nextYear
group by t.user_id, t.theyear
having count(*) = 3
```

> 3-topping Pizza (https://datalemur.com/questions/pizzas-topping-cost)

```sql
select
  concat(a.topping_name,',',b.topping_name,',',c.topping_name) as pizza,
  a.ingredient_cost + b.ingredient_cost + c.ingredient_cost as total_cost
from 
  pizza_toppings a
  cross join
  pizza_toppings b
  cross join
  pizza_toppings c
where a.topping_name < b.topping_name
and b.topping_name < c.topping_name
--and a.topping_name <> c.topping_name
order by total_cost desc, pizza
--order by a.topping_name,b.topping_name,c.topping_name
```

> Total Server Utilization time [link](https://datalemur.com/questions/total-utilization-time)

```sql
with RowNum as
(select server_id,
  session_status,
  status_time,
  row_number() over (partition by server_id, session_status
    order by status_time
  )
from 
server_utilization),
UpTimes as 
(SELECT
  server_id, row_number,
  status_time - lag(status_time) over (
    partition by server_id, row_number 
    order by status_time)
  as uptime
from RowNum)
select 
  round(extract(days from sum(uptime))
    +
  extract(hours from sum(uptime)) / 24)
    as total_uptime_days
  from UpTimes
where uptime is not null
```

> Same week purchases [link](https://datalemur.com/questions/same-week-purchases)

```sql
with RankedPurchases as
(
  select 
    user_id,
    purchase_date,
    row_number() over (partition by user_id order by purchase_date)
      as rnum
  from user_purchases
),
-- we get their first purchase date
FirstPurchase as
(
  select user_id, purchase_date
  from RankedPurchases where rnum = 1
),
-- all signed up users must be counted
-- even those that never had a purchase
SignsUpJoinedWithFirstPurchase as 
(
  select s.user_id,
    s.signup_date,
    case when p.purchase_date is not null
      then p.purchase_date - s.signup_date 
    else null end as first_purchase_after_signup
  from signups s left join firstpurchase p
      on s.user_id = p.user_id
)
select
  round(100.0 * (count(*) filter (where first_purchase_after_signup 
    < '7 days'::interval)::decimal
    / count(*)::decimal),2)
from SignsUpJoinedWithFirstPurchase
```
