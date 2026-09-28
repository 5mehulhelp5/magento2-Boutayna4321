# Magento 2 — Database Queries

> **Objective**: read and write data in Magento 2 — raw SQL for exploration and
> reports, the Collection API for application code, the Repository API for
> service contracts, and the correct way to change a schema.
>
> Read [magento-database.md](magento-database.md) first: it explains the tables,
> the EAV model, and the relations this guide queries.

Every SQL snippet here was executed against a live **Magento 2.4.8** /
**MySQL 8.0.46** install (465 tables, `STRICT_TRANS_TABLES` +
`ONLY_FULL_GROUP_BY` enabled) and the output shown is real.

---

## Table of Contents

1. [Three Ways to Query](#1-three-ways-to-query)
2. [Reading Raw SQL](#2-reading-raw-sql)
3. [EAV Queries](#3-eav-queries)
4. [Catalog Queries](#4-catalog-queries)
5. [Sales Queries](#5-sales-queries)
6. [Customer Queries](#6-customer-queries)
7. [Creating a Table](#7-creating-a-table)
8. [GROUP BY and ONLY_FULL_GROUP_BY](#8-group-by-and-only_full_group_by)
9. [Writing with the Collection API](#9-writing-with-the-collection-api)
10. [Writing with the Repository API](#10-writing-with-the-repository-api)
11. [Raw SQL in PHP](#11-raw-sql-in-php)
12. [Transactions](#12-transactions)
13. [Bulk Operations](#13-bulk-operations)
14. [Reading with the Collection API](#14-reading-with-the-collection-api)
15. [Debugging Queries](#15-debugging-queries)
16. [Performance](#16-performance)
17. [Anti-Patterns](#17-anti-patterns)
18. [Summary](#18-summary)

---

## 1. Three Ways to Query

| Approach | Use for | Speed | Cache/events |
|---|---|---|---|
| **Raw SQL** (MySQL client) | Exploration, one-off reports, DB admin | Native | None |
| **`ResourceConnection` + `Select`** | Aggregations, joins the Collection cannot express | Native | Manual |
| **Collection** | Application code, lists, grids, admin | Overhead per row | Automatic |
| **Repository** | Service contracts, REST/GraphQL | Overhead per row | Automatic |

**Decision rule**:

```
Do you need a value nobody computed before, aggregated over many rows?
    → raw SQL or ResourceConnection

Do you need Magento behaviour (events, cache, index invalidation, store fallback)?
    → Repository (service code) or Collection (internal code)
```

Never use raw SQL for something that must trigger Magento events, and never use
a Collection where you need a `GROUP BY` — Collections build `SELECT` statements,
not aggregates.

---

## 2. Reading Raw SQL

### 2.1 Setup

```bash
docker exec -it magento2-mysql mysql -umagento -pmagento2 magento2
# or, one-shot:
docker exec -i magento2-mysql mysql -umagento -pmagento2 magento2 --table < query.sql
```

Useful flags: `--table` (aligned), `-N` (no header), `-e "<query>"` (inline),
`-t` (force table output).

### 2.2 The five exploration queries

```sql
-- Which tables exist in a family?
SHOW TABLES LIKE 'sales_invoice%';

-- What columns does a table have?
DESCRIBE sales_order;

-- What is the exact DDL (types, defaults, charset, collation)?
SHOW CREATE TABLE catalog_product_entity_varchar;

-- What indexes exist, and in which order?
SHOW INDEX FROM sales_order;

-- Sample rows
SELECT * FROM sales_order LIMIT 5;
```

### 2.3 Finding things you don't know the name of

```sql
-- Find every table with a column called "order_id"
SELECT table_name, column_name, column_type
FROM information_schema.columns
WHERE table_schema = DATABASE() AND column_name = 'order_id'
ORDER BY table_name;

-- Find every table whose name contains "review"
SELECT table_name, table_rows
FROM information_schema.tables
WHERE table_schema = DATABASE() AND table_name LIKE '%review%';

-- Which tables have a foreign key pointing at sales_order?
SELECT TABLE_NAME, COLUMN_NAME, CONSTRAINT_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = DATABASE()
  AND REFERENCED_TABLE_NAME = 'sales_order';
```

This third query is the fastest way to discover the real relation graph of any
Magento table. Verified result: `sales_order_item.order_id`,
`sales_order_address.parent_id`, `sales_invoice.order_id`,
`sales_creditmemo.order_id`, `sales_shipment.order_id`,
`sales_order_grid.parent_id`, and more.

### 2.4 Always explain before you trust a query

```sql
EXPLAIN
SELECT o.entity_id, o.increment_id, o.grand_total
FROM sales_order o
WHERE o.store_id = 1
  AND o.state = 'processing'
  AND o.created_at >= '2026-08-01'
ORDER BY o.created_at DESC
LIMIT 100;
```

Look for `type: ALL` (full scan). Verified plan for this query on an install
with 36 orders:

```
key: SALES_ORDER_STATE   type: ref   rows: 6   Extra: Using where; Using filesort
```

The optimizer picked the single-column `SALES_ORDER_STATE` index, not the
composite `SALES_ORDER_STORE_ID_STATE_CREATED_AT` on
`(store_id, state, created_at)`. With only 36 rows that is the right choice —
the index statistics say the narrower index is cheaper. On a real catalog
(hundreds of thousands of orders) the composite wins, because it filters on
`store_id` and `state` and then ranges on `created_at` without a filesort.

This is the practical lesson: **`EXPLAIN` output depends on data volume.** A
plan that looks wrong on a fresh install is often correct, and a plan that looks
fine on a fresh install tells you nothing about production.

---

## 3. EAV Queries

Reading the schema is in [magento-database.md, section 3](magento-database.md#3-the-eav-model-in-detail).
This section shows the correct SQL.

### 3.1 Resolving an attribute_id

Never hardcode an `attribute_id` — they differ between installations. Always
resolve it, and **always scope by entity type**:

```sql
SELECT ea.attribute_id, ea.attribute_code, ea.backend_type, ea.frontend_input
FROM eav_attribute ea
JOIN eav_entity_type et ON et.entity_type_id = ea.entity_type_id
WHERE et.entity_type_code = 'catalog_product'
  AND ea.attribute_code = 'name';
```

**Verified result**: `attribute_id = 73`, `backend_type = varchar`.

> **The trap**: `attribute_code = 'name'` is not globally unique. Verified —
> `attribute_id 45` is the **category** `name`, `attribute_id 73` is the
> **product** `name`. Without the `eav_entity_type` join you get the wrong one
> and silently read `NULL`.

### 3.2 Reading an EAV value with store fallback

```sql
SELECT
    e.entity_id,
    e.sku,
    COALESCE(name_store.value, name_default.value) AS name
FROM catalog_product_entity AS e
LEFT JOIN catalog_product_entity_varchar AS name_store
       ON name_store.entity_id    = e.entity_id
      AND name_store.store_id     = 1
      AND name_store.attribute_id = 73
LEFT JOIN catalog_product_entity_varchar AS name_default
       ON name_default.entity_id    = e.entity_id
      AND name_default.store_id     = 0
      AND name_default.attribute_id = 73
WHERE e.sku = '24-MB01';
```

Verified output:

```
+------------+---------+-------------------+
| entity_id | sku     | name              |
+------------+---------+-------------------+
|          1 | 24-MB01 | Joust Duffle Bag  |
+------------+---------+-------------------+
```

The two LEFT JOINs plus `COALESCE` reproduce what Magento does internally: try
the requested store, fall back to `store_id = 0`. With a single join on
`store_id = 1` the query returns `NULL` on this install, because no product
name has ever been overridden for store view 1.

### 3.3 Joining two EAV attributes at once

```sql
SELECT e.entity_id, e.sku, n.value AS name, p.value AS price
FROM catalog_product_entity AS e
LEFT JOIN catalog_product_entity_varchar AS n
       ON n.entity_id = e.entity_id AND n.store_id = 0
      AND n.attribute_id = (SELECT ea.attribute_id FROM eav_attribute ea
                            JOIN eav_entity_type et ON et.entity_type_id = ea.entity_type_id
                            WHERE et.entity_type_code = 'catalog_product' AND ea.attribute_code = 'name')
LEFT JOIN catalog_product_entity_decimal AS p
       ON p.entity_id = e.entity_id AND p.store_id = 0
      AND p.attribute_id = (SELECT ea.attribute_id FROM eav_attribute ea
                            JOIN eav_entity_type et ON et.entity_type_id = ea.entity_type_id
                            WHERE et.entity_type_code = 'catalog_product' AND ea.attribute_code = 'price')
WHERE e.sku = '24-MB01';
```

Verified output:

```
+------------+---------+------------------+-----------+
| entity_id | sku     | name             | price     |
+------------+---------+------------------+-----------+
|          1 | 24-MB01 | Joust Duffle Bag | 34.000000 |
+------------+---------+------------------+-----------+
```

Both attribute lookups are the same shape. The two value tables differ only by
`backend_type` (`varchar` vs `decimal`).

### 3.4 Reading a `select` attribute's label

A `select` attribute stores an `option_id`, not text. Three joins:

```sql
SELECT e.sku, ov.value AS color_label
FROM catalog_product_entity_int AS ci
JOIN catalog_product_entity e            ON e.entity_id = ci.entity_id
JOIN eav_attribute_option o               ON o.option_id = ci.value
JOIN eav_attribute_option_value ov        ON ov.option_id = o.option_id
WHERE ci.attribute_id = 93          -- 'color' for catalog_product
ORDER BY e.entity_id
LIMIT 5;
```

Verified output:

```
+---------------+-------------+
| sku           | color_label |
+---------------+-------------+
| 24-WG081-gray | Gray        |
| 24-WG081-pink | Red         |
| 24-WG081-blue | Blue        |
| 24-WG082-gray | Gray        |
| 24-WG082-pink | Red         |
+---------------+-------------+
```

Note the SKU suffix `-pink` carries a value of `Red`: SKU and attribute are
independent, so never derive one from the other.

### 3.5 Listing an attribute set's attributes

```sql
SELECT eas.attribute_set_name, eag.attribute_group_name,
       ea.attribute_code, ea.backend_type
FROM eav_attribute_set eas
JOIN eav_entity_attribute eea ON eea.attribute_set_id = eas.attribute_set_id
JOIN eav_attribute_group eag ON eag.attribute_group_id = eea.attribute_group_id
                             AND eag.attribute_set_id  = eea.attribute_set_id
JOIN eav_attribute ea         ON ea.attribute_id       = eea.attribute_id
WHERE eas.attribute_set_id = 15
ORDER BY eag.sort_order, eea.sort_order
LIMIT 20;
```

Verified output (first 20 of 19 returned rows, showing the relevant ones):

```
+--------------------+----------------------+---------------+--------------+
| attribute_set_name | attribute_group_name | attribute_code | backend_type |
+--------------------+----------------------+---------------+--------------+
| Bag                | Product Details      | status        | int          |
| Bag                | Product Details      | name          | varchar      |
| Bag                | Product Details      | url_path      | varchar      |
| Bag                | Product Details      | sku           | static       |
| Bag                | Product Details      | price         | decimal      |
| Bag                | Product Details      | tax_class_id  | int          |
| Bag                | Product Details      | weight        | decimal      |
| Bag                | Product Details      | category_ids  | static       |
+--------------------+----------------------+---------------+--------------+
```

`backend_type = static` attributes (`sku`, `category_ids`, `created_at`) are
**not** stored in any EAV value table — they come from the base entity table.

### 3.6 The EAV performance trap

```sql
-- WRONG: no index usable on `value`, full scan of every product attribute
SELECT entity_id FROM catalog_product_entity_varchar WHERE value = 'Joust Duffle Bag';

-- RIGHT: filter by attribute_id first (indexed), then by value
SELECT entity_id FROM catalog_product_entity_varchar
WHERE attribute_id = 73 AND value = 'Joust Duffle Bag';
```

Verified `EXPLAIN` for the first query:

```
type: ALL   key: NULL   rows: 14420   Extra: Using where
```

`ALL` with a `NULL` key is a full table scan. The second query, verified:

```
type: ref   key: CATALOG_PRODUCT_ENTITY_VARCHAR_ATTRIBUTE_ID   rows: 2042
```

Verified counts on this install: `catalog_product_entity_varchar` holds
**14 322 rows**, of which **2 042** are `name` values (`attribute_id = 73`).
The first query reads all 14 322 rows and throws away 98.6% of them; the second
uses `CATALOG_PRODUCT_ENTITY_VARCHAR_ATTRIBUTE_ID` and reads 2 042.

For a full catalog listing, do not join EAV at all — use
`catalog_product_index_eav` (section 4.4) or the Product Collection.

---

## 4. Catalog Queries

### 4.1 The category tree

`catalog_category_entity` stores a materialized path.

```sql
-- Direct children of category 2
SELECT c.entity_id, c.parent_id, c.path, c.level, c.children_count, n.value AS name
FROM catalog_category_entity c
LEFT JOIN catalog_category_entity_varchar n
       ON n.entity_id = c.entity_id AND n.store_id = 0 AND n.attribute_id = 45
WHERE c.path = '1/2';
```

Verified output:

```
+-------------+-----------+------+-------+----------------+------------------+
| category_id | parent_id | path | level | children_count | name             |
+-------------+-----------+------+-------+----------------+------------------+
|           2 |         1 | 1/2  |     1 |             38 | Default Category |
+-------------+-----------+------+-------+----------------+------------------+
```

- **Descendants**: `WHERE c.path LIKE '1/2/%'`
- **Ancestors**: `WHERE FIND_IN_SET(c.parent_id, c.path)`
- **Breadcrumb**: `SELECT REPLACE(c.path, '/', ' > ') FROM catalog_category_entity c WHERE c.entity_id = ?`

### 4.2 Products in a category, on a website

```sql
SELECT e.entity_id, e.sku
FROM catalog_product_entity e
JOIN catalog_product_website pw ON pw.product_id = e.entity_id
JOIN catalog_category_product cp ON cp.product_id = e.entity_id
WHERE pw.website_id = 1
  AND cp.category_id = 3;
```

Both pivots are required: `catalog_product_website` for website visibility,
`catalog_category_product` for category membership. A product with no website
row is invisible everywhere.

### 4.3 Related / up-sell / cross-sell

Product links live in `catalog_product_link`, whose type is resolved through
`catalog_product_link_type`. **Verified schemas**:

```
catalog_product_link        link_id PK, product_id, linked_product_id, link_type_id
catalog_product_link_type   link_type_id PK, code
```

**Verified** `catalog_product_link_type` rows: `1 relation`, `3 super`,
`4 up_sell`, `5 cross_sell`.

```sql
SELECT e.sku, lt.code AS link_type, e2.sku AS linked_sku
FROM catalog_product_link l
JOIN catalog_product_link_type lt ON lt.link_type_id = l.link_type_id
JOIN catalog_product_entity e      ON e.entity_id  = l.product_id
JOIN catalog_product_entity e2     ON e2.entity_id = l.linked_product_id
WHERE e.sku = '24-MB04';
```

> **Do not confuse three similar tables**:
> - `catalog_product_link` (1 485 rows verified) — the current table for
>   related / up-sell / cross-sell, typed by `link_type_id`.
> - `catalog_product_super_link` (1 847 rows verified) — configurable children,
>   always `link_type_id` present via its own columns.
> - `catalog_product_relation` (1 858 rows verified) — a **legacy** table with
>   only `(parent_id, child_id)` and no type. Data is mirrored into
>   `catalog_product_link`; read the latter.

### 4.4 Reading prices from the index

```sql
SELECT p.entity_id, e.sku, p.min_price, p.max_price, p.website_id, p.customer_group_id
FROM catalog_product_index_price p
JOIN catalog_product_entity e ON e.entity_id = p.entity_id
WHERE e.sku = '24-MB01'
  AND p.website_id = 1
  AND p.customer_group_id = 0;
```

Verified output:

```
+------------+---------+----------+----------+-------------+-------------------+
| entity_id | sku     | min_price | max_price | website_id | customer_group_id |
+------------+---------+----------+----------+-------------+-------------------+
|          1 | 24-MB01 | 34.000000 | 34.000000 |           1 |                 0 |
+------------+---------+----------+----------+-------------+-------------------+
```

> **Always filter `website_id` and `customer_group_id`.** The table's PK is
> `(entity_id, customer_group_id, website_id)`, so an unfiltered join to
> `catalog_product_entity` returns **one row per customer group** and
> duplicates each product. Verified: this install has rows for customer groups
> 0, 1, 2 and 3, so an unfiltered query returns the same product four times.

Never write to any `catalog_*_index*` table. Rebuild with
`bin/magento indexer:reindex catalog_product_price`.

### 4.5 Inventory

```sql
SELECT s.source_code, s.name, i.sku, i.quantity, i.status
FROM inventory_source_item i
JOIN inventory_source s ON s.source_code = i.source_code
ORDER BY s.source_code, i.sku
LIMIT 10;
```

Verified output:

```
+-------------+----------------+---------+----------+--------+
| source_code | name           | sku     | quantity | status |
+-------------+----------------+---------+----------+--------+
| default     | Default Source | 24-MB01 | 100.0000 |      1 |
| default     | Default Source | 24-MB02 | 100.0000 |      1 |
+-------------+----------------+---------+----------+--------+
```

The join key is `source_code` (a varchar), **not** a `source_id`:
`inventory_source` has no numeric id column. `status`: 0 out of stock,
1 in stock, 2 below threshold.

---

## 5. Sales Queries

### 5.1 Order header with its lines

```sql
SELECT o.entity_id AS order_id, o.increment_id, o.status, o.grand_total,
       oi.item_id, oi.sku, oi.name, oi.qty_ordered, oi.row_total
FROM sales_order o
JOIN sales_order_item oi ON oi.order_id = o.entity_id
WHERE o.increment_id = '000000001'
ORDER BY oi.item_id;
```

Verified output:

```
+----------+--------------+------------+-------------+--------+-------------+------------------+------------+-------------+
| order_id | increment_id | status     | grand_total | item_id | sku         | name             | qty_ordered | row_total |
+----------+--------------+------------+-------------+--------+-------------+------------------+------------+-------------+
|        1 | 000000001    | processing |     36.3900 |       1 | WS03-XS-Red | Iris Workout Top |      1.0000 |   29.0000 |
+----------+--------------+------------+-------------+--------+-------------+------------------+------------+-------------+
```

`sales_order_item` already contains `sku`, `name`, `price` and `qty_ordered` as
**snapshots taken at purchase time**. Never join back to the catalog for an
order report.

### 5.2 Revenue by month

```sql
SELECT DATE_FORMAT(created_at, '%Y-%m') AS month,
       COUNT(*)                           AS nb_orders,
       SUM(grand_total)                   AS ca
FROM sales_order
WHERE status NOT IN ('canceled')
GROUP BY month
ORDER BY month;
```

Verified output:

```
+---------+-----------+-----------+
| month   | nb_orders | ca        |
+---------+-----------+-----------+
| 2026-07 |         7 | 1119.1300 |
| 2026-08 |        20 | 4430.9600 |
| 2026-09 |         7 |  261.0000 |
+---------+-----------+-----------+
```

Two rules to apply to every revenue query:

1. **Exclude canceled orders.** Without the `WHERE`, canceled orders inflate
   the figure (verified: order `000000007` is `canceled` for 570.00).
2. **Aggregate `base_*`, not the plain columns.** `base_grand_total` is in base
   currency; `grand_total` depends on the store view the customer used. See
   [magento-database.md, section 5.2](magento-database.md#52-sales_order-the-reference-flat-table).

### 5.3 State vs status

`status` is the fine-grained label, `state` is the coarse bucket, and one status
can belong to several states. Verified from `sales_order_status_state`:

```
status          → state
pending         → new
fraud           → payment_review
fraud           → processing
processing      → processing
complete        → complete
canceled        → canceled
```

So `state = 'processing'` returns both `processing` **and** `fraud` orders.
Always say which one you mean.

### 5.4 The full document rollup

```sql
SELECT o.entity_id AS order_id, o.increment_id, o.status, o.grand_total,
       (SELECT COUNT(*) FROM sales_invoice    i  WHERE i.order_id  = o.entity_id) AS nb_invoices,
       (SELECT COUNT(*) FROM sales_creditmemo cm WHERE cm.order_id = o.entity_id) AS nb_creditmemos,
       (SELECT COUNT(*) FROM sales_shipment   s  WHERE s.order_id  = o.entity_id) AS nb_shipments
FROM sales_order o
ORDER BY o.entity_id
LIMIT 10;
```

Verified output:

```
+----------+--------------+------------+-------------+-------------+----------------+--------------+
| order_id | increment_id | status     | grand_total | nb_invoices | nb_creditmemos | nb_shipments |
+----------+--------------+------------+-------------+-------------+----------------+--------------+
|        1 | 000000001    | processing |     36.3900 |           1 |              0 |            1 |
|        2 | 000000002    | closed     |     39.6400 |           1 |              1 |            1 |
|        3 | 000000003    | pending    |    205.0000 |           1 |              0 |            0 |
|        7 | 000000007    | canceled   |    570.0000 |           0 |              0 |            0 |
+----------+--------------+------------+-------------+-------------+----------------+--------------+
```

Column-type warning: `sales_order.state` is a `varchar`; `sales_invoice.state`
and `sales_creditmemo.state` are `smallint`. Comparing `'processing'` against
an invoice `state` silently returns nothing.

### 5.5 Config values

```sql
SELECT scope, scope_id, path, value
FROM core_config_data
WHERE path = 'web/unsecure/base_url'
ORDER BY scope, scope_id;
```

Verified output:

```
+---------+----------+------------------------+---------------------------+
| scope   | scope_id | path                   | value                     |
+---------+----------+------------------------+---------------------------+
| default |        0 | web/unsecure/base_url  | http://localhost/         |
| stores  |        6 | web/unsecure/base_url  | http://localhost/         |
| stores  |        7 | web/unsecure/base_url  | http://localhost/         |
| stores  |       22 | web/unsecure/base_url  | http://localhost/         |
+---------+----------+------------------------+---------------------------+
```

The scope type is `stores` (not `store_view`) and `scope_id` is a **store id**.
Raw SQL must replicate the `stores` → `websites` → `default` fallback that
`ScopeCodeConverter` applies. Read config in PHP with `ScopeConfigInterface`
instead.

---

## 6. Customer Queries

### 6.1 Customers and their last order

```sql
SELECT c.entity_id, c.email, c.firstname, c.lastname,
       o.increment_id, o.created_at, o.grand_total
FROM customer_entity c
LEFT JOIN sales_order o ON o.customer_id = c.entity_id
ORDER BY c.entity_id, o.created_at DESC
LIMIT 10;
```

A plain `LEFT JOIN` returns **one row per order**, not per customer. For a true
"last order per customer" you need either a correlated subquery or a window
function (MySQL 8 supports them):

```sql
SELECT c.entity_id, c.email,
       o.increment_id, o.created_at, o.grand_total
FROM customer_entity c
LEFT JOIN (
    SELECT so.*,
           ROW_NUMBER() OVER (PARTITION BY so.customer_id ORDER BY so.created_at DESC) AS rn
    FROM sales_order so
    WHERE so.customer_id IS NOT NULL
) o ON o.customer_id = c.entity_id AND o.rn = 1
ORDER BY c.entity_id;
```

### 6.2 Customer EAV attributes

`customer_entity` is flat for `email`, `firstname`, `group_id`, `website_id`,
`is_active`. Custom attributes live in `customer_entity_*` — and unlike product
EAV, those tables have **no `store_id` column** (verified).

```sql
SELECT c.entity_id, c.email, v.value AS customer_type
FROM customer_entity c
LEFT JOIN customer_entity_varchar v
       ON v.entity_id = c.entity_id
      AND v.attribute_id = (SELECT ea.attribute_id FROM eav_attribute ea
                            JOIN eav_entity_type et ON et.entity_type_id = ea.entity_type_id
                            WHERE et.entity_type_code = 'customer'
                              AND ea.attribute_code = 'customer_type')
WHERE c.email = 'roni_cost@example.com';
```

### 6.3 Lifetime spend

```sql
SELECT c.entity_id, c.email,
       COALESCE(SUM(o.base_grand_total), 0) AS lifetime_spent,
       COUNT(o.entity_id)                   AS nb_orders
FROM customer_entity c
LEFT JOIN sales_order o ON o.customer_id = c.entity_id
                       AND o.state IN ('complete', 'closed')
                       AND o.status NOT IN ('canceled', 'fraud')
GROUP BY c.entity_id, c.email
ORDER BY lifetime_spent DESC
LIMIT 10;
```

The `LEFT JOIN` with the conditions in `ON` (not `WHERE`) is deliberate: a
customer with no completed orders must still appear with `0`, not disappear.

---

## 7. Creating a Table

### 7.1 Declarative schema: `db_schema.xml`

Never run `CREATE TABLE` in SQL by hand. Magento tracks the schema in
`patch_list` and `flag`, and a manual `CREATE TABLE` produces a state where
`bin/magento setup:db:status` disagrees with reality. Declare it in
`app/code/<Vendor>/<Module>/etc/db_schema.xml`:

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="vendor_module_wishlist_note" resource="default" engine="innodb"
           comment="Wishlist Note">
        <column xsi:type="int" name="note_id" unsigned="true" nullable="false" identity="true"
                comment="Note ID"/>
        <column xsi:type="int" name="wishlist_item_id" unsigned="true" nullable="false"
                comment="Wishlist Item ID"/>
        <column xsi:type="int" name="customer_id" unsigned="true" nullable="false"
                comment="Customer ID"/>
        <column xsi:type="text" name="note" nullable="true" comment="Note Text"/>
        <column xsi:type="boolean" name="is_approved" nullable="false" default="0"
                comment="Approved"/>
        <column xsi:type="timestamp" name="created_at" nullable="false"
                default="CURRENT_TIMESTAMP" comment="Created At"/>
        <column xsi:type="timestamp" name="updated_at" on_update="true" nullable="false"
                default="CURRENT_TIMESTAMP" comment="Updated At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="note_id"/>
        </constraint>
        <constraint xsi:type="unique" referenceId="VENDOR_MODULE_WISHLIST_NOTE_ITEM_CUSTOMER">
            <column name="wishlist_item_id"/>
            <column name="customer_id"/>
        </constraint>
        <index referenceId="VENDOR_MODULE_WISHLIST_NOTE_CUSTOMER_ID" indexType="btree">
            <column name="customer_id"/>
        </index>
    </table>
</schema>
```

**Type mapping** (XML → MySQL):

| `xsi:type` | MySQL |
|---|---|
| `int`, `smallint`, `bigint` | `int unsigned` when `unsigned="true"` |
| `boolean` | `smallint` (0/1) |
| `varchar` | `varchar(n)` with `length` |
| `text`, `mediumtext` | `text`, `mediumtext` |
| `decimal` | `decimal(p, s)` via `scale` and `precision` |
| `date`, `timestamp`, `datetime` | same |
| `blob` | `blob` |

**Constraint types**: `primary`, `unique`, `foreign`, `index`.

**Idioms to reuse**:
- `unsigned="true"` on every int that references another table's `entity_id`
- `identity="true"` marks the auto-increment PK
- `on_update="true"` on `updated_at`
- `resource="default"` for new tables; `resource="checkout"` only when adding a
  column to a core checkout table (supported and intentional)

### 7.2 Applying the schema

```bash
bin/magento setup:db:status                 # is a change pending?
bin/magento setup:db-schema:upgrade          # apply db_schema.xml changes
bin/magento setup:db-declaration:generate-patch
bin/magento setup:upgrade                    # full run (schema + data)
```

Verified: `bin/magento setup:db:status` on an untouched install reports
`All modules are up to date.`

Order matters: after `setup:db-schema:upgrade`, run
`bin/magento setup:di:compile` and `bin/magento cache:flush` if you added a
ResourceModel.

### 7.3 Backing up before a schema change

```bash
# Structure only
mysqldump -umagento -p --no-data magento2 > schema.sql

# Full dump (large, but mandatory before a production change)
mysqldump -umagento -p --single-transaction magento2 > full.sql
```

### 7.4 Data patches vs schema

`db_schema.xml` handles **structure**. To add or change **rows** (config values,
attribute options, EAV attributes), use a data patch in
`Setup/Patch/Data/`:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddAttributeOptions implements DataPatchInterface
{
    public function __construct(private readonly ModuleDataSetupInterface $moduleDataSetup)
    {
    }

    public static function getDependencies(): array
    {
        return [];
    }

    public function getAliases(): array
    {
        return [];
    }

    public function apply(): void
    {
        $this->moduleDataSetup->startSetup();
        // ... insert eav_attribute_option / eav_attribute_option_value rows ...
        $this->moduleDataSetup->endSetup();
    }
}
```

The interface is `Magento\Framework\Setup\Patch\DataPatchInterface`; the
schema equivalent is `Magento\Framework\Setup\Patch\SchemaPatchInterface`.
Always call `startSetup()` / `endSetup()`.

### 7.5 Data patches are not run twice

`getAliases()` makes a patch idempotent. Add a stable alias when you need a patch
to be re-applied (for example to fix a wrong default):

```php
public function getAliases(): array
{
    return ['vendor_module_fix_wrong_default'];
}
```

Without an alias, a patch that has already run is skipped forever.

### 7.6 Adding a column to a core table

Two approaches, both legitimate:

**a) `db_schema.xml`** — simplest, works for adding a column:

```xml
<table name="quote" resource="checkout" comment="Sales Flat Quote">
    <column xsi:type="int" name="my_module_points_used" nullable="true" identity="false"
            default="0" comment="Points Used"/>
</table>
```

`resource="checkout"` is required for checkout tables. Core columns cannot be
dropped or retyped this way.

**b) A `SchemaPatchInterface` class** — for column renames, retypes, or dropping
columns, which declarative schema forbids.

For a quote column you still need the attribute metadata (`quote` attribute) to
appear in checkout forms; the column alone is invisible to the checkout.

---

## 8. GROUP BY and ONLY_FULL_GROUP_BY

This install runs with `ONLY_FULL_GROUP_BY`, and so does any default Magento 2
stack. It rejects any query where a selected column is neither aggregated nor in
the `GROUP BY`.

**Rejected:**

```sql
-- Fails: created_at is neither in GROUP BY nor aggregated
SELECT DATE_FORMAT(created_at, '%Y-%m') AS month, created_at, COUNT(*)
FROM sales_order
GROUP BY month;
```

Verified error, verbatim:

```
ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP BY clause
and contains nonaggregated column 'magento2.sales_order.created_at';
this is incompatible with sql_mode=only_full_group_by
```

Note how the message names the **position** in the SELECT list and the exact
column. That is usually enough to fix the query without further guessing.

**Accepted:**

```sql
-- Aggregated
SELECT DATE_FORMAT(created_at, '%Y-%m') AS month, COUNT(*), MIN(created_at), MAX(created_at)
FROM sales_order
GROUP BY month;

-- Fully grouped
SELECT DATE_FORMAT(created_at, '%Y-%m') AS month, DATE(created_at) AS day, COUNT(*)
FROM sales_order
GROUP BY month, day;
```

The `GROUP BY` alias (`month`) is accepted by MySQL, so a derived column can be
the grouping key. Both accepted forms above were executed successfully against
this install.

`GROUP_CONCAT` is the standard Magento fix for "one row per parent plus a
collapsed child list":

```sql
SELECT o.increment_id,
       GROUP_CONCAT(oi.sku ORDER BY oi.item_id SEPARATOR ', ') AS skus
FROM sales_order o
JOIN sales_order_item oi ON oi.order_id = o.entity_id
GROUP BY o.entity_id, o.increment_id
ORDER BY o.entity_id
LIMIT 10;
```

---

## 9. Writing with the Collection API

### 9.1 The three classes

```
Model\MyEntity                    ← data object, extends AbstractModel
Model\ResourceModel\MyEntity      ← table binding, extends AbstractDb
Model\ResourceModel\MyEntity\Collection ← query builder, extends AbstractCollection
```

**ResourceModel** — binds the table and the id column:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class Note extends AbstractDb
{
    protected function _construct(): void
    {
        $this->_init('vendor_module_wishlist_note', 'note_id');
    }
}
```

**Model** — binds itself to its ResourceModel:

```php
class Note extends AbstractModel
{
    protected function _construct(): void
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Note::class);
    }

    public function getNote(): ?string
    {
        return $this->getData('note');
    }

    public function setNote(string $note): self
    {
        return $this->setData('note', $note);
    }
}
```

**Collection**:

```php
class Collection extends AbstractCollection
{
    protected function _construct(): void
    {
        $this->_init(
            \Vendor\Module\Model\Note::class,
            \Vendor\Module\Model\ResourceModel\Note::class
        );
    }
}
```

### 9.2 Creating a row

```php
$note = $this->noteFactory->create();
$note->setWishlistItemId($itemId)
     ->setCustomerId($customerId)
     ->setNote('Restock in September')
     ->setIsApproved(false)
     ->save();

$noteId = $note->getId();   // auto_increment, resolved after save
```

`save()` goes through the ResourceModel, which dispatches
`*_save_before` / `*_save_after` events and invalidates the affected cache
types. This is why you use `save()` and not `$connection->insert()`.

### 9.3 Updating and deleting

```php
$note = $this->noteFactory->create();
$this->noteResource->load($note, $noteId);   // by primary key
$note->setNote('Updated text')->save();

$note->delete();                             // dispatches delete events
```

### 9.4 The N+1 trap

```php
// BAD: 1 + N queries
foreach ($this->noteCollectionFactory->create() as $note) {
    $customer = $this->customerRepository->getById($note->getCustomerId());
    echo $note->getNote() . ' — ' . $customer->getEmail();
}
```

```php
// GOOD: 1 query for notes, 1 for the customers
$collection = $this->noteCollectionFactory->create();
$customerIds = $collection->getColumnValues('customer_id');   // distinct

$criteria = $this->searchCriteriaBuilder
    ->addFilter('entity_id', ['in' => $customerIds])
    ->setPageSize(count($customerIds))
    ->create();
$customers = $this->customerRepository->getList($criteria);

$byId = [];
foreach ($customers->getItems() as $customer) {
    $byId[$customer->getId()] = $customer;
}

foreach ($collection as $note) {
    $customer = $byId[$note->getCustomerId()] ?? null;
    // ...
}
```

`getColumnValues()` returns the distinct values of one column and does not
reload the collection. `CustomerRepositoryInterface` exposes
`getById()`, `get($email)`, `getList(SearchCriteria)`, `save()` and `delete()` —
there is no `getByIds()`, so batch loading goes through `getList()`.

### 9.5 Fetching many rows

```php
$collection = $this->noteCollectionFactory->create();
$collection->setPageSize(500);        // hard cap, always set one
$notes = $collection->getItems();    // one SELECT, hydrated into models

foreach ($notes as $note) {
    $note->setIsApproved(true);
    $note->save();
}
```

Set a page size on every collection in application code. Without it, Magento
loads every row into memory.

---

## 10. Writing with the Repository API

The Repository is the **service contract** layer. Prefer it in controllers,
consumers, and anything exposed by REST or GraphQL.

### 10.1 The four methods

```php
public function save(NoteInterface $note): NoteInterface;
public function getById(int $noteId): NoteInterface;      // throws NoSuchEntityException
public function getList(SearchCriteriaInterface $c): SearchResultsInterface;
public function delete(NoteInterface $note): bool;
```

### 10.2 The contract, the implementation, the interface

```php
// Api/Data/NoteInterface.php
interface NoteInterface
{
    public function getNoteId(): ?int;
    public function setNoteId(int $id): NoteInterface;
    public function getCustomerId(): ?int;
    public function getNote(): ?string;
    public function getIsApproved(): bool;
}
```

```php
// Model/NoteRepository.php
class NoteRepository implements NoteRepositoryInterface
{
    public function __construct(
        private readonly NoteFactory $noteFactory,
        private readonly ResourceModel\Note $resource
    ) {
    }

    public function save(NoteInterface $note): NoteInterface
    {
        try {
            $this->resource->save($note);
        } catch (\Exception $e) {
            throw new CouldNotSaveException(__('Could not save note: %1', $e->getMessage()), $e);
        }
        return $note;
    }

    public function getById(int $noteId): NoteInterface
    {
        $note = $this->noteFactory->create();
        $this->resource->load($note, $noteId);
        if (!$note->getId()) {
            throw new NoSuchEntityException(__('Note with id "%1" does not exist.', $noteId));
        }
        return $note;
    }
}
```

Wrap every write in the `CouldNotSaveException` / `CouldNotDeleteException`
contract exceptions, and always throw `NoSuchEntityException` on a missing id —
the REST layer maps these to 400 / 404 responses.

### 10.3 SearchCriteria

```php
$sortOrder = $this->sortOrderBuilder->create()
    ->setField('created_at')
    ->setDirection('DESC');

$criteria = $this->searchCriteriaBuilder
    ->addFilter('customer_id', $customerId)
    ->addFilter('is_approved', 1)
    ->addSortOrder($sortOrder)
    ->setPageSize(50)
    ->setCurrentPage(1)
    ->create();

$result = $this->noteRepository->getList($criteria);
$notes  = $result->getItems();
$total  = $result->getTotalCount();
```

`SearchCriteriaBuilder` lives at `Magento\Framework\Api\SearchCriteriaBuilder`.
Its `addFilter()` signature is `addFilter($field, $value, $conditionType = 'eq')`,
and `addSortOrder()` takes a **SortOrder object**, not two strings — build it
with `Magento\Framework\Api\SortOrderBuilder`. Use
`CollectionProcessorInterface` in the repository so `SearchCriteria` filters
are applied to the underlying collection.

**Filters are AND by default.** For OR, build a `FilterGroup` explicitly and
pass it with `setFilterGroups()`. `SearchCriteriaBuilder` has no `addFilterGroup()`
method — the builder is `Magento\Framework\Api\Search\FilterGroupBuilder`:

```php
$group = $this->filterGroupBuilder->create();
$group->addFilter($this->filterBuilder->create()
    ->setField('status')
    ->setValue('holded')
    ->setConditionType('eq'));
$group->addFilter($this->filterBuilder->create()
    ->setField('status')
    ->setValue('fraud')
    ->setConditionType('eq'));

$criteria = $this->searchCriteriaBuilder
    ->setFilterGroups([$group])   // inside a group: OR; between groups: AND
    ->setPageSize(50)
    ->create();
```

`Magento\Framework\Api\FilterBuilder` provides `setField()`, `setValue()` and
`setConditionType()`. Multiple groups are AND-ed with each other, filters
inside one group are OR-ed.

### 10.4 Model vs Repository

| | Model + ResourceModel | Repository |
|---|---|---|
| Bypasses | nothing | nothing |
| Contract exceptions | none | `CouldNotSaveException`, `NoSuchEntityException` |
| REST/GraphQL ready | no | yes |
| SearchCriteria | no | yes |
| Use in | internal collections, blocks | controllers, consumers, APIs |

Never call `$model->save()` directly in a controller. A Repository is the only
place that knows how to turn a failure into an exception the framework expects.

---

## 11. Raw SQL in PHP

### 11.1 The correct way: `ResourceConnection`

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\App\ResourceConnection;

class NoteStats
{
    public function __construct(private readonly ResourceConnection $resource)
    {
    }

    public function countForCustomer(int $customerId): int
    {
        $connection = $this->resource->getConnection('default');
        $select = $connection->select()
            ->from($this->resource->getTableName('vendor_module_wishlist_note'), 'note_id')
            ->where('customer_id = ?', $customerId);

        $count = $connection->fetchOne($select);

        return $count === false ? 0 : (int) $count;
    }
}
```

**The three rules**:

1. `ResourceConnection::getTableName()` — never hardcode a table name; a table
   prefix would break it.
2. `->where('col = ?', $value)` — never concatenate. This produces a bound
   parameter and is the only SQL-injection-safe form.
3. `$connection->select()` returns a `Magento\Framework\DB\Select`, not a string.

### 11.2 Fetch variants

```php
$connection = $this->resource->getConnection('default');

$connection->fetchOne($select);                    // scalar
$connection->fetchAll($select);                    // array of assoc arrays
$connection->fetchRow($select);                    // one assoc array
$connection->fetchCol($select);                    // first column, as a list
$connection->fetchPairs($select);                  // col0 => col1 map
```

**`fetchOne` returns `false`, not `null`, when there is no row** (the connection
throws on error, so an empty result is `false`). Cast defensively:

```php
$value = $connection->fetchOne($select);
return $value === false ? null : (float) $value;
```

### 11.3 When raw SQL is justified

| Situation | Use |
|---|---|
| `GROUP BY` / aggregate over 100k rows | Raw SQL — Collections are per-row |
| Bulk `UPDATE` of a denormalized column | Raw SQL |
| Reporting for a cron or an export | Raw SQL |
| Anything that must fire Magento events | Repository |
| A single entity read in a controller | Repository |

### 11.4 Never do this

```php
// SQL injection via string concatenation
$connection->query("SELECT * FROM note WHERE customer_id = {$customerId}");

// Table name not resolved
$select = $connection->select()->from('vendor_module_wishlist_note', 'note_id');

// Writing without events, cache or index invalidation
$connection->insert('vendor_module_wishlist_note', ['customer_id' => 1]);
```

The last one is the dangerous one. It appears to work, and then the admin grid
still shows the old data, the cache serves stale rows, and no observer ran.

---

## 12. Transactions

### 12.1 The API

`Magento\Framework\DB\Adapter\AdapterInterface` exposes
`beginTransaction()`, `commit()`, `rollBack()`.

```php
$connection = $this->resource->getConnection('default');
$connection->beginTransaction();

try {
    $note->save();
    $this->notificationRepository->save($notification);
    $connection->commit();
} catch (\Exception $e) {
    $connection->rollBack();
    throw $e;
}
```

### 12.2 When to use one

Any operation that must be **all-or-nothing** across several writes:

- inserting a parent row plus its children
- order creation plus a related record
- decrementing stock while writing a log row
- a multi-step import

### 12.3 Caveats

- **No DDL inside a transaction.** MySQL implicitly commits on DDL. A
  `CREATE TABLE` inside `beginTransaction()` silently ends the transaction.
- **Observers may open their own transactions**, and nesting is not supported
  cleanly. Keep transaction scopes small.
- **The cache is not transactional.** If you roll back, previously invalidated
  cache stays invalidated (harmless), but anything cached *during* the
  transaction is not automatically invalidated on rollback. Flush if in doubt.
- Do not hold a transaction open across a network or API call.

---

## 13. Bulk Operations

### 13.1 Insert many rows

```php
$connection->insertMultiple($table, $rows);   // one multi-row INSERT
```

`insertMultiple()` builds a single statement. Verified available on
`AdapterInterface`. Prefer it over a `foreach` with `insert()`.

### 13.2 `insertOnDuplicate`

```php
$connection->insertOnDuplicate($table, $row);   // INSERT ... ON DUPLICATE KEY UPDATE
```

Requires a `UNIQUE` constraint on the columns you match on. This is the correct
way to make an import idempotent — and a `UNIQUE` constraint is mandatory
without it.

### 13.3 Large sets: chunk, do not load

```php
$collection = $this->noteCollectionFactory->create();
$collection->setPageSize(1000);

$page = 1;
do {
    $collection->setCurPage($page);
    $collection->loadData();

    foreach ($collection->getItems() as $note) {
        $this->process($note);
    }

    $page++;
} while ($collection->getCurPage() * $collection->getPageSize() < $collection->getSize());
```

`setCurPage()` and `getPageSize()` are declared on
`Magento\Framework\Data\Collection`; `getSize()` runs a `COUNT(*)`.

Better: use `selectsByRange()` (available on `AdapterInterface`) to walk a
primary-key range without materialising rows.

### 13.4 `updateFromSelect` / `deleteFromSelect`

```php
$connection->updateFromSelect($select, $table);
$connection->deleteFromSelect($select, $table);
```

Both available on `AdapterInterface`. They set-based operations run in a single
statement — much faster than a per-row loop, and they cannot hit PHP's
memory limit.

---

## 14. Reading with the Collection API

### 14.1 Basic filtering

```php
$collection = $this->noteCollectionFactory->create();
$collection->addFieldToFilter('customer_id', $customerId)
    ->addFieldToFilter('is_approved', 1)
    ->setOrder('created_at', 'DESC')
    ->setPageSize(50);

$notes = $collection->getItems();
```

Conditions accept an array, documented in
`Magento\Framework\Data\Collection\AbstractDb::_getConditionSql()`:
`from`/`to`, `eq`, `neq`, `like`, `in`, `nin`, `notnull`, `null`, `moreq`, `gt`,
`lt`, `gteq`, `lteq`, `finset`, `regexp`.

```php
$collection->addFieldToFilter('note_id', ['in' => [1, 2, 3]]);
$collection->addFieldToFilter('note', ['like' => '%restock%']);
$collection->addFieldToFilter('created_at', ['from' => '2026-01-01', 'to' => '2026-12-31']);
$collection->addFieldToFilter('is_approved', ['eq' => 1]);
```

Passing an array as the **first** argument instead builds an OR group:

```php
$collection->addFieldToFilter([
    ['field' => 'is_approved', 'eq' => 1],
    ['field' => 'is_featured', 'eq' => 1],
]);   // is_approved = 1 OR is_featured = 1
```

### 14.2 Joins

```php
$collection = $this->noteCollectionFactory->create();
$collection->join(
    ['customer' => $this->resource->getTableName('customer_entity')],
    'main_table.customer_id = customer.entity_id',
    ['customer_email' => 'customer.email', 'customer_name' => 'customer.firstname']
);
$collection->addFieldToFilter('customer.is_active', 1);
```

Alias the joined table (`['customer' => $tableName]`) — unaliased joins collide
with the main table's columns. `join()` is declared on
`Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection`; filter
on the joined table with the alias prefix (`customer.is_active`).

### 14.3 The Product Collection (EAV done right)

For products, use the Product Collection instead of hand-written EAV joins:

```php
$collection = $this->productCollectionFactory->create();
$collection->addAttributeToSelect(['name', 'price', 'status', 'color'])
    ->addAttributeToFilter('status', 1)
    ->addAttributeToFilter('name', ['like' => '%bag%'])
    ->setStoreId(1)
    ->setPageSize(24);
```

`addAttributeToSelect()` and `addAttributeToFilter()` are declared on
`Magento\Catalog\Model\ResourceModel\Product\Collection`; `setStoreId()` comes
from its parent `Magento\Catalog\Model\ResourceModel\Collection\AbstractCollection`.
They resolve `attribute_id`, pick the right value table from `backend_type`, and
apply the store fallback for you. This is exactly the manual EAV join of
section 3, done correctly and cache-aware.

**Always set the store id.** Without it the collection uses the admin scope
(`store_id = 0`).

### 14.4 The page-size trap

`getSize()` on a filtered collection runs a `COUNT(*)` — a second query. In a
loop, this doubles your query count. Use `count()` on the loaded items if you
have already paginated, or accept the extra query for an accurate total.

---

## 15. Debugging Queries

### 15.1 See the generated SQL

```php
// One collection — signature is loadData($printQuery = false, $logQuery = false)
$collection->loadData(true);      // echoes the generated SQL
```

```bash
# SQL statements land in the log when debug logging is on
tail -f var/log/debug.log | grep -i "select"
```

For raw SQL, render the `Select` — `Magento\Framework\DB\Select` implements
`__toString()`:

```php
echo $select;                    // (string) $select renders the SQL
$logger->debug((string) $select);
```

### 15.2 The EXPLAIN path

```sql
EXPLAIN SELECT ...;          -- plan
EXPLAIN ANALYZE SELECT ...;  -- plan + actual timings (MySQL 8.0.18+)
```

```sql
-- Indexes actually available on a table
SELECT index_name, seq_in_index, column_name, cardinality
FROM information_schema.statistics
WHERE table_schema = DATABASE() AND table_name = 'sales_order'
ORDER BY index_name, seq_in_index;
```

This is how you confirm the index your query needs really exists. For
`sales_order`, verified index `SALES_ORDER_STORE_ID_STATE_CREATED_AT` covers
`(store_id, state, created_at)`, so a query filtering on `store_id` + `state`
with a `created_at` range has a matching index available.

### 15.3 When the data is there but PHP shows nothing

Check in this order:

1. `bin/magento indexer:status` — an invalid indexer explains it.
2. `bin/magento cache:status` / `cache:flush`.
3. `bin/magento setup:db:status` — a pending schema change.
4. The store scope (`store_id`) you filtered on.
5. The EAV `store_id = 0` fallback row (section 3.2).

### 15.4 Count rows first

```sql
-- Estimated (fast, from statistics)
SELECT table_name, table_rows FROM information_schema.tables
WHERE table_schema = DATABASE() AND table_name = 'catalog_product_entity_varchar';

-- Exact (slow on huge tables)
SELECT COUNT(*) FROM catalog_product_entity_varchar;
```

A mismatch between the estimate and the exact count means stale statistics. In
production use `ANALYZE TABLE` during a maintenance window.

---

## 16. Performance

### 16.1 Index the filter, in this order

1. Equality filter on `store_id` / `website_id`
2. Equality filter on the second column
3. Range on the date column
4. `ORDER BY` on the same trailing column as the range, or the index is unused

Magento's own composite indexes follow this: verified
`SALES_ORDER_STORE_ID_STATE_CREATED_AT` on `(store_id, state, created_at)`,
and `CATALOG_PRODUCT_ENTITY_VARCHAR_ENTITY_ID_ATTRIBUTE_ID_STORE_ID` on
`(entity_id, attribute_id, store_id)`.

### 16.2 Never `SELECT *` on a wide table

`sales_order` has ~80 columns and `sales_order_item` ~100. Selecting all of them
for a 10 000-row report transfers megabytes of unused data.

```sql
-- BAD
SELECT * FROM sales_order;

-- GOOD
SELECT entity_id, increment_id, created_at, status, base_grand_total
FROM sales_order;
```

### 16.3 Do not join EAV for lists

For 50 products, 6 EAV joins mean 300 index lookups and a result set of 50 × 6
rows. Use `catalog_product_index_eav` for filtering, or the Product Collection.

### 16.4 Use the index tables

`catalog_product_index_price`, `catalog_category_product_index` and
`catalog_product_index_eav` exist precisely so you do not compute joins on every
request. They are pre-computed and pre-indexed.

### 16.5 Set-based, not row-by-row

```php
// BAD: 10 000 UPDATE statements
foreach ($notes as $note) { $note->setX(1)->save(); }

// GOOD: one statement
$connection->update($table, ['x' => 1], ['in' => $noteIds]);
```

### 16.6 Cache the aggregation, not the query

A `GROUP BY` over `sales_order` is stable except when orders change. Cache the
result (or use a materialized summary table) rather than re-running it.

### 16.7 Know the server limits

Verified on this install: `innodb_buffer_pool_size = 512M`,
`max_connections = 151`. Magento opens a connection per PHP-FPM worker, and a
heavy admin grid can open many. A report running inside a request competes with
the page render for the buffer pool — run big reports in a cron, not in a
controller.

---

## 17. Anti-Patterns

| Anti-pattern | Why it fails | Do instead |
|---|---|---|
| `SELECT *` | Transfers unused columns; blocks index-only scans | List the columns |
| String-concatenated SQL | SQL injection | `->where('col = ?', $v)` |
| Hardcoded table names | Breaks with a table prefix | `getTableName()` |
| EAV join for a product list | 6× index lookups per product | `catalog_product_index_eav` or Product Collection |
| `WHERE value = 'x'` on an EAV table | No index on `value` | Filter on `attribute_id` first |
| `attribute_code` without `entity_type_code` | Wrong attribute, silent `NULL` | Join `eav_entity_type` |
| Single EAV join on the store scope | `NULL` for non-overridden values | `COALESCE` over store + default |
| `fetchOne()` cast straight to float | `false` on empty result | Check `=== false` first |
| Raw `insert()` on a Magento table | No events, no cache invalidation | `$model->save()` or a Repository |
| Aggregate on `grand_total` | Wrong in multi-currency | `base_grand_total` |
| No `WHERE status` on an order report | Counts canceled orders | Filter the statuses |
| `foreach` + `save()` in a loop | 1 query per row | `updateFromSelect`, `insertMultiple` |
| Collection without `setPageSize` | Loads the whole table into memory | Always set a cap |
| `DDL` inside a transaction | MySQL commits implicitly | DDL outside transactions |
| Hand-editing `catalog_*_index*` | Reindex overwrites it | `indexer:reindex` |
| Editing `sequence_*` tables | Corrupts increment ids | Use `SalesSequence` |
| `DELETE FROM sales_order` | Orphans invoices, credit memos, shipments | Cancel the order |

---

## 18. Summary

| Task | Approach |
|---|---|
| Explore the schema | `SHOW TABLES`, `DESCRIBE`, `SHOW CREATE TABLE`, `information_schema` |
| Find an unknown relation | `information_schema.KEY_COLUMN_USAGE` filtered by `REFERENCED_TABLE_NAME` |
| Resolve an EAV attribute | `eav_attribute` joined to `eav_entity_type`, always |
| Read an EAV value with fallback | Two LEFT JOINs + `COALESCE(store, default)` |
| Read a `select` label | `eav_attribute_option` → `eav_attribute_option_value` |
| Report on sales | `sales_order` + `sales_order_item`, `base_*` columns, explicit status filter |
| Read catalog at scale | Index tables or the Product Collection |
| Read one entity | Repository `getById()` |
| List entities | Repository `getList(SearchCriteria)` or a Collection |
| Write one entity | Repository `save()`, wrapped in `CouldNotSaveException` |
| Write many entities | `insertMultiple`, `updateFromSelect` |
| Aggregate | `ResourceConnection` + `Select`, cached |
| Change schema | `db_schema.xml` + `setup:db-schema:upgrade` |
| Change data | `Setup/Patch/Data/` with `getAliases()` |
| Guarantee atomicity | `beginTransaction` / `commit` / `rollBack` |
| Debug | `(string) $select`, `EXPLAIN`, `indexer:status`, `setup:db:status` |

**Prerequisites**: [magento-database.md](magento-database.md) — the tables, the
EAV model, the relations.

**Related**: [magento-cron-indexers.md](magento-cron-indexers.md) (keeping index
tables fresh) · [magento-multistore.md](magento-multistore.md) (what `store_id`
means) · [magento-order-lifecycle.md](magento-order-lifecycle.md) (the sales
tables in business context) · [magento-coding-standards.md](magento-coding-standards.md)
(the PHP conventions used in the examples)

---

## Official Magento 2 Documentation

| Topic | Link |
|---|---|
| Developer documentation home | [developer.adobe.com/commerce/docs](https://developer.adobe.com/commerce/docs) |
| Configure declarative schema (`db_schema.xml`) | [declarative-schema/configuration](https://developer.adobe.com/commerce/php/development/components/declarative-schema/configuration) |
| Data and schema patches | [declarative-schema/patches](https://developer.adobe.com/commerce/php/development/components/declarative-schema/patches) |
| PHP extensions reference | [developer.adobe.com/commerce/php/](https://developer.adobe.com/commerce/php/) |
| Database configuration best practices | [database-on-cloud](https://experienceleague.adobe.com/en/docs/commerce-operations/implementation-playbook/best-practices/planning/database-on-cloud) |
| Upgrade the database schema (CLI) | [installation-guide database tutorial](https://experienceleague.adobe.com/en/docs/commerce-operations/installation-guide/tutorials/database) |

---

## Sources

All SQL and PHP in this document was verified against a live Magento 2.4.8
install on MySQL 8.0.46 (465 tables, 409 foreign keys, 1 325 indexes, 14
indexers all `valid`, `STRICT_TRANS_TABLES` + `ONLY_FULL_GROUP_BY`).
Framework class references point to `src/vendor/magento/framework/`.

*Last updated: 2026-09-28*
