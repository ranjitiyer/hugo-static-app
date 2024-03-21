+++
title = 'Dates in Postgres'
date = 2024-03-19T21:17:15-07:00
+++

> ***Time is a fundamental aspect of our perception. Time is a measure in which events can be ordered from the past through the present into the future, and also the measure of the duration of events and the intervals between them.***

Time and Date are fundamntal types software engineers need to work with in programming langauges and software systems. In this post we'll explore functions in Postgres SQL to work with date, time and durations. Let's start with the simplest example of asking the system for the current time.

```sql
select current_time
04:41:53.718402+00
```
It's 21:43 PST at the time of this writing, which means we're given current time in UTC. Can we ask for the time in PST? Sure, we can 
use the `AT TIME ZONE` [operator](https://www.postgresql.org/docs/14/functions-datetime.html#FUNCTIONS-DATETIME-ZONECONVERT) to convert from one time zone to another.

```sql
select current_time AT TIME ZONE 'America/Los_Angeles'
or
select current_time at TIME ZONE 'PDT' -- Pacfic daylight
```
Now we get California time. `PDT` is Pacific Daylight Time and if this feels like a magic string, a full list of supported time zone names can be found [here](https://www.postgresql.org/docs/7.2/timezones.html). Let's apply this learning to dates. Date is just the date without time and there's a specific function to return it.

```sql
select current_date
2024-03-20
```
Although it 3/19 at the time of this writing, we get 3/20 since it's UTC. Let's apply the time zone operator.

```sql
select current_date at TIME ZONE 'America/Los_Angeles'
2024-03-19 17:00:00.000000
```
Hmm, It is `22:01` now but we're getting `17:00` and I'm not sure why, but we'll park that for now and keep going.

Postgres represents date and time as the data type `timestamp` and we can call a function to give us this value

```sql
select current_timestamp
2024-03-20 05:05:54.914408 +00:00

select current_timestamp at time zone 'PDT'
2024-03-19 22:04:11.228748
```

Finally, there are functions that return local time

```sql
select localtime
select localtimestamp
```

**Date parts**

Often we need to extract parts of a date or timestamp to, say, count records per hour in a given day. Postgres offers the flexible `extract` function or the more SQL compliant `date_part`. Let's look at both. 

Let's get the day, month, year and more.

```sql
select date_part('year', current_timestamp)
select date_part('month', current_timestamp)
select date_part('day', current_timestamp)
select date_part('hour', current_timestamp)
select date_part('minute', current_timestamp)
select date_part('seconds', current_timestamp)
```
`Extract` is not a SQL standard allows using year, month, day, etc. as symbolic constants without needing to specify then as strings. It also makes code more readable. Examples:

```sql
select extract(day from current_date)
select extract(hour from current_timestamp)
```

**Interval**

Java has `Duration`, C# has `TimeSpan` and Postgres has `Interval` to represents a duration of time. Let's say it's 10 years, 3 months and 5 days since we bought our house, this can be represented as an Interval like so

```sql
select '10 years 3 months 5 days'::interval
10 years 3 mons 5 days 0 hours 0 mins 0.0 secs
```

`date_part` and `extract` can operate on `Intervals` too

```sql
select date_part('months', '10 years 3 months 5 days'::interval)
3

select extract(month from '10 years 3 months 5 days'::interval)
3
```

**Date math**

Now that we've looked at representing and extracting component parts of a date/timestamp we turn our focus to date math for time window search queries. For example, count something between some date in the past and now.

Adding days can be done two ways

```sql
select current_date + 1
select current_date + '1 day'::interval;
```

Likewise for subtracting days

```sql
select current_date - 1
select current_date - '1 day'::interval
```

Let's go one back year
```sql
select current_date - '1 year'::interval
```

Let's make a date that is the first day of the current month from one year ago.
```sql
select (date_part('year', current_date - '1 year'::interval) || '-' || date_part('month', current_date) || '-' || '01')::date
```

or more simply using `date_trunc` which rounds the given timestamp or interval to the nearest unit specified. We can get to the top of the hour or first day of the month, or first day of the year, etc.

```sql
select date_trunc('year', current_date - '1 year'::interval)
```

**Casting**

We can cast date and time values using `select cast(<expression> as type)` or using the convenient the more convenient `expression::type` syntax. Say we have dates in a textual form we would use

```sql
select cast('2023-03-01' as time)
Error
```
This is an error because the text value doesn't have a time component. 

```sql
select cast('2023-03-01' as date)
Works
```

To cast to time we'd need this

```sql
select cast('2023-03-01 00:00:00' as time)
00:00:00
```

**Formatting**

In the real world dates can come in different formats and we need to bring them into a standard date, time and timestamp values. Here's an example.

```sql
select to_timestamp('03/19/2024 15:30:45', 'DD/MM/YYYY HH24:MI:SS');

select to_timestamp('03/19/2024 15:30:45', 'DD/MM/YYYY HH24:MI:SS');

select to_date('2024-03-19', 'YYYY-MM-DD');
```
Full list of format specifiers can be found [here](https://www.postgresql.org/docs/14/functions-formatting.html#FUNCTIONS-FORMATTING-DATETIME-TABLE)





