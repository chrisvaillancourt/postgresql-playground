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

## Window functions

A window function performs a calculation across a set of table rows that are
somehow related to the current row.

Similar types of calculations to aggregate function except window functions
don't cause rows to become grouped into a single output row like non-window
aggregate calls would.

For example, to compare each employee's salary with the average salary in his
or her department:

```sql
SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY depname) FROM empsalary;
```

yields:

```
 depname  | empno | salary |          avg
-----------+-------+--------+-----------------------
 develop   |    11 |   5200 | 5020.0000000000000000
 develop   |     7 |   4200 | 5020.0000000000000000
 develop   |     9 |   4500 | 5020.0000000000000000
 develop   |     8 |   6000 | 5020.0000000000000000
 develop   |    10 |   5200 | 5020.0000000000000000
 personnel |     5 |   3500 | 3700.0000000000000000
 personnel |     2 |   3900 | 3700.0000000000000000
 sales     |     3 |   4800 | 4866.6666666666666667
 sales     |     1 |   5000 | 4866.6666666666666667
 sales     |     4 |   4800 | 4866.6666666666666667
```

↑ The first three output columns come directly from the table `empsalary`, and
there is one output row for each row in the table. The fourth column represents
an average taken across all the table rows that have the same `depname` value as
the current row.

The `OVER` clause causes the `avg` aggregate to be treated as a window function
and computed across the window frame.
A window function call always contains an OVER clause directly following the
window function's name and argument(s). That's the only way to tell it isn't a
normal aggregation.

The `OVER` clause determines exactly how the rows of the query are split up for
processing by the window function. The `PARTITION` BY clause within `OVER`
divides the rows into groups, or partitions, that share the same values of the
`PARTITION BY` expression(s). For each row, the window function is computed
across the rows that fall into the same partition as the current row.

You can control the order in which rows are processed by window functions using
`ORDER BY` within `OVER`:

```sql
SELECT depname, empno, salary,
       rank() OVER (PARTITION BY depname ORDER BY salary DESC)
FROM empsalary;
```

yields:

```
  depname  | empno | salary | rank
-----------+-------+--------+------
 develop   |     8 |   6000 |    1
 develop   |    10 |   5200 |    2
 develop   |    11 |   5200 |    2
 develop   |     9 |   4500 |    4
 develop   |     7 |   4200 |    5
 personnel |     2 |   3900 |    1
 personnel |     5 |   3500 |    2
 sales     |     1 |   5000 |    1
 sales     |     4 |   4800 |    2
 sales     |     3 |   4800 |    2
```

The the `rank` function produces a numerical rank for each distinct `ORDER BY`
value in the current row's partition, using the order defined by the `ORDER BY`
clause.

The rows considered by a window function are those of the “virtual table”
produced by the query's `FROM` clause as filtered by its `WHERE`, `GROUP BY`, and `HAVING` clauses if any. Rows excluded with a `WHERE` clause aren't seen by any window function.

When working with window functions, there's a concept of window frames: the set
of rows within its partition.
Some window functions act only on the rows of the window frame, rather than of
the whole partition.
By default, if `ORDER BY` is supplied then the frame consists of all rows from
the start of the partition up through the current row, plus any following rows
that are equal to the current row according to the ORDER BY clause
When `ORDER` BY is omitted the default frame consists of all rows in the
partition.
i.e.

```sql
SELECT salary, sum(salary) OVER () FROM empsalary
```

```
 salary |  sum
--------+-------
   5200 | 47100
   5000 | 47100
   3500 | 47100
   4800 | 47100
   3900 | 47100
   4200 | 47100
   4500 | 47100
   4800 | 47100
   6000 | 47100
   5200 | 47100
```

↑ Since we don't have a `ORDER BY` in the `OVER` clause, the window frame is
the same as the partition which is the whole table since we don't have a
`PARTITION BY`. i.e. each sum is taken over the whole table and so we get the
same result for each output row.

If we add an `ORDER BY` clause to the previous example: we get the sum is taken from the first (lowest) salary up through the current one, including any duplicates of the current one (notice the results for the duplicated salaries).

