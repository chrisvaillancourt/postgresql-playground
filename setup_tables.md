# Setup tables for exercies

## Product and customer table

```sql
CREATE TABLE orders (
    order_id integer UNIQUE,
    customer_id integer,
    product_name text,
    order_total numeric
);
CREATE TABLE customers (
    customer_id integer UNIQUE,
    customer_name text
);
INSERT INTO customers (customer_id, customer_name) VALUES
    (1, 'Topher'),
    (2, 'Alex'),
    (3, 'Susan'),
    (4, 'Patricia');
INSERT INTO orders (order_id, customer_id, product_name, order_total) VALUES
    (1, 1, 'Oranges', 10),
    (2, 1, 'Bananas', 5),
    (3, 2, 'Cabbage', 2),
    (4, 3, 'Cake', 20),
    (5, 3, 'Milk', 5),
    (6, 3, 'Flour', 10),
    (7, 4, 'Steak', 40),
    (8, 4, 'Charcoal', 10),
    (9, 4, 'Hamburger', 20);
```

## Subqueries and Lateral subqueries

```sql
-- Create the customers table
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(255) NOT NULL
);

-- Create the orders table
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);

-- Insert data into the customers table
INSERT INTO customers (customer_name) VALUES
('Alice Smith'),
('Bob Johnson'),
('Charlie Brown'),
('David Lee');

-- Insert data into the orders table
INSERT INTO orders (customer_id, order_total, order_date) VALUES
(1, 50.00, '2023-10-26'),
(1, 75.00, '2023-10-27'),
(1, 25.00, '2023-10-28'),
(2, 100.00, '2023-10-25'),
(2, 150.00, '2023-10-27'),
(3, 200.00, '2023-10-24'),
(3, 50.00, '2023-10-26'),
(4, 75.00, '2023-10-25'),
(4, 125.00, '2023-10-27'),
(1, 60.00, '2023-10-29'); -- Added an order for Alice after the initial ones
```
