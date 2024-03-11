+++
title = 'Braintree SQL coding challenge'
date = 2024-03-10T16:11:44-07:00
+++

For this SQL challenge we're told to load country GDP data from CSV files into a database and answer some SQL questions. Let's go

### Load data from CSV files

Before loading to the DB we `head` each CSV file to examine the contents and note the column names which will become the table columns. 
```
% head continent_map.csv
country_code,continent_code
ABW,NA
AFG,AS
. . . 

% head continents.csv
continent_code,continent_name
AF,Africa
AS,Asia
. . .

% head countries.csv 
country_code,country_name
ABW,Aruba
AND,Andorra
. . .

% head per_capita.csv| more
country_code,year,gdp_per_capita
ABW,2004,22566.68216
AND,2004,29372.16674
. . . 
```

Then we create required tables

```sql

create table countries
(
    country_code varchar(3),
    country_name varchar(128)
);

create table continents
(
    continent_code varchar(2),
    continent_name varchar(128)
);

create table continent_map
(
    country_code   varchar(3),
    continent_code varchar(2)
);

create table gdp_per_capita
(
    country_code   varchar(3),
    year           integer,
    gdp_per_capita numeric
);
```

It's finally time to load data and do a quick `select`

```sql

copy continents from '/Users/ranjitiyer/work/postgres/percapita/data_csv/continents.csv' DELIMITER ',' CSV HEADER;

copy continent_map from '/Users/ranjitiyer/work/postgres/percapita/data_csv/continent_map.csv' DELIMITER ',' CSV HEADER;

copy countries from '/Users/ranjitiyer/work/postgres/percapita/data_csv/countries.csv' DELIMITER ',' CSV HEADER;

copy gdp_per_capita from '/Users/ranjitiyer/work/postgres/percapita/data_csv/per_capita.csv' DELIMITER ',' CSV HEADER;

```

`select * from gdp_per_capita limit 1`

| country\_code | year | gdp\_per\_capita |
| :--- | :--- | :--- |
| ABW | 2004 | 22566.68216 |

And now onto the questions!

**Question 1**

> Alphabetically list all of the country codes in the continent_map table that appear more than once. Display any values where country_code is null as country_code = "FOO" and make this row appear first in the list, even though it should alphabetically sort to the middle. Provide the results of this query as your answer.

```sql
with FooOnTop as
(select distinct
        case when country_code is null then 'AAA'
            else country_code end
                 as country_code,
        count(*) as count
    from continent_map
    group by country_code
    having count(*) > 1
    order by country_code)
select
    case when country_code = 'AAA' then 'FOO' else country_code end,
    count
from FooOnTop
```

**Question 2**

> For all countries that have multiple rows in the continent_map table, delete all multiple records leaving only the 1 record per country. The record that you keep should be the first one when sorted by the continent_code alphabetically ascending. Provide the query/ies and explanation of step(s) that you follow to delete these records.

```sql

start transaction;

select 'Before Delete', count(*) from continent_map;

-- Find duplicate country_codes
with Duplicates as
(select
    country_code
    from continent_map
    group by country_code
    having count(*) > 1
),
ToDelete as
(
    select *,
        row_number() over (partition by country_code order by continent_code) as rn
    from continent_map
    where country_code in (select country_code from Duplicates)
)
delete from continent_map
where
    (
        country_code in (select distinct country_code from ToDelete where rn  > 1)
            and
        continent_code in (select distinct continent_code from ToDelete where rn > 1)
    );

select 'After delete', count(*) from continent_map;
rollback
```

**Question 3**
> List the countries ranked 10-12 in each continent by the percent of year-over-year growth descending from 2011 to 2012. The percent of growth should be calculated as: ((2012 gdp - 2011 gdp) / 2011 gdp)