```sql
SELECT salary, sum(salary) OVER (ORDER BY salary) FROM empsalary;
```

yields:

```
 salary |  sum
--------+-------
   3500 |  3500
   3900 |  7400
   4200 | 11600
   4500 | 16100
   4800 | 25700
   4800 | 25700
   5000 | 30700
   5200 | 41100
   5200 | 41100
   6000 | 47100
```

Window functions are permitted only in the `SELECT` list and the `ORDER BY`
clause of the query.
Window function execute after non-window aggregate functions.

Use a sub-select to filter or group rows after the window calculations are
performed:

```sql
SELECT depname, empno, salary, enroll_date
FROM
  (SELECT depname, empno, salary, enroll_date,
          rank() OVER (PARTITION BY depname ORDER BY salary DESC, empno) AS pos
     FROM empsalary
  ) AS ss
WHERE pos < 3;
```

↑ only shows the rows from the inner query having rank less than 3.

We can name windowing behavior to create queries with multiple window functions:

```sql
SELECT sum(salary) OVER w, avg(salary) OVER w
  FROM empsalary
  WINDOW w AS (PARTITION BY depname ORDER BY salary DESC);
```

## Inheritance

Tables can inherit from other tables like in classical object oriented
languages. A table can inherit from multiple tables

```sql
CREATE TABLE cities (
  name       text,
  population real,
  elevation  int     -- (in ft)
);

CREATE TABLE capitals (
  state      char(2) UNIQUE NOT NULL
) INHERITS (cities);
```

↑ `capitals` inherits all columns from `cities`.

The following returns all cities (incliuding capitals):

```sql
SELECT name, elevation
  FROM cities
  WHERE elevation > 500;
```

The following finds all non-capital cities and are situated
at an elevation over 500 feet:

```sql
SELECT name, elevation
    FROM ONLY cities
    WHERE elevation > 500;
```

