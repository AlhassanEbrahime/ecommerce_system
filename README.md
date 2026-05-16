# E-Commerce Database

A relational database schema for an e-commerce platform, with PostgreSQL and MySQL DDL plus SQL operations for search, recommendations, reporting, and sale history automation.

## Contents

### Database Design

- [ERD](#erd)
- [DDL](#ddl)

### Operations

| Task Name | Description |
|---|---|
| [Daily Revenue Report](#daily-revenue-report) | Calculates order count, sold items, and revenue for a selected date. |
| [Monthly Top-Selling Products](#monthly-top-selling-products) | Lists the top 10 products by quantity sold in a selected month. |
| [High-Value Customers](#high-value-customers-last-30-days) | Finds customers who spent more than 500 in the last 30 days. |
| [Task 1: Search Products by Keyword](#task-1-search-products-by-keyword) | Searches products by name or description using a case-insensitive keyword. |
| [Task 2: Suggest Popular Products from the Same Category](#task-2-suggest-popular-products-from-the-same-category) | Recommends available products from the same category as a purchased product. |
| [Task 3: Create Sale History Table](#task-3-create-sale-history-table) | Creates a denormalized table for historical sale records. |
| [Task 4: Create Indexes for Sale History](#task-4-create-indexes-for-sale-history) | Adds indexes to improve sale history query performance. |
| [Task 5: Create Sale History Trigger Function](#task-5-create-sale-history-trigger-function) | Builds a PostgreSQL trigger function that inserts sale history rows. |
| [Task 6: Create Trigger](#task-6-create-trigger) | Attaches the sale history trigger to `ORDER_DETAILS`. |
| [Task 7: Test the Trigger](#task-7-test-the-trigger) | Inserts sample order data and verifies that sale history is created. |
| [Task 8: Lock Product Quantity and Row](#task-8-lock-product-quantity-and-row) | Uses transactions to prevent concurrent updates to product ID 211. |

---

## Database Design

### ERD

The database supports product cataloguing, hierarchical categories, customer accounts, orders, and order line items.

| Table | Purpose |
|---|---|
| `CATEGORY` | Hierarchical product categories with optional parent categories. |
| `PRODUCT` | Product catalogue with pricing, descriptions, stock, and category links. |
| `CUSTOMER` | Registered customer accounts. |
| `ORDERS` | Order headers placed by customers. |
| `ORDER_DETAILS` | Line items for products inside each order. |

IDs start at 100 in both implementations. Soft delete using `IS_DELETED` is used on `CATEGORY` and `ORDERS` to preserve audit history.

```text
+-----------------------+
|       CATEGORY        |
+-----------------------+
| PK  CATEGORY_ID  int  |<-------------------------------+
| FK  PARENT_ID    int? |--------------------------------+  self-reference
|     CATEGORY_NAME     |
|     IS_DELETED        |
+-----------+-----------+
            | 1
            | ON DELETE SET NULL
            | 0..*
+-----------v-----------+
|        PRODUCT        |
+-----------------------+
| PK  PRODUCT_ID   int  |<-----------------------------------------+
| FK  CATEGORY_ID  int? |                                          |
|     NAME              |                                          |
|     DESCRIPTION       |                                          |
|     PRICE    dec(6,2) |                                          |
|     STOCK_QTY    int  |                                          |
+-----------------------+                                          |
                                                                   | 0..*
+-----------------------+           +-----------------------------------------------+
|       CUSTOMER        |           |                ORDER_DETAILS                  |
+-----------------------+           +-----------------------------------------------+
| PK  CUSTOMER_ID  int  |<--+       | PK  ORDER_DETAIL_ID  int                      |
|     FIRST_NAME        |   |       | FK  ORDER_ID          int  --> ORDERS          |
|     LAST_NAME         |   |       | FK  PRODUCT_ID        int  --> PRODUCT         |
|     EMAIL    unique   |   |       |     QUANTITY           int                     |
|     PASSWORD          |   |       |     UNIT_PRICE   dec(6,2)                      |
|     IS_ACTIVE         |   |       +-----------------------------------------------+
+-----------------------+   |                        ^
                            | 1                      | 1..*
                            | ON DELETE RESTRICT     |
                            | 0..*                   |
                   +--------+----------------+       |
                   |         ORDERS          |-------+
                   +-------------------------+
                   | PK  ORDER_ID       int  |
                   | FK  CUSTOMER_ID    int  |
                   |     ORDER_DATE    date  |
                   |     TOTAL_AMOUNT dec(10,2)
                   |     IS_DELETED  boolean |
                   +-------------------------+
```

| Relationship | Type | Rule on Delete |
|---|---|---|
| `CATEGORY` to `CATEGORY` | 1:N | RESTRICT |
| `PRODUCT` to `CATEGORY` | N:1 | SET NULL |
| `ORDERS` to `CUSTOMER` | N:1 | RESTRICT |
| `ORDER_DETAILS` to `ORDERS` | N:1 | CASCADE implicitly |
| `ORDER_DETAILS` to `PRODUCT` | N:1 | CASCADE implicitly |

```text
CATEGORY   --< CATEGORY        self-referencing, PARENT_ID, RESTRICT
CATEGORY   >-- PRODUCT         1 category to many products, SET NULL
CUSTOMER   --< ORDERS          1 customer to many orders, RESTRICT
ORDERS     --< ORDER_DETAILS   1 order to many line items
PRODUCT    --< ORDER_DETAILS   1 product to many order lines
```

### DDL

#### PostgreSQL

##### CATEGORY

```sql
CREATE TABLE CATEGORY
(
    CATEGORY_ID   INTEGER,
    PARENT_ID     INTEGER,
    CATEGORY_NAME VARCHAR(100),
    IS_DELETED    BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (CATEGORY_ID),
    FOREIGN KEY (PARENT_ID) REFERENCES CATEGORY ON DELETE RESTRICT
);
```

##### PRODUCT

```sql
CREATE TABLE PRODUCT
(
    PRODUCT_ID  INTEGER,
    CATEGORY_ID INTEGER,
    NAME        VARCHAR(100)  NOT NULL,
    DESCRIPTION VARCHAR(250),
    PRICE       NUMERIC(6, 2) NOT NULL CHECK (PRICE > 0 AND PRICE <= 9999.99),
    STOCK_QTY   INTEGER       NOT NULL CHECK (STOCK_QTY >= 0),

    PRIMARY KEY (PRODUCT_ID),
    FOREIGN KEY (CATEGORY_ID) REFERENCES CATEGORY ON DELETE SET NULL
);
```

##### CUSTOMER

```sql
CREATE TABLE CUSTOMER
(
    CUSTOMER_ID INTEGER,
    FIRST_NAME  VARCHAR(20)        NOT NULL,
    LAST_NAME   VARCHAR(20),
    EMAIL       VARCHAR(50) UNIQUE NOT NULL,
    PASSWORD    VARCHAR(100)       NOT NULL,
    IS_ACTIVE   BOOLEAN            NOT NULL DEFAULT FALSE,
    PRIMARY KEY (CUSTOMER_ID)
);
```

##### ORDERS

```sql
CREATE TABLE ORDERS
(
    ORDER_ID     INTEGER,
    CUSTOMER_ID  INTEGER        NOT NULL,
    ORDER_DATE   DATE                    DEFAULT CURRENT_DATE NOT NULL,
    TOTAL_AMOUNT NUMERIC(10, 2) NOT NULL CHECK (TOTAL_AMOUNT > 0),
    IS_DELETED   BOOLEAN        NOT NULL DEFAULT FALSE,
    PRIMARY KEY (ORDER_ID),
    FOREIGN KEY (CUSTOMER_ID) REFERENCES CUSTOMER ON DELETE RESTRICT
);
```

##### ORDER_DETAILS

```sql
CREATE TABLE ORDER_DETAILS
(
    ORDER_DETAIL_ID INTEGER,
    ORDER_ID        INTEGER       NOT NULL,
    PRODUCT_ID      INTEGER       NOT NULL,
    QUANTITY        INTEGER       NOT NULL CHECK (QUANTITY > 0),
    UNIT_PRICE      NUMERIC(6, 2) NOT NULL CHECK (UNIT_PRICE > 0 AND UNIT_PRICE <= 9999.99),
    PRIMARY KEY (ORDER_DETAIL_ID),
    FOREIGN KEY (ORDER_ID) REFERENCES ORDERS,
    FOREIGN KEY (PRODUCT_ID) REFERENCES PRODUCT
);
```

#### MySQL

##### CATEGORY

```sql
CREATE TABLE CATEGORY
(
    CATEGORY_ID   INTEGER AUTO_INCREMENT,
    PARENT_ID     INTEGER,
    CATEGORY_NAME VARCHAR(100),
    IS_DELETED    TINYINT(1) NOT NULL DEFAULT FALSE,
    PRIMARY KEY (CATEGORY_ID),
    FOREIGN KEY (PARENT_ID) REFERENCES CATEGORY (CATEGORY_ID) ON DELETE RESTRICT
);
```

##### PRODUCT

```sql
CREATE TABLE PRODUCT
(
    PRODUCT_ID  INTEGER AUTO_INCREMENT,
    CATEGORY_ID INTEGER,
    NAME        VARCHAR(100)  NOT NULL,
    DESCRIPTION VARCHAR(250),
    PRICE       DECIMAL(6, 2) NOT NULL CHECK (PRICE > 0 AND PRICE <= 9999.99),
    STOCK_QTY   INTEGER       NOT NULL CHECK (STOCK_QTY >= 0),

    PRIMARY KEY (PRODUCT_ID),
    FOREIGN KEY (CATEGORY_ID) REFERENCES CATEGORY (CATEGORY_ID) ON DELETE SET NULL
);
```

##### CUSTOMER

```sql
CREATE TABLE CUSTOMER
(
    CUSTOMER_ID INTEGER AUTO_INCREMENT,
    FIRST_NAME  VARCHAR(20)        NOT NULL,
    LAST_NAME   VARCHAR(20),
    EMAIL       VARCHAR(50) UNIQUE NOT NULL,
    PASSWORD    VARCHAR(100)       NOT NULL,
    IS_ACTIVE   TINYINT(1)         NOT NULL DEFAULT FALSE,
    PRIMARY KEY (CUSTOMER_ID)
);
```

##### ORDERS

```sql
CREATE TABLE ORDERS
(
    ORDER_ID     INTEGER AUTO_INCREMENT,
    CUSTOMER_ID  INTEGER        NOT NULL,
    ORDER_DATE   DATE           NOT NULL DEFAULT (CURRENT_DATE),
    TOTAL_AMOUNT DECIMAL(10, 2) NOT NULL CHECK (TOTAL_AMOUNT > 0),
    IS_DELETED   TINYINT(1)     NOT NULL DEFAULT FALSE,
    PRIMARY KEY (ORDER_ID),
    FOREIGN KEY (CUSTOMER_ID) REFERENCES CUSTOMER (CUSTOMER_ID) ON DELETE RESTRICT
);
```

##### ORDER_DETAILS

```sql
CREATE TABLE ORDER_DETAILS
(
    ORDER_DETAIL_ID INTEGER AUTO_INCREMENT,
    ORDER_ID        INTEGER       NOT NULL,
    PRODUCT_ID      INTEGER       NOT NULL,
    QUANTITY        INTEGER       NOT NULL CHECK (QUANTITY > 0),
    UNIT_PRICE      DECIMAL(6, 2) NOT NULL CHECK (UNIT_PRICE > 0 AND UNIT_PRICE <= 9999.99),
    PRIMARY KEY (ORDER_DETAIL_ID),
    FOREIGN KEY (ORDER_ID) REFERENCES ORDERS (ORDER_ID),
    FOREIGN KEY (PRODUCT_ID) REFERENCES PRODUCT (PRODUCT_ID)
);
```

#### Indexes

```sql
CREATE INDEX idx_orders_date           ON ORDERS        (ORDER_DATE);
CREATE INDEX idx_category_parent       ON CATEGORY      (PARENT_ID);
CREATE INDEX idx_orders_customer       ON ORDERS        (CUSTOMER_ID);
CREATE INDEX idx_product_category      ON PRODUCT       (CATEGORY_ID);
CREATE INDEX idx_order_details_order   ON ORDER_DETAILS (ORDER_ID);
CREATE INDEX idx_order_details_product ON ORDER_DETAILS (PRODUCT_ID);
```

| Index | Table | Column | Purpose |
|---|---|---|---|
| `idx_orders_date` | `ORDERS` | `ORDER_DATE` | Fast date-range filtering in reports. |
| `idx_category_parent` | `CATEGORY` | `PARENT_ID` | Hierarchical category traversal. |
| `idx_orders_customer` | `ORDERS` | `CUSTOMER_ID` | Lookup all orders for a customer. |
| `idx_product_category` | `PRODUCT` | `CATEGORY_ID` | Listing products by category. |
| `idx_order_details_order` | `ORDER_DETAILS` | `ORDER_ID` | Fetching line items for an order. |
| `idx_order_details_product` | `ORDER_DETAILS` | `PRODUCT_ID` | Finding orders containing a product. |

#### Auto-Increment Configuration

PostgreSQL identity columns start at 100:

```sql
ALTER TABLE CATEGORY
    ALTER COLUMN CATEGORY_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);

ALTER TABLE PRODUCT
    ALTER COLUMN PRODUCT_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);

ALTER TABLE CUSTOMER
    ALTER COLUMN CUSTOMER_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);

ALTER TABLE ORDERS
    ALTER COLUMN ORDER_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);

ALTER TABLE ORDER_DETAILS
    ALTER COLUMN ORDER_DETAIL_ID ADD GENERATED ALWAYS AS IDENTITY (START WITH 100);
```

MySQL IDs start at 1 by default. To start at 100:

```sql
ALTER TABLE CATEGORY      AUTO_INCREMENT = 100;
ALTER TABLE PRODUCT       AUTO_INCREMENT = 100;
ALTER TABLE CUSTOMER      AUTO_INCREMENT = 100;
ALTER TABLE ORDERS        AUTO_INCREMENT = 100;
ALTER TABLE ORDER_DETAILS AUTO_INCREMENT = 100;
```

Set the InnoDB engine explicitly for foreign key support:

```sql
ALTER TABLE CATEGORY      ENGINE = InnoDB;
ALTER TABLE PRODUCT       ENGINE = InnoDB;
ALTER TABLE CUSTOMER      ENGINE = InnoDB;
ALTER TABLE ORDERS        ENGINE = InnoDB;
ALTER TABLE ORDER_DETAILS ENGINE = InnoDB;
```

#### Table Reference

##### CATEGORY

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `CATEGORY_ID` | `INTEGER` | NO | auto | Primary key. |
| `PARENT_ID` | `INTEGER` | YES | `NULL` | Foreign key to `CATEGORY(CATEGORY_ID)`. |
| `CATEGORY_NAME` | `VARCHAR(100)` | YES | `NULL` | Category name. |
| `IS_DELETED` | `BOOLEAN` | NO | `FALSE` | Soft-delete flag. |

##### PRODUCT

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `PRODUCT_ID` | `INTEGER` | NO | auto | Primary key. |
| `CATEGORY_ID` | `INTEGER` | YES | `NULL` | Foreign key to `CATEGORY(CATEGORY_ID)`. |
| `NAME` | `VARCHAR(100)` | NO | required | Product name. |
| `DESCRIPTION` | `VARCHAR(250)` | YES | `NULL` | Product description. |
| `PRICE` | `NUMERIC(6,2)` | NO | required | Must be greater than 0 and no more than 9999.99. |
| `STOCK_QTY` | `INTEGER` | NO | required | Must be at least 0. |

##### CUSTOMER

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `CUSTOMER_ID` | `INTEGER` | NO | auto | Primary key. |
| `FIRST_NAME` | `VARCHAR(20)` | NO | required | Customer first name. |
| `LAST_NAME` | `VARCHAR(20)` | YES | `NULL` | Customer last name. |
| `EMAIL` | `VARCHAR(50)` | NO | required | Unique customer email. |
| `PASSWORD` | `VARCHAR(100)` | NO | required | Store hashed passwords only. |
| `IS_ACTIVE` | `BOOLEAN` | NO | `FALSE` | Account activation flag. |

##### ORDERS

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `ORDER_ID` | `INTEGER` | NO | auto | Primary key. |
| `CUSTOMER_ID` | `INTEGER` | NO | required | Foreign key to `CUSTOMER(CUSTOMER_ID)`. |
| `ORDER_DATE` | `DATE` | NO | `CURRENT_DATE` | Order date. |
| `TOTAL_AMOUNT` | `NUMERIC(10,2)` | NO | required | Must be greater than 0. |
| `IS_DELETED` | `BOOLEAN` | NO | `FALSE` | Soft-delete flag. |

##### ORDER_DETAILS

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `ORDER_DETAIL_ID` | `INTEGER` | NO | auto | Primary key. |
| `ORDER_ID` | `INTEGER` | NO | required | Foreign key to `ORDERS(ORDER_ID)`. |
| `PRODUCT_ID` | `INTEGER` | NO | required | Foreign key to `PRODUCT(PRODUCT_ID)`. |
| `QUANTITY` | `INTEGER` | NO | required | Must be greater than 0. |
| `UNIT_PRICE` | `NUMERIC(6,2)` | NO | required | Captures price at purchase time. |

#### PostgreSQL vs MySQL Key Differences

| Feature | PostgreSQL | MySQL |
|---|---|---|
| Auto-increment | `GENERATED ALWAYS AS IDENTITY` | `AUTO_INCREMENT` |
| Boolean type | Native `BOOLEAN` | `TINYINT(1)` |
| Decimal type | `NUMERIC(p, s)` | `DECIMAL(p, s)` |
| Date truncation | `DATE_TRUNC('month', date)` | `YEAR(date)` and `MONTH(date)` |
| Interval syntax | `INTERVAL '1 month'` | `INTERVAL 1 MONTH` |
| HAVING alias | Repeat aggregate expression | Alias can be referenced directly |
| Default date | `DEFAULT CURRENT_DATE` | `DEFAULT (CURRENT_DATE)` |

#### Design Decisions

Soft-delete on `ORDERS` and `CATEGORY` preserves audit history and avoids breaking order history. Reports filter with `WHERE is_deleted = FALSE`.

`UNIT_PRICE` is stored in `ORDER_DETAILS` so historical order data remains correct even when `PRODUCT.PRICE` changes.

`TOTAL_AMOUNT` is denormalized on `ORDERS` to avoid recalculating order totals for common reporting reads.

`CUSTOMER` deletion is restricted when orders exist. Customers should be deactivated with `IS_ACTIVE = FALSE` instead.

`CATEGORY` deletion sets `PRODUCT.CATEGORY_ID` to `NULL`, so products remain available and can be recategorized later.

The self-referencing `CATEGORY.PARENT_ID` supports nested categories such as Electronics to Phones to Smartphones.

---

## Operations

The operations below assume the database contains these tables:

- `CATEGORY`
- `PRODUCT`
- `CUSTOMER`
- `ORDERS`
- `ORDER_DETAILS`

PostgreSQL examples use PostgreSQL syntax. MySQL alternatives are provided for the reporting queries where the syntax differs.

### Daily Revenue Report

Returns total orders, total items sold, and total revenue for a specific date. Soft-deleted orders are excluded.

| Column | Description |
|---|---|
| `ORDER_DATE` | The queried date. |
| `TOTAL_ORDERS` | Count of distinct orders on that date. |
| `TOTAL_ITEMS_SOLD` | Count of order detail rows. |
| `TOTAL_REVENUE` | Sum of `QUANTITY * UNIT_PRICE`. |

#### PostgreSQL

```sql
SELECT o.order_date,
       COUNT(DISTINCT o.order_id)       AS TOTAL_ORDERS,
       COUNT(od.order_detail_id)        AS TOTAL_ITEMS_SOLD,
       SUM(od.quantity * od.unit_price) AS TOTAL_REVENUE
FROM   ORDERS o
JOIN   ORDER_DETAILS od ON o.order_id = od.order_id
WHERE  o.ORDER_DATE = :'report_date'
  AND  o.is_deleted = FALSE
GROUP  BY o.order_date;
```

#### MySQL

```sql
SET @report_date = '2026-01-03';

SELECT  o.ORDER_DATE,
        COUNT(DISTINCT o.ORDER_ID)       AS TOTAL_ORDERS,
        COUNT(od.ORDER_DETAIL_ID)        AS TOTAL_ITEMS_SOLD,
        SUM(od.QUANTITY * od.UNIT_PRICE) AS TOTAL_REVENUE
FROM    ORDERS o
JOIN    ORDER_DETAILS od ON o.ORDER_ID = od.ORDER_ID
WHERE   o.ORDER_DATE = @report_date
  AND   o.IS_DELETED = FALSE
GROUP   BY o.ORDER_DATE;
```

### Monthly Top-Selling Products

Returns the top 10 products by units sold in a selected calendar month. Soft-deleted orders are excluded.

#### PostgreSQL

```sql
SELECT p.product_id,
       p.name,
       SUM(od.quantity) AS total_quantity_sold
FROM   order_details od
JOIN   product p ON od.product_id = p.product_id
JOIN   orders  o ON od.order_id   = o.order_id
WHERE  o.is_deleted = FALSE
  AND  DATE_TRUNC('month', o.order_date) = DATE_TRUNC('month', DATE '2026-03-01')
GROUP  BY p.product_id, p.name
ORDER  BY total_quantity_sold DESC
LIMIT  10;
```

#### MySQL

```sql
SELECT  p.product_id,
        p.name,
        SUM(od.quantity) AS total_quantity_sold
FROM    order_details od
JOIN    product p ON od.product_id = p.product_id
JOIN    orders  o ON od.order_id   = o.order_id
WHERE   o.is_deleted = FALSE
  AND   YEAR(o.order_date)  = 2026
  AND   MONTH(o.order_date) = 3
GROUP   BY p.product_id, p.name
ORDER   BY total_quantity_sold DESC
LIMIT   10;
```

### High-Value Customers Last 30 Days

Returns customers who have spent more than 500 in the last 30 days, sorted by highest spend. Soft-deleted orders are excluded.

#### PostgreSQL

```sql
SELECT c.customer_id,
       c.first_name,
       c.last_name,
       SUM(o.total_amount) AS total_spent
FROM   customer c
JOIN   orders o ON c.customer_id = o.customer_id
WHERE  o.is_deleted = FALSE
  AND  o.order_date >= CURRENT_DATE - INTERVAL '1 month'
GROUP  BY c.customer_id, c.first_name, c.last_name
HAVING SUM(o.total_amount) > 500
ORDER  BY total_spent DESC;
```

#### MySQL

```sql
SELECT  c.customer_id,
        c.first_name,
        c.last_name,
        SUM(o.total_amount) AS total_spent
FROM    customer c
JOIN    orders o ON c.customer_id = o.customer_id
WHERE   o.is_deleted = FALSE
  AND   o.order_date >= CURRENT_DATE - INTERVAL 1 MONTH
GROUP   BY c.customer_id, c.first_name, c.last_name
HAVING  total_spent > 500
ORDER   BY total_spent DESC;
```

PostgreSQL requires repeating the aggregate in `HAVING`; MySQL allows referencing the `total_spent` alias directly.

### Task 1: Search Products by Keyword

#### Objective

Search for products that contain the word `camera` in either the product name or product description. The search is case-insensitive.

#### SQL Query

```sql
SELECT *
FROM PRODUCT
WHERE NAME ILIKE '%camera%'
   OR DESCRIPTION ILIKE '%camera%';
```

#### Explanation

`ILIKE` is a PostgreSQL operator for case-insensitive pattern matching. It matches values such as `camera`, `Camera`, `CAMERA`, and `CaMeRa`.

#### Result

<img width="1572" height="608" alt="Search products result" src="https://github.com/user-attachments/assets/68284ca9-0bb3-486a-9af4-5946b9ae475d" />

### Task 2: Suggest Popular Products from the Same Category

#### Objective

Suggest popular products from the same category as a purchased product. The purchased product itself is excluded from the recommendations.

#### SQL Query

```sql
SELECT
    p.PRODUCT_ID,
    p.NAME,
    p.DESCRIPTION,
    p.PRICE,
    p.CATEGORY_ID,
    COALESCE(SUM(od.QUANTITY), 0) AS TOTAL_SOLD
FROM PRODUCT p
LEFT JOIN ORDER_DETAILS od
    ON p.PRODUCT_ID = od.PRODUCT_ID
WHERE p.CATEGORY_ID = (
    SELECT CATEGORY_ID
    FROM PRODUCT
    WHERE PRODUCT_ID = :purchased_product_id
)
AND p.PRODUCT_ID <> :purchased_product_id
AND p.STOCK_QTY > 0
GROUP BY
    p.PRODUCT_ID,
    p.NAME,
    p.DESCRIPTION,
    p.PRICE,
    p.CATEGORY_ID
ORDER BY TOTAL_SOLD DESC
LIMIT 10;
```

#### Result

With purchased product ID = 105:

<img width="1570" height="395" alt="Popular products result" src="https://github.com/user-attachments/assets/9caf8837-b756-431c-80b1-985ecd762afb" />

The query works as follows:

1. Finds the category of the purchased product.
2. Searches for other products in the same category.
3. Excludes the purchased product itself.
4. Excludes products that are out of stock.
5. Counts total sold quantity using `ORDER_DETAILS`.
6. Orders products by popularity.

### Task 3: Create Sale History Table

#### Objective

Create a sale history table that stores historical sales records.

Each sale history record captures:

- order ID
- order date
- customer ID
- product ID
- quantity
- unit price
- total amount
- creation timestamp

#### SQL Script

```sql
CREATE TABLE SALE_HISTORY
(
    SALE_HISTORY_ID INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    ORDER_ID        INTEGER NOT NULL,
    ORDER_DATE      DATE NOT NULL,
    CUSTOMER_ID     INTEGER NOT NULL,
    PRODUCT_ID      INTEGER NOT NULL,
    QUANTITY        INTEGER NOT NULL,
    UNIT_PRICE      NUMERIC(6, 2) NOT NULL,
    TOTAL_AMOUNT    NUMERIC(10, 2) NOT NULL,
    CREATED_AT      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (ORDER_ID) REFERENCES ORDERS(ORDER_ID),
    FOREIGN KEY (CUSTOMER_ID) REFERENCES CUSTOMER(CUSTOMER_ID),
    FOREIGN KEY (PRODUCT_ID) REFERENCES PRODUCT(PRODUCT_ID)
);
```

#### Explanation

`SALE_HISTORY` keeps a permanent record of sold products for reporting, analytics, auditing, customer purchase tracking, and product sales performance tracking.

### Task 4: Create Indexes for Sale History

#### Objective

Improve query performance on the `SALE_HISTORY` table.

#### SQL Script

```sql
CREATE INDEX idx_sale_history_customer
ON SALE_HISTORY (CUSTOMER_ID);

CREATE INDEX idx_sale_history_product
ON SALE_HISTORY (PRODUCT_ID);

CREATE INDEX idx_sale_history_order_date
ON SALE_HISTORY (ORDER_DATE);
```

#### Explanation

These indexes speed up common queries such as sales by customer, sales by product, and sales by date.

```sql
SELECT *
FROM SALE_HISTORY
WHERE CUSTOMER_ID = 100;
```

### Task 5: Create Sale History Trigger Function

#### Objective

Automatically insert a sale history record whenever a new row is inserted into `ORDER_DETAILS`.

Although the requirement mentions triggering on `ORDERS`, the correct table is `ORDER_DETAILS` because product and quantity data are stored there.

#### SQL Script

```sql
CREATE OR REPLACE FUNCTION create_sale_history()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO SALE_HISTORY
    (
        ORDER_ID,
        ORDER_DATE,
        CUSTOMER_ID,
        PRODUCT_ID,
        QUANTITY,
        UNIT_PRICE,
        TOTAL_AMOUNT
    )
    SELECT
        o.ORDER_ID,
        o.ORDER_DATE,
        o.CUSTOMER_ID,
        NEW.PRODUCT_ID,
        NEW.QUANTITY,
        NEW.UNIT_PRICE,
        NEW.QUANTITY * NEW.UNIT_PRICE
    FROM ORDERS o
    WHERE o.ORDER_ID = NEW.ORDER_ID;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

#### Explanation

`NEW` refers to the newly inserted row in `ORDER_DETAILS`.

For every inserted order detail, the function:

1. Gets the related order from `ORDERS`.
2. Reads the order date and customer ID.
3. Reads the product, quantity, and unit price from the new `ORDER_DETAILS` row.
4. Calculates the total amount using `NEW.QUANTITY * NEW.UNIT_PRICE`.
5. Inserts the result into `SALE_HISTORY`.

### Task 6: Create Trigger

#### Objective

Attach the trigger function to the `ORDER_DETAILS` table.

#### SQL Script

```sql
CREATE TRIGGER trg_create_sale_history
AFTER INSERT ON ORDER_DETAILS
FOR EACH ROW
EXECUTE FUNCTION create_sale_history();
```

#### Explanation

This trigger runs automatically after inserting a new row into `ORDER_DETAILS`. Because it is defined with `FOR EACH ROW`, it runs once for every inserted order detail record.

### Task 7: Test the Trigger

#### Objective

Insert a new order and order detail to verify that the trigger automatically inserts into `SALE_HISTORY`.

#### SQL Script

```sql
INSERT INTO ORDERS
(
    CUSTOMER_ID,
    ORDER_DATE,
    TOTAL_AMOUNT
)
VALUES
(
    100,
    CURRENT_DATE,
    2500.00
);

INSERT INTO ORDER_DETAILS
(
    ORDER_ID,
    PRODUCT_ID,
    QUANTITY,
    UNIT_PRICE
)
VALUES
(
    (SELECT MAX(ORDER_ID) FROM ORDERS),
    105,
    2,
    500.00
);

SELECT *
FROM SALE_HISTORY
ORDER BY SALE_HISTORY_ID DESC;
```

#### Expected Result

After inserting into `ORDER_DETAILS`, a new row should automatically appear in `SALE_HISTORY`.

| SALE_HISTORY_ID | ORDER_ID | ORDER_DATE | CUSTOMER_ID | PRODUCT_ID | QUANTITY | UNIT_PRICE | TOTAL_AMOUNT |
|---|---|---|---|---|---|---|---|
| 1 | 100 | 2026-05-09 | 100 | 105 | 2 | 500.00 | 1000.00 |

#### Important Note

The trigger is placed on `ORDER_DETAILS`, not `ORDERS`, because `ORDERS` does not contain product or quantity information.

#### Full Execution Order

1. Create `SALE_HISTORY` table.
2. Create indexes.
3. Create trigger function.
4. Create trigger.
5. Insert test order.
6. Insert test order details.
7. Query `SALE_HISTORY`.

#### Summary

This implementation provides:

- case-insensitive product search
- popular product recommendations
- sale history table
- performance indexes
- automatic sale history creation using triggers
- test insert statements

### Task 8: Lock Product Quantity and Row

#### Objective

Use a transaction to lock product ID 211 so other transactions cannot update it until the current transaction is committed or rolled back.

SQL databases such as PostgreSQL and MySQL lock rows, not a single field independently. Selecting only `STOCK_QTY` targets the quantity value in the query result, but the lock is still applied to the product row.

#### Lock the Quantity Field for Product ID 211

```sql
BEGIN;

SELECT STOCK_QTY
FROM PRODUCT
WHERE PRODUCT_ID = 211
FOR UPDATE;

-- Any needed quantity logic can run here while the row is locked.

COMMIT;
```

#### Lock the Full Product Row for Product ID 211

```sql
BEGIN;

SELECT *
FROM PRODUCT
WHERE PRODUCT_ID = 211
FOR UPDATE;

-- Any update to this product row by another transaction waits until COMMIT or ROLLBACK.

COMMIT;
```

#### Rollback Option

Use `ROLLBACK` instead of `COMMIT` if the transaction should be canceled:

```sql
ROLLBACK;
```
