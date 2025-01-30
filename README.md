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

## Aggregate functions

computes a single result from multiple input rows.
I.e. compute the count, sum, avg (average), max (maximum) and min (minimum)
over a set of rows.

To find the max temperature:

```sql
SELECT max(temp_lo) FROM weather;
```

To find the cities that had that computed value, we need to use a subquery:

```sql
SELECT city FROM weather
    WHERE temp_lo = (SELECT max(temp_lo) FROM weather);
```

Use a `GROUP BY` clause to get stats on combined rows.
I.e. we can get the number of readings and the maximum low temperature observed
in each city with:

```sql
SELECT city, count(*), max(temp_lo)
    FROM weather
    GROUP BY city;
```

↑ The aggregate result is computed over rows matching each city.

Combine with `HAVING` to filter group rows.
i.e. get the number of temperature readings by city that had a low temperature
below 40:

```sql
SELECT city, count(*), max(temp_lo)
    FROM weather
    GROUP BY city
    HAVING max(temp_lo) < 40;
```

### Difference between `WHERE` and `HAVING`

`WHERE` selects input rows before groups and aggregates are computed (it controls which rows go into the aggregate computation).
`HAVING` selects group rows after groups and aggregates are computed.
That's why you can't put an aggregation into a `WHERE` clause. A `HAVING` clause almost always uses an aggregate function.

### Filter

use the `FILTER` aggregation to select the rows that go into an aggregate
computation. It's a per-aggregate options.

`FILTER` is like `WHERE` except `FILTER` removes rows only from the input of
the particular aggregate function that it is attached to.

I.e. the count aggregate counts only rows with temp_lo below 45; but the max aggregate is still applied to all rows, so it still finds the reading of 46:

```sql
SELECT city, count(*) FILTER (WHERE temp_lo < 45), max(temp_lo)
    FROM weather
    GROUP BY city;
```

## Updates

use the `UPDATE` command to modify existing rows.
i.e. to lower temperature readings after November 28 by 2 degrees:

```sql
UPDATE weather
    SET temp_hi = temp_hi - 2,  temp_lo = temp_lo - 2
    WHERE date > '1994-11-28';
```

## Deletions

Remove rows from a table with `DELETE` command. i.e. to delete weather records
for the city of Hayward:

```sql
DELETE FROM weather WHERE city = 'Hayward';
```

ALl rows will be deleted if you don't provide a `WHERE` clause:

```sql
DELETE FROM tablename;
```

## Views

Allow you to save a query and allows us to refer to it like a table.
i.e. to combine the weather records and city location:

```sql
CREATE VIEW myview AS
    SELECT name, temp_lo, temp_hi, prcp, date, location
        FROM weather, cities
        WHERE city = name;
```

Then we can query it like any other table:

```sql
SELECT * FROM myview;
```

Views allow you to encapsulate the details of the structure of your tables,
which might change as your application evolves, behind consistent interfaces.

You can create views of other views.

To delete a view: `DROP VIEW [IF EXISTS] view_name [CASCADE | RESTRICT];`

## Foreign keys

Allow you to apply a constraint on a table like requiring a record to exist in
one table before being added to another table.
i.e. to ensure you can't add weather records to the weather table that don't
have a record in the citites table:

```sql
CREATE TABLE cities (
        name     varchar(80) primary key,
        location point
);

CREATE TABLE weather (
        city      varchar(80) references cities(name),
        temp_lo   int,
        temp_hi   int,
        prcp      real,
        date      date
);
```

## Transactions

Combines multiple steps into one, all or nothing operation.
The intermediate states between the steps are not visible to other concurrent
transactions, and if some failure occurs that prevents the transaction from
completing, then none of the steps affect the database at all.

Transactions are atomic to the perspective of other transactions. ALl updates
are permanent after a transaction completes.

Transactions prevent other concurrent transactions from seeing incomplete
changes.

a transaction is set up by surrounding the SQL commands of the transaction with `BEGIN` and `COMMIT` commands.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
-- etc etc
COMMIT;
```

We can issue a `ROLLBACK` command instead of a `COMMIT` to cancel changes.

You can use savepoints to get more granular control over a transaction.
Savepoints allow you to selectively discard parts of a transaction while
commiting the rest.
After defining a savepoint with `SAVEPOINT`, you can if needed roll back to the
savepoint with `ROLLBACK TO`. All the transaction's database changes between
defining the savepoint and rolling back to it are discarded, but changes
earlier than the savepoint are kept.

For example, if we removed 100 from Alice's account and add it to Bob's but
later learn we need to credit Wally's account:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
SAVEPOINT my_savepoint;
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Bob';
-- oops ... forget that and use Wally's account
ROLLBACK TO my_savepoint;
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Wally';
COMMIT;
```

