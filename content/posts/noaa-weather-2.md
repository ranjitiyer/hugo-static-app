+++
title = 'Noaa Weather 2'
date = 2024-06-15T14:59:49-07:00
+++

In the previous post we loaded NOAA weather station metadata and did some ad-hoc analysis. In this post we'll answer questions that could plausibly be then served through an REST API to be rendered on a UI for reporting.

**Question 1**

Given a state, show me weather trends in the last 10 years - temperature and precipitation

Detour

NOAA station data provides a lat/lon and a station code name (starting with 2 chars of country code, followed by 2 chars state code for US weather stations). A lay user however will expect their exploration to start with a city or state. So we have two options - reverse geocode all the lat/lon locations apriori to obtain city and state information, then find K nearest weather stations and pick a weather station. Let's go with the first option and limit ourselves to only weather stations in India otherwise we'll run into account limits very soon.

I'm going to use ArcGIS Online's reverse geocoding service to obtain location information with a lat/lon pair and here's the python script snippet I used

```python
import requests
def reverse_geocode(lat, lon) -> Location:
    try:
        url = f'https://geocode.arcgis.com/arcgis/rest/services/World/GeocodeServer/reverseGeocode?f=json&token=<token>&location={lon},{lat}'
        print(f'{lon},{lat}')
        resp = requests.get(url)
        json = resp.json()
        city=json['address']['City']
        county=json['address']['Subregion']
        state=json['address']['Region']
        country_name=json['address']['CntryName']
        country=json['address']['CountryCode']
        print(f'{city}')
        return Location(city=city,county=county,state=state,country_name=country_name,country=country)
    except Exception as e:
        print(e)
        return None
```

And load it into the our stations table by first staging the new cols in a temp table then updating station with the new rows

```sql
alter table station add column city varchar(128);
alter table station add column state varchar(128);
alter table station add column county varchar(128);
alter table station add column country varchar(128);
alter table station add column country_code varchar(3);

copy temporary(lat,lon,city,county,state,country,country_code)
from '/Users/ranjitiyer/work/data/weather/processed/inventory_stations_in_active_formatted.txt'
delimiter ','
CSV HEADER

update station
set city = t.city,
    county = t.county,
    state = t.state,
    country = t.country,
    country_code = t.country_code
   from temporary t where station.lat = t.lat and station.lon = t.lon
```

![stations](/images/noaa/stations_with_city_names.png)

Now we can query for aggregates starting at the state level. Let's start by calculating daily precipitation in the state of Maharashtra in July 2024
```sql
select 
        distinct
        'Maharashtra' as state,
        measuredon,
        round((sum(val) over (partition by state, measuredon)::decimal / 10::decimal) / 25::decimal,2) as rain_in_inches
        from daily_2024 daily inner join station s on 
        daily.station = s.id and s.state = 'Maharashtra'
        and date_part('month', measuredon) = 07
where measure = 'PRCP'
and station like 'IN%'
```
![results](/images/noaa/mah_daily_2024_july.png)

We'll export the result to a CSV file and render it as a bar graph for which we'll use pandas and configure matplotlib to work in interactive mode

```python
>>> import pandas as pd
>>> df = pd.read_csv('plots/mah_2024_july_rains.csv')
>>> df.head()
         state  measuredon  rain_in_inches
0  Maharashtra  2024-07-01            7.56
1  Maharashtra  2024-07-02            8.08
2  Maharashtra  2024-07-03            3.89
3  Maharashtra  2024-07-04            4.08
>>> import matplotlib.pyplot as plt
>>> plt.ion()
>>> df.plot(kind='bar', x='measuredon', y='rain_in_inches', legend=True)
```
![plot](/images/noaa/mah_daily_2024_july_plt.png)

Next let's measure rainfall accumulation to find weeks or days of heavy rains. Variations are day over day for a month, or month over month in a year, or year over year in an decade. We'll need a CTE for this

```sql
with DailyTotal as
(select 
        distinct
        'Maharashtra' as state,
        measuredon,
        round((sum(val) over (partition by state, measuredon)::decimal / 10::decimal) / 25::decimal,2) as rain_in_inches
        from daily_2024 daily inner join station s on 
        daily.station = s.id and s.state = 'Maharashtra'
        and date_part('month', measuredon) = 07
where measure = 'PRCP'
and station like 'IN%')
select state, measuredon as day,
sum(rain_in_inches) over (order by measuredon rows between unbounded preceding and current row ) as accumulated_rainfall
from DailyTotal
```
And this allows us to identify signifcant rainy days in July, for example 7/9 and 7/15 where accumulated rainfall jumped about 30 inches from the previous day
![image](/images/noaa/mah_daily_2024_july_accum.png)
![plot](/images/noaa/mah_daily_2024_july_accum_plt.png)


We can extend this idea to look at accumulation month over month to answer the question - What are the significant rain months in the state. (We'll use 2023 day since at the time of writing this blog, we haven't seen the entirety of 2024 yet)

![image](/images/noaa/mah_monthly_2023_accum.png)

Based on this data, June, July, Aug, Sept seem to the rainy months.

To wrap this up, let's attempt to plot annual rainfall measured in the last 10 years. To answer this query we're going to need to load data for last 10 years. They are currently held in individual csv files, one per year. We can load each file into a separate table first then union them all into one final table to avoid having to join across 10 tables!

```bash
-rw-r--r--   1 ranjitiyer  staff  1297937668 Aug 18 17:12 2014.csv
-rw-r--r--   1 ranjitiyer  staff  1316118769 Aug 18 17:14 2015.csv
. . .
. . . 
-rw-r--r--@  1 ranjitiyer  staff  1346626149 Jun  8 12:19 2023.csv
```

So the steps will be:

1. Pre-process each file to retain the first 4 cols
2. Load each file into it's own table (daily_20xx)
3. Load into final table with a select union all approach


