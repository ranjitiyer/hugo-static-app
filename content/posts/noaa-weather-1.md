+++
title = 'Noaa Weather 1'
date = 2024-06-15T11:55:15-07:00
+++

In this next series of posts, we'll look at how to work with NOAA weather data. NOAA provides weather data received from thousands of weather stations across the world. NOAA structures this data and makes them available in a few different ways. The full list of data sets (some of which are specific to the US) are available [here](https://www.ncdc.noaa.gov/cdo-web/datasets), although we'll focus

What makes this dataset interesting is that in addition to learning about weather trends, it also entails working through several classical stages in setting up a data processing pipeline. For example, although some datasets are CSV files, datasets containing metadata about stations, weather station inventory, etc. fixed column sizes and space separated files which need to be converted to CSV before loading to a relational database for analysis. Furthermore, weather station data is provided as lat/long coordinates and a weather station name. In the least we need to derive city, state and country names for human understandable reporting.

We'll first work with **Daily Summaries** and start by looking at how NOAA [structures](https://www.ncei.noaa.gov/pub/data/ghcn/daily/) the data files.

```bash
root
	/by_station
	/by_year
	/ghcnd-inventory.txt
	/ghcnd-stations.txt
	/ghcnd-countries.txt
	. .
```

Let's start with loading station metadata. The [readme](https://www.ncei.noaa.gov/pub/data/ghcn/daily/readme.txt) for this tells me rows in this file are space separated with fixed length columns, so we must first convert them to a CSV and use `|` as the separator since some of station names seem to have quotes in them which causes load to error. Additionally, we dropped columns that classifies a weather station using `pandas`

```python
df = pd.read_csv()
cols_to_keep = [0,1,2,3,..]
df = df.iloc[:,cols_to_keep]
df.write_csv(/path, index=False)
```
Now we can focus on the CSV conversion. Here's a snippet from the `readme` that describes the schema of the file

```bash
# ------------------------------
# Variable   Columns   Type
# ------------------------------
# ID            1-11   Character
# LATITUDE     13-20   Real
# LONGITUDE    22-30   Real
. . . 
```
And here's the python script to parse out the columns and write them out as a CSV

```python
lines=[]
lines.append('id|lat|lon|elev|name')
with open('/Users/ranjitiyer/work/data/weather/stations.txt', 'r') as st:
    for line in st.readlines():
        id = line[:11].strip()
        lat = line[12:20].strip()
        lon = line[21:30].strip()
        elev = line[31:37].strip()
        name = line[41:71].strip()
        lines.append(f'{id}|{lat}|{lon}|{elev}|{name}')
        
with open('/path/to/stations_formatted.txt','w') as st:
    st.write("\n".join(lines))
```
Now we can create a table and load into Postgres
```sql
create table station
(
  id varchar(32),
  lat decimal,
  lon decimal,
  elev decimal,
  name varchar(128)
);

copy station
from '/path/to/stations_formatted.txt'
delimiter '|' CSV header
```

```sql
select * from stations limit 2

IN001020700	14.5830	77.6330	364.0	PBO ANANTAPUR
IN001050200	16.9500	82.2330	8.0	KAKINADA
```

Great, we have a process that can be applied to `country` and `inventory` and have all our metadata loaded. Here goes `inventory`
```python
------------------------------
Variable   Columns   Type
------------------------------
ID            1-11   Character
LATITUDE     13-20   Real
LONGITUDE    22-30   Real
ELEMENT      32-35   Character
FIRSTYEAR    37-40   Integer
LASTYEAR     42-45   Integer
------------------------------
lines=[]
lines.append('id|lat|lon|elem|firstyear|lastyear')
with open('/Users/ranjitiyer/work/data/weather/inventory.txt', 'r') as st:
    for line in st.readlines():
        id = line[:11].strip()
        lat = line[12:20].strip()
        lon = line[21:30].strip()
        elev = line[31:35].strip()
        firstyear = line[37:40].strip()
        lastyear = line[41:45].strip()
        lines.append(f'{id}|{lat}|{lon}|{elev}|{firstyear}|{lastyear}')
        
with open('/path/to/inventory_formatted.txt','w') as st:
    st.write("\n".join(lines))
```

```sql
create table inventory
(
  id varchar(32),
  lat decimal,
  lon decimal,
  element varchar(32),
  firstyear integer,
  lastyear integer
)

copy inventory
from '/Users/ranjitiyer/work/data/weather/inventory_formatted.txt'
delimiter '|'
CSV HEADER
```

And `country` which I omitted writing about here. We finally have metadata loaded into Postgres.  Even with just metadata loaded, we can start asking some questions. It turns out weather stations have a lifespan and although `inventory` has many distinct stations many of them seem to be non-operational!

```sql
-- total weather stations
select count(distinct id) from inventory
125976

-- weather stations active in 2024
select count (distinct id) from inventory
where lastyear = 2024
34181
```

This makes me ask what is the oldest operative weather stations in the world, which turns out to be a station in Australia! 

```sql
select distinct i.id, s.name, firstyear, lastyear, lastyear - firstyear as age
from inventory i inner join station s on i.id = s.id
where lastyear=2024
order by age desc limit 1

ASN00001019	KALUMBURU	1750	2024	274
```
Wow, it's impressive but is it factual? [Wikipedia](https://en.wikipedia.org/wiki/Hohenpei%C3%9Fenberg_Meteorological_Observatory) reports a weather station in Germany to be oldest, operating from 1751 and according to the [Govt](http://www.bom.gov.au/climate/data/lists_by_element/stations.txt) of Australia `KALUMBURU` has been operative only since `1997` and yet looking at NOAA weather data from this station, we see reporting since March of 1750, so let's assume for now this is factual and move on.

```bash
head ASN00001019.csv
ASN00001019,17500301,TMAX,375,,,a,
ASN00001019,17500302,TMAX,368,,,a,
ASN00001019,17500303,TMAX,348,,,a,
ASN00001019,17500304,TMAX,359,,,a,
```

Now let's count the number of weather stations older than 100 years and still operative grouped by country. WStation IDs follow the convention where the first two letters are the Country code. Example `AEM00041194` is a weather station in the `UAE` and `AFM00040938` is one in Afghanistan. This query will require us to join the inventory table with country code to get the country name.

```sql
select c.name,
       count(*)
       from 
       (
			select distinct id from inventory
			where lastyear-firstyear >= 100
            and lastyear = 2024
        ) as a
       inner join Country c on c.code = substring(id,1,2)
       group by c.name
       order by count(*) desc

United States 	1911
Russia 			94
India 			80
Germany 		71
Australia 		66
Norway 			59
Sweden 			42       
. . .
```

These results are believable, if a country's population and size are taken into account. That's it for this post. More fun exploration in the next post. Stay tuned!