The ONLY before cities indicates that the query should be run over only the
cities table, and not tables below cities in the inheritance hierarchy.
Inheritance hasn't been integrated with unique contraints or foreign keys.
See [inheritance caveats](https://www.postgresql.org/docs/17/ddl-inherit.html).

<!-- the SQL language -->

## identifiers and keywords

[Complete list of keywords](https://www.postgresql.org/docs/17/sql-keywords-appendix.html)

Key words and unquoted identifiers are case-insensitive.

these are functionally identical:

```sql
UPDATE MY_TABLE SET A = 5;
uPDaTE my_TabLE SeT a = 5;
```

It's convention to use upper case for key words and names in lower case.
Anything inside of quotes (`""`) is a deliminated identifier (name).
i.e. `"select"` could be used as a table name but `select` will always be
treated as a keyword.

Quoting an identifier also makes it case-sensitive, whereas unquoted names are always folded to lower case.

## Default values

Column's can have a default value so that one is assigned if not provided.

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric DEFAULT 9.99
);
```

The default value can be an expression, which will be evaluated whenever the default value is inserted (not when the table is created). A common example is for a timestamp column to have a default of `CURRENT_TIMESTAMP`, so that it gets set to the time of row insertion.

Another common example is generating a “serial number” for each row. In PostgreSQL this is typically done by something like:

```sql
CREATE TABLE products (
    product_no integer DEFAULT nextval('products_product_no_seq'),
    name text,
    price numeric DEFAULT 9.99,
);
```

The pattern of using a `nextval()` function supplies successive values from a
sequence object so we can do:

```sql
CREATE TABLE products (
    product_no SERIAL,
    name text,
    price numeric DEFAULT 9.99
);
```

`INSERT INTO  products (name) VALUES ('apples');`

```sql
SELECT * FROM products;
```

yields:

```
product_no |  name  | price
------------+--------+-------
          1 | apples |  9.99
```

```sql
INSERT INTO  products (name, price) VALUES ('oranges', 12.99);
```

yields:

```
 product_no |  name   | price
------------+---------+-------
          1 | apples  |  9.99
          2 | oranges | 12.99
```

[More on serial](https://www.postgresql.org/docs/17/datatype-numeric.html#DATATYPE-SERIAL).

## Identity columns

An identity column is a special column that is generated automatically from an
implicit sequence. It can be used to generate key values.

To create an identity column, use the `GENERATED ... AS IDENTITY` clause in
`CREATE TABLE`, for example:

```sql
CREATE TABLE people (
    id bigint GENERATED ALWAYS AS IDENTITY,
    name text,
    address text
);
```

Or:

```sql
CREATE TABLE people (
    id bigint GENERATED BY DEFAULT AS IDENTITY,
    name text,
    address text
);
```

The `DEFAULT` keyword can be used when inserting or updating to explicitly
request a generatred value:

```sql
INSERT INTO people (id, name, address) VALUES (DEFAULT, 'C', 'baz');
```

The clauses `ALWAYS` and `BY DEFAULT` in the column definition determine how
explicitly user-specified values are handled in `INSERT` and `UPDATE` commands.
In an `INSERT` command, if `ALWAYS` is selected, a user-specified value is only
accepted if the `INSERT` statement specifies `OVERRIDING SYSTEM VALUE`.
If `BY DEFAULT` is selected, then the user-specified value takes precedence.

Using `BY DEFAULT` results in a behavior more similar to default values, where
the default value can be overridden by an explicit value, whereas `ALWAYS`
provides some more protection against accidentally inserting an explicit value.

An identity column does not guarantee uniqueness. Uniqueness needs to be
enforced using a `PRIMARY KEY` or `UNIQUE` constraint.

In table inheritance hierarchies, identity columns and their properties in a
child table are independent of those in its parent tables. A child table does
not inherit identity columns or their properties automatically from the parent.

## Generated columns

A generated column is a special column that is always computed from other
columns. It's like table views except for columns.
There are two types of generated columns: stored and virtual.

A stored generated column is computed when it is written (inserted or updated)
and occupies storage as if it were a normal column.
A virtual generated column occupies no storage and is computed when it is read.
Thus, a virtual generated column is similar to a view and a stored generated
column is similar to a materialized view (except that it is always updated
automatically). **PostgreSQL currently implements only stored generated columns**.

To create a generated column, use the `GENERATED ALWAYS AS` clause in
`CREATE TABLE`:

```sql
CREATE TABLE people (
    id bigint GENERATED ALWAYS AS IDENTITY,
    name text,
    address text,
    height_cm numeric,
    height_in numeric GENERATED ALWAYS AS (height_cm / 2.54) STORED
);
```

```sql
INSERT INTO people (name, address, height_cm) VALUES ('A', 'foo', 182.8);
SELECT * FROM people;
```

yields:

```
 id | name | address | height_cm |      height_in
----+------+---------+-----------+---------------------
  1 | A    | foo     |     182.8 | 71.9685039370078740
```

A generated column is updated whenever the row changes and cannot be overridden.
There are several restrictions on generated columns.

## Constraints

Allow us to set allowed limits to a column or constrain a column's data with
respect to other columns.

A check constraint is the most generic constraint type. It allows you to
specify that the value in a certain column must satisfy a Boolean (truth-value)
expression. For instance, to require positive product prices, you could use:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0)
);
```

You can also give the constraint a separate name. This clarifies error messages
and allows you to refer to the constraint when you need to change it. To
specify a named constraint, use the key word `CONSTRAINT` followed by an
identifier followed by the constraint definition.

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CONSTRAINT positive_price CHECK (price > 0)
);
```

A check constraint can also refer to several columns. Say you store a regular
price and a discounted price, and you want to ensure that the discounted price
is lower than the regular price:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0),
    discounted_price numeric CHECK (discounted_price > 0),
    CHECK (price > discounted_price)
);
```