```sql

with yoy as
(select
        cont.continent_name,
        c.country_code,
        c.country_name,
        gdp.year,
        gdp_per_capita,
        case when year = 2012 then
            round(100.0 * (
                (gdp.gdp_per_capita - lag(gdp_per_capita) over (partition by c.country_code order by year))
                    /
                lag(gdp_per_capita) over (partition by c.country_code order by year)),2)
            else null end as yoy
from countries c
    inner join continent_map cm on c.country_code = cm.country_code    -- to get continent_code
    inner join continents cont on cm.continent_code = cont.continent_code -- to get continent name
    inner join gdp_per_capita gdp on gdp.country_code = c.country_code -- to get gdp
where gdp.year in (2011,2012)),
Ranked as
(
    select
        rank() over (partition by continent_name order by yoy desc) as rnk,
        continent_name,
        country_code,
        country_name,
        yoy::text || '%' as growth_percent
    from yoy
    where yoy is not null
)
select * from Ranked
where rnk in (10,11,12)
```

**Question 4**

> For the year 2012, create a 3 column, 1 row report showing the percent share of gdp_per_capita for the following regions: (i) Asia, (ii) Europe, (iii) the Rest of the World. Your result should look something like
> Asia	Europe	Rest of World
> 25.0%	25.0%	50.0%

```sql
with Share as(
select
    distinct c.continent_code,
    round(100.0 *
    (sum(gdp_per_capita) over (partition by c.
        continent_code)
        /
    sum(gdp_per_capita) over ()),2) as gdp_share
from gdp_per_capita gdp
    inner join continent_map c on gdp.country_code = c.country_code
where year = 2012
order by gdp_share desc)
select
    sum(case when continent_code = 'AS' then gdp_share else 0 end)::text || '%' as Asia,
    sum(case when continent_code = 'EU' then gdp_share else 0 end)::text || '%' as Europe,
    sum(case when continent_code not in ('AS', 'EU') then gdp_share else 0 end)::text || '%' as RestofWorld
from Share
```

**Question 5**

> What is the count of countries and sum of their related gdp_per_capita values for the year 2007 where the string 'an' (case insensitive) appears anywhere in the country name?

```sql
select
    count(*) as count,
    concat('$', round(ceil(sum(gdp_per_capita))))
from gdp_per_capita gdp inner join countries c on gdp.country_code = c.country_code
where lower(c.country_name) like '%an%'
and year = 2007;

-- 4b. Repeat question 4a, but this time make the query case sensitive.
select
    count(*) as count,
    concat('$', round(ceil(sum(gdp_per_capita))))
from gdp_per_capita gdp inner join countries c on gdp.country_code = c.country_code
where c.country_name like '%an%'
and year = 2007
```

**Question 6**

> Find the sum of gpd_per_capita by year and the count of countries for each year that have non-null gdp_per_capita where (i) the year is before 2012 and (ii) the country has a null gdp_per_capita in 2012. Your result should have the columns: year, country_count, total

```sql
with CountriesIn2012WithNullGdp
    as
    (
        select distinct country_code
        from gdp_per_capita
        where gdp_per_capita is null and year = 2012
    )
select
    distinct
    year,
    sum(gdp_per_capita) over (partition by year),
    sum(case when gdp_per_capita is not null then 1 else 0 end) over (partition by year) as country_count
from gdp_per_capita
where year < 2012
and country_code in (select country_code from CountriesIn2012WithNullGdp)
```

**Question 7**

> Create a single list of all per_capita records for year 2009 that includes columns: continent_name,country_code,country_name, gdp_per_capital
```sql
select
    continent_name,
       gdp.country_code,
       country_name,
       gdp_per_capita,
       sum(gdp_per_capita) over (partition by continent_name order by continent_name, substr(country_name,2,3) desc
                range between unbounded preceding and current row) as running_total
from
    gdp_per_capita gdp inner join continent_map cmap on gdp.country_code = cmap.country_code
        inner join countries c on gdp.country_code = c.country_code
        inner join continents cont on cont.continent_code = cmap.continent_code
where year = 2009
order by continent_name, substr(country_name,2,3) desc
```

**Question 8**

> Find the country with the highest average gdp_per_capita for each continent for all years.

```sql
with AvgGdp as
(select
    map.continent_code,
    c.country_name,
    round(avg(gdp_per_capita),2) as avg
from gdp_per_capita gdp inner join continent_map map on gdp.country_code = map.country_code
    inner join countries c on c.country_code = gdp.country_code
group by map.continent_code,c.country_name
order by map.continent_code),
Ranked as
(select
    *,
    rank() over (partition by continent_code order by avg desc) as rnk
    from AvgGdp)
select * from Ranked
where rnk = 1
order by continent_code
```

