# PostgreSQL Turorial

start the container with `docker compose up`

connect to the db with `docker exec -u postgres -it postgresql-playground-db-1 psql`

## Setup the db

`CREATE DATABASE mydb;`
connect to it: `\c mydb`
quit psql with `\q`

## Adding data

```sql
CREATE TABLE weather (
    city            varchar(80),
    temp_lo         int,           -- low temperature
    temp_hi         int,           -- high temperature
    prcp            real,          -- precipitation
    date            date
);

CREATE TABLE cities (
    name            varchar(80),
    location        point
);
```

Delete a table with `DROP TABLE tablename;`

Add data with `INSERT` (requires knowing the order of columns)

```sql
INSERT INTO weather VALUES ('San Francisco', 46, 50, 0.25, '1994-11-27');
INSERT INTO cities VALUES ('San Francisco', '(-194.0, 53.0)');
```

Add data by spefifying column names:

```sql
INSERT INTO weather (city, temp_lo, temp_hi, prcp, date)
    VALUES ('San Francisco', 43, 57, 0.0, '1994-11-29');
INSERT INTO weather (date, city, temp_hi, temp_lo)
    VALUES ('1994-11-29', 'Hayward', 54, 37);
```

You can insert multiple rows in a single command:

```sql
INSERT INTO products (product_no, name, price) VALUES
    (1, 'Cheese', 9.99),
    (2, 'Bread', 1.99),
    (3, 'Milk', 2.99);
```

[See here for info about bulk loading](https://www.postgresql.org/docs/17/populate.html).

## Querying

Basic:

```sql
SELECT city, temp_lo, temp_hi, prcp, date FROM weather;
```

With an expression to dynamically calculate a column:

```sql
SELECT city, (temp_hi+temp_lo)/2 AS temp_avg, date FROM weather;
```

Filtering with `WHERE`:

```sql
SELECT * FROM weather
    WHERE city = 'San Francisco' AND prcp > 0.0;
```

Order with `ORDER BY`:

```sql
SELECT * FROM weather
    ORDER BY city, temp_lo;
```

Remove duplicate rows:

```sql
SELECT DISTINCT city
    FROM weather
    ORDER BY city;
```

## Joining

### Inner joins

```sql
SELECT city, temp_lo, temp_hi, prcp, date, location
    FROM weather JOIN cities ON city = name;
```

↑ only works because there isn't overlapping column names in the table.
If there's duplicate names, we need to qualify the column name with the table
name:

```sql
SELECT weather.city, weather.temp_lo, weather.temp_hi,
       weather.prcp, weather.date, cities.location
    FROM weather JOIN cities ON weather.city = cities.name;
```

It's a best practice to always qualify table names.

### Outer join

To get the rows from the weather table that don't have a corresponding row in
the cities table use an outer join. this is specifically a left outer join:

```sql
SELECT *
    FROM weather LEFT OUTER JOIN cities ON weather.city = cities.name;
```

It's called a left outer join because the table mentioned on the left of the join operator will have each of its rows in the output at least once, whereas the table on the right will only have those rows output that match some row of the left table

### Self join

WHen you join a table against itself. For example, we want to find all the
weather records that are in the temperature range of other weather records.
We need to compare the temp_lo and temp_hi columns of each weather row to the
temp_lo and temp_hi columns of all other weather rows:

```sql
SELECT w1.city, w1.temp_lo AS low, w1.temp_hi AS high,
       w2.city, w2.temp_lo AS low, w2.temp_hi AS high
    FROM weather w1 JOIN weather w2
        ON w1.temp_lo < w2.temp_lo AND w1.temp_hi > w2.temp_hi;
```