↑ The third constraint is a table constraint because it is written separately
from any one column definition. There are multiple ways to define a constraint. these are the same:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price > 0 AND price > discounted_price)
);
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price > 0),
    CHECK (price > discounted_price)
);
```

Example assigning a name to a table constraint:

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price > 0),
    CONSTRAINT valid_discount CHECK (price > discounted_price)
);
```

A check constraint is satisfied if the check expression evaluates to true or
the null value. Since most expressions will evaluate to the null value if any
operand is null, they will not prevent null values in the constrained columns.
To ensure that a column does not contain null values, you must use the not-null
constraint.

## not-null constraints

A not-null constraint simply specifies that a column must not assume the null
value:

```sql
CREATE TABLE products (
    product_no integer NOT NULL,
    name text NOT NULL,
    price numeric
);
```

A column can have multiple constraints, one listed after another:

```sql
CREATE TABLE products (
    product_no integer NOT NULL,
    name text NOT NULL,
    price numeric NOT NULL CHECK (price > 0)
);
```

## Unique constraints

ensure that the data contained in a column, or a group of columns, is unique among all the rows in the table:

```sql
CREATE TABLE products (
    product_no integer UNIQUE,
    name text,
    price numeric
);
```

You can define a unique constraint for a group of columns. The uniqueness
applies to the combination of values listed, not the individual columns.

```sql
CREATE TABLE example (
    a integer,
    b integer,
    c integer,
    UNIQUE (a, c)
);
```

You can assign a name to a unique constraint:

```sql
CREATE TABLE products (
    product_no integer CONSTRAINT must_be_different UNIQUE,
    name text,
    price numeric
);
```

In general, a unique constraint is violated if there is more than one row in
the table where the values of all of the columns included in the constraint are
equal. By default, two null values are not considered equal in this comparison.
That means even in the presence of a unique constraint it is possible to store
duplicate rows that contain a null value in at least one of the constrained
columns. This behavior can be changed by adding the clause
`NULLS NOT DISTINCT`:

```sql
CREATE TABLE products (
    product_no integer UNIQUE NULLS NOT DISTINCT,
    name text,
    price numeric
);
```

## Primary keys

A primary key constraint indicates that a column, or group of columns, can be
used as a unique identifier for rows in the table. Only one can exist per table.

```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);
```

## Foreign keys

A foreign key constraint specifies that the values in a column (or a group of
columns) must match the values appearing in some row of another table.
A foreign key maintains the referential integrity between two related tables.

Given a products table:

```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);
```

We can ensure the `orders` table only contains order for products that exist in
the `products` table:

```sql
CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products (product_no),
    quantity integer
);
```

↑ the orders table is the _referencing_ table and the products table is the
_referenced_ table.
You can use a shorter syntax that drops the column list (the primary key of the
referenced table will be assumed):

```sql
CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products,
    quantity integer
);
```

You can use a column in the same table as a foreign key constraint (a self-referential foreign key).

```sql
CREATE TABLE tree (
    node_id integer PRIMARY KEY,
    parent_id integer REFERENCES tree,
    name text
);
```

↑ A top-level node would have NULL parent_id, while non-NULL parent_id entries
would be constrained to reference valid rows of the table.

A table can have more than one foreign key constraint. This is used to implement many-to-many relationships between tables.

For example, you have tables about products and orders, but now you want to
allow one order to contain possibly many products (which the structure above
did not allow):

```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    shipping_address text
);

CREATE TABLE order_items (
    product_no integer REFERENCES products,
    order_id integer REFERENCES orders,
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);
```

What if we want to remove a product after an order is created that references
it? For example, let's implement the following policy on the many-to-many
relationship example above: when someone wants to remove a product that is
still referenced by an order (via order_items), we disallow it. If someone
removes an order, the order items are removed as well:

```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    shipping_address text,
    ...
);

CREATE TABLE order_items (
    product_no integer REFERENCES products ON DELETE RESTRICT,
    order_id integer REFERENCES orders ON DELETE CASCADE,
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);
```

`RESTRICT` prevents deletion of a referenced row.
`NO ACTION` means that if any referencing rows still exist when the constraint
is checked, an error is raised; this is the default behavior if you do not
specify anything.
`CASCADE` specifies that when a referenced row is deleted, row(s) referencing
it should be automatically deleted as well.
Instead of `CASCADE`, we could use `SET NULL` and `SET DEFAULT`.

The appropriate choice of `ON DELETE` action depends on what kinds of objects
the related tables represent. When the referencing table represents something
that is a component of what is represented by the referenced table and cannot
exist independently, then `CASCADE` could be appropriate. If the two tables
represent independent objects, then `RESTRICT` or `NO ACTION` is more appropriate;
an application that actually wants to delete both objects would then have to be
explicit about this and run two delete commands.

The actions `SET NULL` or `SET DEFAULT` can be appropriate if a foreign-key relationship represents optional information

`SET NULL` and `SET DEFAULT` can take a column list to specify which columns to
set. Normally, all columns of the foreign-key constraint are set; setting only
a subset is useful in some special cases:

```sql
CREATE TABLE tenants (
    tenant_id integer PRIMARY KEY
);

CREATE TABLE users (
    tenant_id integer REFERENCES tenants ON DELETE CASCADE,
    user_id integer NOT NULL,
    PRIMARY KEY (tenant_id, user_id)
);

CREATE TABLE posts (
    tenant_id integer REFERENCES tenants ON DELETE CASCADE,
    post_id integer NOT NULL,
    author_id integer,
    PRIMARY KEY (tenant_id, post_id),
    FOREIGN KEY (tenant_id, author_id) REFERENCES users ON DELETE SET NULL (author_id)
);
```

↑ Without the specification of the column, the foreign key would also set the
column `tenant_id` to null, but that column is still required as part of the
primary key.

There's also `ON UPDATE` which is invoked when a referenced column is changed.
Combining `ON UPDATE` with `CASCADE` means that the updated values of the
referenced column(s) should be copied into the referencing row(s).

## Modifying / Altering tables

### Adding a table

```sql
ALTER TABLE products ADD COLUMN description text;
```

With constraints:

```sql
ALTER TABLE products ADD COLUMN description text CHECK (description <> '');
```

### Removing a column

```sql
ALTER TABLE products DROP COLUMN description;
```

If the column is referenced by a foreign key constraint of another table, postgresql will silently drop the constraint. To drop everything that depends on the column:

```sql
ALTER TABLE products DROP COLUMN description CASCADE;
```

### Adding a constraint

```sql
ALTER TABLE products ADD CHECK (name <> '');
ALTER TABLE products ADD CONSTRAINT some_name UNIQUE (product_no);
ALTER TABLE products ADD FOREIGN KEY (product_group_id)
  REFERENCES product_groups;
```

To add a not-null constraint, which cannot be written as a table constraint:

```sql
ALTER TABLE products ALTER COLUMN product_no SET NOT NULL;
```

To remove a constraint, you need to know it's name. If you provided one
initially:

```sql
ALTER TABLE products DROP CONSTRAINT some_name;
```

You need to add CASCADE if you want to drop a constraint that something else
depends on. An example is that a foreign key constraint depends on a unique or
primary key constraint on the referenced column(s).

### Changing a Column's Default Value

Set a new default value:

```sql
ALTER TABLE products ALTER COLUMN price SET DEFAULT 7.77;
```

↑ Only changes the default for future INSERT commands.

### Changing a columns data type

```sql
ALTER TABLE products ALTER COLUMN price TYPE numeric(10,2);
```

↑ only works if the conversion can be done with an implicit cast. Otherwise neeed to use a `USING` clause.

### Rename a column

```sql
ALTER TABLE products RENAME COLUMN product_no TO product_number;
```

### Renaming a table

```sql
ALTER TABLE products RENAME TO items;
```

## Privileges

When an object is created, it is assigned an owner. The owner is normally the
role that executed the creation statement. For most kinds of objects, the
initial state is that only the owner (or a superuser) can do anything with the
object. To allow other roles to use it, privileges must be granted.

[See more here](https://www.postgresql.org/docs/17/ddl-priv.html).

