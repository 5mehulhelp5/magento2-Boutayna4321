# Magento 2 — Database Deep Dive

> **Objective**: understand how Magento 2 stores its data: the EAV model,
> the flat tables, how tables relate to each other, what `store_id` means,
> and how to read the schema without guessing.

All schemas and row counts in this document come from a live **Magento 2.4.8**
install running on **MySQL 8.0.46** (`utf8mb4` / `utf8mb4_unicode_ci`, InnoDB),
with **465 tables**. Every statement marked "verified" was executed against
that database.

**Official documentation**:
- [Declarative schema configuration](https://developer.adobe.com/commerce/php/development/components/declarative-schema/configuration)
- [Data and schema patches](https://developer.adobe.com/commerce/php/development/components/declarative-schema/patches)
- [Site, store, and view scope](https://experienceleague.adobe.com/en/docs/commerce-admin/start/setup/websites-stores-views)

---

## Table of Contents

1. [Connecting to the Database](#1-connecting-to-the-database)
2. [Two Data Models: Flat Tables and EAV](#2-two-data-models-flat-tables-and-eav)
3. [The EAV Model in Detail](#3-the-eav-model-in-detail)
4. [EAV Store Scoping and Fallback](#4-eav-store-scoping-and-fallback)
5. [The Flat Model in Detail](#5-the-flat-model-in-detail)
6. [Table Families](#6-table-families)
7. [Core Relations](#7-core-relations)
8. [Order Lifecycle Tables](#8-order-lifecycle-tables)
9. [Quote to Order](#9-quote-to-order)
10. [Catalog Index Tables](#10-catalog-index-tables)
11. [Inventory Tables (MSI)](#11-inventory-tables-msi)
12. [Store, Website and Configuration Tables](#12-store-website-and-configuration-tables)
13. [Reading the Schema](#13-reading-the-schema)
14. [Database Best Practices](#14-database-best-practices)
15. [Summary](#15-summary)

---

## 1. Connecting to the Database

### 1.1 Where the credentials live

Magento stores the connection in `app/etc/env.php` under `db.connection.default`:

```php
'db' => [
    'connection' => [
        'default' => [
            'host'     => 'mysql',
            'dbname'   => 'magento2',
            'username' => 'magento',
            'prefix'   => '',            // empty = no table prefix
            'engine'   => 'innodb',
            'init_commands' => [
                "SET sql_mode='STRICT_TRANS_TABLES'",
            ],
        ],
    ],
],
```

**Verified on this install**: `table_prefix` is `''` and `dbname` is `magento2`.
The MySQL server runs with `sql_mode = ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES,
NO_ZERO_IN_DATE, NO_ZERO_DATE, ERROR_FOR_DIVISION_BY_ZERO,
NO_ENGINE_SUBSTITUTION`, `innodb_buffer_pool_size = 512M`, `max_connections = 151`.

> **Strict mode matters**: `ONLY_FULL_GROUP_BY` rejects any `SELECT` where a
> non-aggregated column is not in the `GROUP BY`. This is the single most common
> "works in phpMyAdmin, fails in Magento" error. See
> [Queries guide, section 8](magento-database-queries.md#8-group-by-and-only_full_group_by).

### 1.2 Connecting from the CLI

```bash
# Docker environment
docker exec -it magento2-mysql mysql -umagento -p magento2

# Native MySQL client
mysql -h127.0.0.1 -P3306 -umagento -p magento2
```

Useful flags: `--table` for aligned output, `-N` for raw (no header) output,
`-e "<query>"` to run a single statement.

### 1.3 Table prefixes

`'prefix' => ''` here, but Magento supports one. With a prefix of `mg_`, the
table `catalog_product_entity` becomes `mg_catalog_product_entity`. **Never hardcode
a table name in PHP** — always resolve it through `ResourceConnection::getTableName()`.

---

## 2. Two Data Models: Flat Tables and EAV

Magento uses **two different data models side by side**. Understanding which one
a table uses explains most of its structure.

| | **Flat model** | **EAV model** |
|---|---|---|
| Shape | One table, real columns | Split across 3+ tables |
| Schema | Fixed, one column per attribute | Attribute rows describe the columns |
| Stores | `store_id` column on the table | `store_id` column on the *value* row |
| Examples | `sales_order`, `quote`, `catalog_product_entity` | `name`, `description`, `color` of a product |
| Used for | Sales, checkout, inventory, orders, customers, CMS | Product and category custom attributes |
| Query cost | Simple `SELECT` | Join per attribute read |

**Verified on this install** (465 tables total):

| Model | Example table | Rows |
|---|---|---|
| Flat | `catalog_product_entity` | 2 054 products |
| Flat | `sales_order` | 36 orders |
| Flat | `customer_entity` | 11 customers |
| EAV | `catalog_product_entity_varchar` | `name`, `description`, `url_key`, `image` |
| EAV | `catalog_product_entity_int` | `color`, `status`, `visibility` |
| EAV | `catalog_product_entity_decimal` | `price`, `weight` |
| EAV | `catalog_product_entity_text` | short text values |
| EAV | `catalog_product_entity_datetime` | date values |

The `catalog` prefix alone accounts for **118 tables**, `sales` for **48**,
`customer` for **21**.

### 2.1 Why both?

- **EAV** lets an administrator add a product attribute ("Material", "Climate")
  from the admin panel without a developer and without a schema change. That
  flexibility is why `catalog_product_entity` itself only has 8 columns:

```
catalog_product_entity
├── entity_id          (PK, auto_increment)
├── attribute_set_id
├── type_id            simple / configurable / bundle / virtual / grouped / downloadable
├── sku                (indexed)
├── has_options
├── required_options
├── created_at
└── updated_at
```

- **Flat** is used wherever an attribute is essential to the business logic, so
  that queries stay fast and predictable. `sales_order` has ~80 columns because
  every total is stored directly.

---

## 3. The EAV Model in Detail

### 3.1 The three layers

```
eav_entity_type          Which entity: product, category, customer, order, ...
        │  entity_type_id
        ▼
eav_attribute            What attributes exist for that entity
        │  attribute_id
        ▼
catalog_product_entity   The rows (one per product)
        │  entity_id
        ▼
catalog_product_entity_* The values, split by PHP type
```

**Verified** — `eav_entity_type` contents:

| entity_type_id | entity_type_code | entity_table |
|---|---|---|
| 1 | `customer` | `customer_entity` |
| 2 | `customer_address` | `customer_address_entity` |
| 3 | `catalog_category` | `catalog_category_entity` |
| 4 | `catalog_product` | `catalog_product_entity` |
| 5 | `order` | `sales_order` |
| 6 | `invoice` | `sales_invoice` |
| 7 | `creditmemo` | `sales_creditmemo` |
| 8 | `shipment` | `sales_shipment` |

> **Key insight**: an entity type can be EAV *and* have a flat base table.
> `order` is an EAV entity type (EAV attributes can be added to orders), yet
> `sales_order` itself is a flat table with real columns. "Entity type" and
> "EAV" are not synonyms.

### 3.2 The value tables: one per PHP type

`eav_attribute.backend_type` decides which `*_entity_*` table holds the value.

| `backend_type` | Table | Verified column type |
|---|---|---|
| `varchar` | `catalog_product_entity_varchar` | `varchar(255)` |
| `int` | `catalog_product_entity_int` | `int` |
| `decimal` | `catalog_product_entity_decimal` | `decimal(20,6)` |
| `text` | `catalog_product_entity_text` | `text` |
| `datetime` | `catalog_product_entity_datetime` | `datetime` |

**Verified** — the structure of `catalog_product_entity_decimal`:

```
value_id      int unsigned     PK, auto_increment
attribute_id  smallint unsigned  → eav_attribute.attribute_id
store_id      smallint unsigned  → store.store_id
entity_id     int unsigned       → catalog_product_entity.entity_id
value         decimal(20,6)
```

**Verified indexes** on `catalog_product_entity_varchar`:

| Index | Columns | Type |
|---|---|---|
| `PRIMARY` | `value_id` | unique |
| `CATALOG_PRODUCT_ENTITY_VARCHAR_ENTITY_ID_ATTRIBUTE_ID_STORE_ID` | `entity_id, attribute_id, store_id` | **unique** |
| `CATALOG_PRODUCT_ENTITY_VARCHAR_ATTRIBUTE_ID` | `attribute_id` | non-unique |
| `CATALOG_PRODUCT_ENTITY_VARCHAR_STORE_ID` | `store_id` | non-unique |

The unique composite index is the one that makes EAV reads fast: the engine
looks up exactly one row for `(entity, attribute, store)`.

**Verified foreign keys** on `catalog_product_entity_varchar` — all three with
`ON DELETE CASCADE`:

| Constraint | Local column | References |
|---|---|---|
| `CAT_PRD_ENTT_VCHR_ATTR_ID_EAV_ATTR_ATTR_ID` | `attribute_id` | `eav_attribute.attribute_id` |
| `CAT_PRD_ENTT_VCHR_ENTT_ID_CAT_PRD_ENTT_ENTT_ID` | `entity_id` | `catalog_product_entity.entity_id` |
| `CATALOG_PRODUCT_ENTITY_VARCHAR_STORE_ID_STORE_STORE_ID` | `store_id` | `store.store_id` |

This is the one place where Magento *does* enforce referential integrity in the
catalog: deleting a product or deleting an attribute removes its EAV value rows
automatically. That is exactly why the sales tables deliberately have **no**
such FK — see [section 7.2](#72-where-the-boundaries-are-and-where-they-are-not).

Note also that `value` is `varchar(255)` with `COLLATE utf8mb4_general_ci`,
while the table default collation on this database is `utf8mb4_unicode_ci`.
MySQL column collations differ; that only matters for a `GROUP BY value`, which
you should avoid anyway.

### 3.3 Reading a product attribute — the full chain

To read the `name` of product `24-MB01`:

```
1. eav_entity_type          WHERE entity_type_code = 'catalog_product'  → 4
2. eav_attribute            WHERE entity_type_id = 4
                             AND attribute_code = 'name'                 → 73
3. catalog_product_entity   WHERE sku = '24-MB01'                        → entity_id 1
4. catalog_product_entity_varchar
                           WHERE entity_id = 1
                             AND attribute_id = 73
                             AND store_id = 0                            → 'Joust Duffle Bag'
```

**Step 2 must be scoped by entity type.** `attribute_code` is only unique *within*
an entity type. **Verified**: `attribute_code = 'name'` exists twice —
`attribute_id 45` for `catalog_category` and `attribute_id 73` for
`catalog_product`. Looking up `'name'` without the entity type returns the
category one (or nothing, depending on row order) and silently returns
`NULL` for products. This is a classic bug. The [queries guide](magento-database-queries.md#3-eav-queries)
has the correct SQL.

### 3.4 Attribute options: `select` / `multiselect`

A `select` attribute stores an **option_id** in the `int` value table, not a
label. The label lives in two more tables.

```
catalog_product_entity_int        (value = 45, the option_id)
        │  option_id
        ▼
eav_attribute_option              (attribute_id, sort_order)
        │  option_id
        ▼
eav_attribute_option_value        (value = 'Blue', store_id = 0)
```

**Verified**: `eav_attribute_option` holds 211 rows in this install. The
attributes with the most options are `material` (36), `style_general` (26),
`size` (20), `activity` (20), `color` (12).

`eav_attribute_option_value` is itself store-scoped, so option labels can be
translated per store view.

### 3.5 Attribute sets

Products and categories belong to an **attribute set**, which determines which
attributes apply. A configurable product needs `color` and `size`; a bag does not.

```
eav_attribute_set          attribute_set_id, entity_type_id, attribute_set_name
        │
        │  attribute_set_id
        ▼
eav_attribute_group        attribute_group_id, attribute_set_id, attribute_group_name
        │
        │  attribute_group_id + attribute_set_id
        ▼
eav_entity_attribute       entity_type_id, attribute_set_id, attribute_group_id, attribute_id
```

**Verified schemas**:

- `eav_attribute_set(attribute_set_id PK, entity_type_id, attribute_set_name, sort_order)`
- `eav_attribute_group(attribute_group_id PK, attribute_set_id, attribute_group_name, sort_order, default_id, attribute_group_code, tab_group_code)`
- `eav_entity_attribute(entity_attribute_id PK, entity_type_id, attribute_set_id, attribute_group_id, attribute_id, sort_order)`

There is no separate "set → attribute" table: the relation lives in
`eav_entity_attribute`, which is simultaneously the set membership list **and**
the attribute sort order for that set.

**Verified** — attribute sets for `entity_type_id = 4` (catalog_product):
`4 Default`, `10 Bottom`, `11 Gear`, `14 Downloadable`, `15 Bag`, …

```sql
-- Which attributes belong to the "Bag" set (attribute_set_id = 15)?
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

`catalog_product_entity.attribute_set_id` points here. **Verified** value: every
sample product uses `attribute_set_id = 15` (the "Bag" set).

### 3.6 The EAV anti-patterns to avoid

| Trap | Why it breaks | Do instead |
|---|---|---|
| Looking up `attribute_id` without `entity_type_id` | Same code, different attribute | Always join `eav_entity_type` |
| `SELECT * FROM catalog_product_entity_varchar WHERE value = 'x'` | No index on `value`; full scan of every product attribute | Filter on `attribute_id` first |
| Reading a `select` label from the `int` value | You get an integer | Join through `eav_attribute_option` |
| Joining 6 EAV tables for a list of 50 products | 6 × 50 index lookups + huge result set | Use `catalog_product_index_eav` or a Collection with `addAttributeToSelect` |
| Assuming `store_id = 0` always exists | It holds the *default* value, not a fallback row per store | Apply the fallback rule (section 4) |

---

## 4. EAV Store Scoping and Fallback

### 4.1 `store_id` in EAV

Every EAV value row carries a `store_id`:

- `store_id = 0` → the **admin / default** value. One row per entity+attribute.
- `store_id = N` → a **store-view-specific override**. Only exists if somebody
  changed the value in that store view's scope.

**Verified** on this install: `attribute_id 73` (`name`, product) has
**2 042 rows, all with `store_id = 0`**. Nobody has overridden a product name
per store view. This is the normal state of a fresh install.

### 4.2 The fallback rule

Magento reads an EAV attribute like this:

```
for store_id in (requested_store, 0):
    if a row exists for (entity, attribute, store_id):
        return its value
return null
```

A raw SQL join must reproduce this with a `COALESCE` over two LEFT JOINs, or
you will return `NULL` for every product whose value was never overridden.
`Magento\Eav\Model\Entity\Attribute\AbstractAttribute` does this for you; the
[queries guide, section 3.2](magento-database-queries.md#32-reading-an-eav-value-with-store-fallback)
shows the raw SQL equivalent.

### 4.3 Which tables are actually store-scoped

Not every EAV table has `store_id`. **Verified**: `customer_entity_varchar` and
`customer_entity_int` have **no `store_id` column** — a customer's name is not
per store view, it is per **website** (`customer_entity.website_id`).

| Scoped by | Tables |
|---|---|
| `store_id` | `catalog_product_entity_*`, `catalog_category_entity_*`, `customer_address_entity_*`, `eav_attribute_option_value` |
| `website_id` | `customer_entity`, `catalog_product_website` |
| Both present | `sales_order`, `quote`, `sales_invoice`, `sales_creditmemo` |

---

## 5. The Flat Model in Detail

### 5.1 Naming conventions

Flat table naming follows strict conventions, declared in each module's
`etc/db_schema.xml` (for example `src/vendor/magento/module-store/etc/db_schema.xml`).
A ResourceModel binds them together with `$this->_init('table_name', 'id_column')`.

| Convention | Example |
|---|---|
| Primary key is `entity_id` | `sales_order.entity_id` |
| Business key is `<type>_id` | `sales_order.increment_id`, `sales_invoice.increment_id` |
| Parent FK is `<parent>_id` | `sales_order_item.order_id` |
| Hierarchical FK | `sales_order_item.parent_item_id` (configurable child) |
| Timestamps | `created_at`, `updated_at` |
| Booleans | `is_*` / `has_*` as `smallint` (0/1) |
| Enums | `varchar` validated by `*_status` / `*_state` lookup tables |

### 5.2 `sales_order` — the reference flat table

**Verified**: `sales_order` has ~80 columns. They fall into four groups.

| Group | Example columns |
|---|---|
| Identity | `entity_id`, `store_id`, `increment_id`, `customer_id`, `quote_id`, `ext_order_id` |
| Status | `state`, `status`, `protect_code` |
| Money (quote currency) | `grand_total`, `subtotal`, `shipping_amount`, `tax_amount`, `discount_amount` |
| Money (base currency, prefixed `base_`) | `base_grand_total`, `base_subtotal`, `base_tax_amount`, `base_to_global_rate` |

> **The `base_` prefix**: every monetary column exists twice — once in the
> *quote* currency and once in the *base* currency. `base_*` is the truth for
> accounting. A report must use `base_*` columns, or the same query returns
> different numbers depending on the store view the visitor used.

Status columns reference lookup tables. **Verified** — `sales_order_status`:

| status | label |
|---|---|
| `pending` | Pending |
| `processing` | Processing |
| `holded` | On Hold |
| `payment_review` | Payment Review |
| `fraud` | Suspected Fraud |
| `closed` | Closed |
| `complete` | Complete |
| `canceled` | Canceled |

And `sales_order_status_state` groups them into higher-level `state` values:

| status | state | is_default |
|---|---|---|
| `pending` | `new` | 1 |
| `fraud` | `payment_review` | 0 |
| `fraud` | `processing` | 0 |
| `processing` | `processing` | 1 |
| `complete` | `complete` | 1 |
| `canceled` | `canceled` | 1 |
| `holded` | `holded` | 1 |
| `closed` | `closed` | 1 |

> **Why this matters**: a status can map to *several* states (`fraud` → both
> `payment_review` and `processing`). Filtering with `state = 'processing'` is
> broader than `status = 'processing'`. Pick deliberately.

**Verified indexes** on `sales_order` include a composite
`SALES_ORDER_STORE_ID_STATE_CREATED_AT` on `(store_id, state, created_at)` — built
for the admin order grid, and exactly the shape you want for a date-ranged
report on one store.

### 5.3 `sales_order_item` — the snapshot principle

`sales_order_item` **duplicates** the product data at purchase time:

| Snapshot column | Why |
|---|---|
| `sku`, `name`, `description` | The product may be renamed or deleted later; the invoice must not change |
| `price`, `base_price`, `original_price` | Catalog price may change; the order price is fixed |
| `product_options` (`longtext`) | Serialized custom options at purchase time |
| `applied_rule_ids` (`text`) | Which cart price rules were applied |
| `parent_item_id` | Link from a configurable child to its parent line |

Because of this, **never join `sales_order_item` back to the catalog to build an
order report**. The join is both wrong (you get today's name and price) and slow.

---

## 6. Table Families

| Family | Key tables | Model |
|---|---|---|
| **Catalog base** | `catalog_product_entity`, `catalog_category_entity` | Flat base + EAV attributes |
| **Catalog EAV** | `catalog_product_entity_{int,varchar,decimal,text,datetime}` | EAV |
| **Catalog links** | `catalog_category_product`, `catalog_product_website`, `catalog_product_relation` | Flat pivot |
| **Catalog options** | `catalog_product_option`, `catalog_product_option_type_value`, `catalog_product_super_attribute`, `catalog_product_super_link` | Flat |
| **Catalog index** | `catalog_product_index_price`, `catalog_category_product_index`, `catalog_product_index_eav` | Derived index |
| **Prices/rules** | `catalogrule_product_price`, `catalogrule_product`, `catalogrule_website`, `catalog_product_index_price` | Flat |
| **Sales** | `sales_order`, `sales_order_item`, `sales_order_address`, `sales_order_status_history` | Flat |
| **Documents** | `sales_invoice`, `sales_creditmemo`, `sales_shipment` (+ `_item`, `_comment`, `_track`) | Flat |
| **Checkout** | `quote`, `quote_item`, `quote_address`, `quote_payment`, `quote_shipping_rate` | Flat |
| **Customer** | `customer_entity`, `customer_address_entity`, `customer_group` | Flat + EAV |
| **Inventory (MSI)** | `inventory_source`, `inventory_source_item`, `cataloginventory_stock_item` | Flat |
| **Tax** | `tax_class`, `tax_calculation`, `tax_calculation_rate` | Flat |
| **Store** | `store`, `store_group`, `store_website` | Flat |
| **Config** | `core_config_data` | Flat |
| **Review** | `review`, `review_detail`, `rating_option_vote` | Flat |
| **System** | `admin_user`, `admin_user_session`, `cron_schedule`, `queue`, `indexer_state`, `flag` | Flat |
| **Sequence** | `sequence_order_0`, `sequence_invoice_0`, … | Single counter column |

**Verified**: `catalog_category_product` is a pivot with a `position` column,
holding `(entity_id, category_id, product_id, position)` — it orders products
inside a category.

**Verified**: sequence tables each have exactly **one** column,
`sequence_value int unsigned PRIMARY KEY AUTO_INCREMENT`, and are named
`sequence_<type>_<shard>`. There are shards `0` and `1` per entity type
(`sequence_order_0`, `sequence_order_1`, `sequence_invoice_0`, …). The shard
selection is handled by `Magento\SalesSequence\Model\Sequence`, so **never
increment a sequence table by hand**.

---

## 7. Core Relations

### 7.1 Relationship map

```
store_website ──< store ──< store_group
     │              │
     │              └──< catalog_product_website >── catalog_product_entity
     │
     └──< customer_entity ──< customer_address_entity
                    │                    │
                    │                    └──< customer_entity_varchar  (EAV)
                    │
                    ├──< sales_order ──< sales_order_item
                    │            │              │        │
                    │            │              │        └── parent_item_id (self)
                    │            │              └──< sales_invoice_item >── sales_invoice
                    │            │              └──< sales_creditmemo_item >── sales_creditmemo
                    │            │              └──< sales_shipment_item >── sales_shipment
                    │            ├──< sales_order_address
                    │            ├──< sales_order_status_history
                    │            ├──< sales_order_tax / _tax_item
                    │            ├──< sales_payment_transaction
                    │            └── quote_id ──> quote.entity_id
                    │
                    ├──< quote ──< quote_item ──< quote_item_option
                    │       ├──< quote_address ──< quote_address_item
                    │       ├──< quote_payment
                    │       ├──< quote_shipping_rate
                    │       └── customer_id ──> customer_entity.entity_id

catalog_category_entity ──< catalog_category_product >── catalog_product_entity
        │                                                  │
        └── path, level, parent_id (self-referencing)      └──< catalog_product_entity_*
```

### 7.2 Where the boundaries are, and where they are not

Magento declares **409 foreign keys** (verified). The split is deliberate, and
understanding *which* side is enforced is what tells you what a `DELETE` will
actually do.

**Cascade — the child cannot outlive its parent:**

| Constraint | Local | References |
|---|---|---|
| `SALES_ORDER_ITEM_ORDER_ID_SALES_ORDER_ENTITY_ID` | `sales_order_item.order_id` | `sales_order.entity_id` |
| `SALES_INVOICE_ORDER_ID_SALES_ORDER_ENTITY_ID` | `sales_invoice.order_id` | `sales_order.entity_id` |
| `SALES_ORDER_ADDRESS_PARENT_ID_SALES_ORDER_ENTITY_ID` | `sales_order_address.parent_id` | `sales_order.entity_id` |
| `SALES_INVOICE_ITEM_PARENT_ID_SALES_INVOICE_ENTITY_ID` | `sales_invoice_item.parent_id` | `sales_invoice.entity_id` |
| `QUOTE_ITEM_QUOTE_ID_QUOTE_ENTITY_ID` | `quote_item.quote_id` | `quote.entity_id` |
| `QUOTE_ITEM_PARENT_ITEM_ID_QUOTE_ITEM_ITEM_ID` | `quote_item.parent_item_id` | `quote_item.item_id` |
| `CAT_PRD_LNK_LNKED_PRD_ID_CAT_PRD_ENTT_ENTT_ID` | `catalog_product_link.linked_product_id` | `catalog_product_entity.entity_id` |
| `CATALOG_PRODUCT_LINK_PRODUCT_ID_CATALOG_PRODUCT_ENTITY_ENTITY_ID` | `catalog_product_link.product_id` | `catalog_product_entity.entity_id` |
| `CAT_PRD_ENTT_VCHR_ATTR_ID_EAV_ATTR_ATTR_ID` | `catalog_product_entity_varchar.attribute_id` | `eav_attribute.attribute_id` |
| `CAT_PRD_ENTT_VCHR_ENTT_ID_CAT_PRD_ENTT_ENTT_ID` | `catalog_product_entity_varchar.entity_id` | `catalog_product_entity.entity_id` |

So `DELETE FROM sales_order` **does** cascade to `sales_order_item`,
`sales_invoice`, `sales_order_address`. That is precisely why the
[best practices](magento-database-queries.md#17-anti-patterns) forbid deleting
an order: it is not a soft orphan, it destroys the invoice and the credit memo
with it.

**Set null — the reference is meaningful but must survive the parent:**

| Constraint | Local | On delete |
|---|---|---|
| `SALES_ORDER_CUSTOMER_ID_CUSTOMER_ENTITY_ENTITY_ID` | `sales_order.customer_id` | `SET NULL` |
| `SALES_ORDER_ITEM_STORE_ID_STORE_STORE_ID` | `sales_order_item.store_id` | `SET NULL` |
| `SALES_INVOICE_STORE_ID_STORE_STORE_ID` | `sales_invoice.store_id` | `SET NULL` |
| `QUOTE_STORE_ID_STORE_STORE_ID` | `quote.store_id` | `SET NULL` |

An order survives its customer being deleted, with `customer_id = NULL` — the
row becomes a guest order. This is how GDPR erasure works: the account goes,
the accounting record stays.

**No foreign key at all (verified absent):**

- `sales_order_item.product_id` → `catalog_product_entity.entity_id`
- `quote.customer_id` → `customer_entity.entity_id`
- `sales_order_item.quote_id` → `quote.entity_id`

`sales_order_item.product_id` is the important one. The order line stores
`sku`, `name` and `price` as snapshots precisely because the product reference
is intentionally unenforced: a deleted product must not destroy or invalidate
the historical order line. There is no cascade to accidentally trigger, and no
`SET NULL` to erase the link.

> **Practical consequence**: joining `sales_order_item` back to
> `catalog_product_entity` can return fewer rows than you have order items —
> a deleted product drops out. Use `LEFT JOIN` in any report that must account
> for every purchased line, and read the SKU and name from
> `sales_order_item` itself.
>
> If you need referential safety on your own module tables, declare
> `xsi:type="foreign"` in `db_schema.xml` — see
> [queries guide, section 7](magento-database-queries.md#7-creating-a-table).

### 7.3 The self-referencing category tree

`catalog_category_entity` stores the whole path, not just the parent:

**Verified** — `SHOW CREATE TABLE catalog_category_entity` columns:
`entity_id`, `parent_id`, `path` (`varchar(255)`, e.g. `1/2`),
`path_length`, `level` (`smallint`), `position`, `children_count`, `created_at`, `updated_at`.

```sql
-- A category and its direct children (verified output)
SELECT c.entity_id, c.parent_id, c.path, c.level, c.children_count, n.value AS name
FROM catalog_category_entity c
LEFT JOIN catalog_category_entity_varchar n
       ON n.entity_id = c.entity_id AND n.store_id = 0 AND n.attribute_id = 45
WHERE c.path = '1/2';
```

Result on this install:

```
+-------------+-----------+------+-------+----------------+------------------+
| category_id | parent_id | path | level | children_count | name             |
+-------------+-----------+------+-------+----------------+------------------+
|           2 |         1 | 1/2  |     1 |             38 | Default Category |
+-------------+-----------+------+-------+----------------+------------------+
```

`path` is a materialized path: to get descendants of category 2, filter
`path LIKE '1/2/%'`. To get ancestors, `FIND_IN_SET(parent_id, path)`. This is
why the tree is stored denormalized — one indexed `LIKE` instead of a recursive
query per level.

### 7.4 The product ↔ category ↔ website triangle

A product is visible in the catalog only if it is **both** assigned to at least
one category **and** enabled on at least one website. Both are pivot tables:

**Verified** — `catalog_product_website` rows: `(1,1)`, `(2,1)`, `(3,1)`.
**Verified** — `catalog_category_product` rows: `(1,3,1,0)`, `(2,4,1,0)`, `(3,3,2,0)`.

```sql
-- Products visible on website 1, in category 3, with SKU
SELECT e.entity_id, e.sku
FROM catalog_product_entity e
JOIN catalog_product_website pw ON pw.product_id = e.entity_id
JOIN catalog_category_product cp ON cp.product_id = e.entity_id
WHERE pw.website_id = 1
  AND cp.category_id = 3;
```

---

## 8. Order Lifecycle Tables

One order can produce many invoices, many credit memos and many shipments.
All three point back at `sales_order.entity_id`.

```
sales_order (entity_id, increment_id, state, status, grand_total)
     │
     ├──< sales_order_item          (order_id)         the purchased lines
     ├──< sales_order_address       (parent_id)        billing + shipping snapshots
     ├──< sales_order_status_history(order_id)         the audit trail of state changes
     ├──< sales_order_tax / _tax_item (order_id)       tax lines
     ├──< sales_order_grid                            admin grid denormalization
     │
     ├──< sales_invoice (order_id) ──< sales_invoice_item (parent_id, order_item_id)
     ├──< sales_creditmemo (order_id) ──< sales_creditmemo_item (parent_id)
     └──< sales_shipment (order_id) ──< sales_shipment_item (parent_id, order_item_id)
```

**Verified rollup** (one row per order, correlated subqueries):

```sql
SELECT o.entity_id AS order_id, o.increment_id, o.status, o.grand_total,
       (SELECT COUNT(*) FROM sales_invoice    i  WHERE i.order_id  = o.entity_id) AS nb_invoices,
       (SELECT COUNT(*) FROM sales_creditmemo cm WHERE cm.order_id = o.entity_id) AS nb_creditmemos,
       (SELECT COUNT(*) FROM sales_shipment   s  WHERE s.order_id  = o.entity_id) AS nb_shipments
FROM sales_order o
ORDER BY o.entity_id
LIMIT 10;
```

Real output from this install:

```
+-----------+--------------+------------+-------------+-------------+----------------+--------------+
| order_id  | increment_id | status     | grand_total | nb_invoices | nb_creditmemos | nb_shipments |
+-----------+--------------+------------+-------------+-------------+----------------+--------------+
|         1 | 000000001    | processing |     36.3900 |           1 |              0 |            1 |
|         2 | 000000002    | closed     |     39.6400 |           1 |              1 |            1 |
|         3 | 000000003    | pending    |    205.0000 |           1 |              0 |            0 |
|         7 | 000000007    | canceled   |    570.0000 |           0 |              0 |            0 |
+-----------+--------------+------------+-------------+-------------+----------------+--------------+
```

Note the naming: `sales_invoice` and `sales_creditmemo` store `state` as an
**int** (verified: `state` = `2`), while `sales_order.state` is a **varchar**
status string. Always check the column type before comparing.

---

## 9. Quote to Order

A quote is a cart. `quote` and `quote_item` are structurally twins of
`sales_order` and `sales_order_item`, and at order placement Magento copies one
into the other.

| Quote | Order | Difference |
|---|---|---|
| `quote.entity_id` | `sales_order.entity_id` | — |
| `quote_item.qty` | `sales_order_item.qty_ordered` | `qty` vs `qty_*` family |
| `quote.grand_total` | `sales_order.grand_total` | — |
| — | `sales_order.quote_id` | **FK back to the quote** |

**Verified** — `quote_item` has `quote_id`, `parent_item_id`, `product_id`,
`store_id`, `sku`, `name`, `qty`, `price`, `row_total`,
`row_total_with_discount`, `product_options` and the full tax/discount family.
The `_with_discount` column exists on the quote but not on the order item.

Other quote tables: `quote_address` (billing/shipping forms),
`quote_address_item` (address line items), `quote_payment` (payment method),
`quote_shipping_rate` (shipping methods with `shipping_amount`),
`quote_item_option` (selected custom options), `quote_id_mask` (hashed quote id
for guest carts).

**Verified** — active carts with their items:

```sql
SELECT q.entity_id AS quote_id, q.store_id, q.customer_email, q.items_qty,
       qi.item_id, qi.sku, qi.name, qi.qty, qi.row_total
FROM quote q
JOIN quote_item qi ON qi.quote_id = q.entity_id
WHERE q.is_active = 1
ORDER BY q.entity_id, qi.item_id
LIMIT 20;
```

> `is_active = 1` is the only reliable filter for "current carts". A quote with
> `is_active = 0` is a converted or abandoned cart kept for reporting. Guest
> carts have `customer_email` and `customer_id` NULL — a missing email does not
> mean a broken row.

---

## 10. Catalog Index Tables

The `catalog_*_index*` tables are **derived data**. Never read them as truth,
never write to them by hand.

| Table | Rebuilt by indexer | Contains |
|---|---|---|
| `catalog_product_index_price` | `catalog_product_price` | `min_price`, `max_price`, `final_price` per (entity, customer_group, website, tax_class) |
| `catalog_category_product_index` | `catalog_product_category` | category membership + position per store |
| `catalog_product_index_eav` | `catalog_product_attribute` | attribute values denormalized for filtering |
| `catalog_product_website` | `catalog_product_website` | website assignment |

> **Search is not a table.** This install has no `catalogsearch_fulltext` table
> (verified: only `catalogsearch_fulltext_cl`, the staging table, exists). With
> OpenSearch or Elasticsearch configured, the `catalogsearch_fulltext` indexer
> writes the search document to the search engine, not to MySQL. Never write a
> search query against MySQL.

**Verified**: `catalog_product_index_price` has a composite PK on
`(entity_id, customer_group_id, website_id)`. One product has **one row per
customer group**, which is why a naive join to `catalog_product_entity`
duplicates products.

```sql
-- Verified: always filter the dimensions
SELECT p.entity_id, e.sku, p.min_price, p.max_price, p.website_id
FROM catalog_product_index_price p
JOIN catalog_product_entity e ON e.entity_id = p.entity_id
WHERE e.sku = '24-MB01'
  AND p.website_id = 1
  AND p.customer_group_id = 0;
```

The family also contains three physical variants per index:
`<index>`, `<index>_replica` (for read/write splitting, only in async
indexer mode), and `<index>_tmp` (staging area during reindex). A reindex in
progress leaves `_tmp` populated. **Verified**: 1 325 distinct indexes exist
across the 465 tables.

---

## 11. Inventory Tables (MSI)

Magento 2.3+ uses **MSI** (Multi Source Inventory). Stock lives in
`inventory_source_item`, not in `cataloginventory_stock_item`.

**Verified** — `inventory_source` schema:

```
source_code     varchar(255)   PRIMARY KEY   ← natural key, not a numeric id
name            varchar(255)
enabled         smallint unsigned
latitude / longitude / country_id / region_id / city / street / postcode
is_pickup_location_active  tinyint(1)
```

**Verified** — `inventory_source_item`:

```
source_item_id  int unsigned  PRIMARY KEY auto_increment
source_code     varchar(255)  → inventory_source.source_code   (MUL)
sku             varchar(64)   (MUL)
quantity        decimal(12,4)
status          smallint unsigned
```

```sql
-- Verified
SELECT s.source_code, s.name, i.sku, i.quantity, i.status
FROM inventory_source_item i
JOIN inventory_source s ON s.source_code = i.source_code
ORDER BY s.source_code, i.sku
LIMIT 10;
```

```
+-------------+----------------+---------+----------+--------+
| source_code | name           | sku     | quantity | status |
+-------------+----------------+---------+----------+--------+
| default     | Default Source | 24-MB01 | 100.0000 |      1 |
| default     | Default Source | 24-MB02 | 100.0000 |      1 |
+-------------+----------------+---------+----------+--------+
```

> **The join key is `source_code`, not an ID.** `inventory_source` has no
> `source_id` column. This is a deliberate Magento design choice (a source code
> is stable across environments); the trap is assuming the usual `_id` pattern.

`status`: `0` = out of stock, `1` = in stock, `2` = below stock threshold.

---

## 12. Store, Website and Configuration Tables

### 12.1 The three-level hierarchy

**Verified** — `store_website`:

| website_id | code | name | is_default |
|---|---|---|---|
| 0 | `admin` | Admin | 0 |
| 1 | `base` | Main Website | 1 |

**Verified** — `store` (store views):

| store_id | code | website_id | group_id | name | is_active |
|---|---|---|---|---|---|
| 0 | `admin` | 0 | 0 | Admin | 1 |
| 1 | `default` | 1 | 1 | Default Store View | 1 |
| 6 | `french` | 1 | 1 | French | 1 |
| 7 | `german` | 1 | 1 | German | 1 |
| 22 | `default_fr` | 1 | 1 | Default French | 0 |

Note `store_id = 0` is reserved for the admin scope and `is_active = 0` means a
disabled store view (the row survives, the frontend does not serve it).
`store_group` holds the store-group (store) level with columns `store_group_id`,
`website_id`, `name`, `is_active`, `default_store_id`.

### 12.2 `core_config_data`

All admin configuration lives in one table, scoped:

| Column | Meaning |
|---|---|
| `config_id` | PK |
| `scope` | `default`, `websites`, or `stores` |
| `scope_id` | `0` for default; `website_id` for `websites`; `store_id` for `stores` |
| `path` | Slash-delimited config path |
| `value` | The value |

**Verified**:

```
scope      scope_id  path                        rows
default    0         web/unsecure/base_url      1
stores     6         web/unsecure/base_url      1
stores     7         web/unsecure/base_url      1
stores     8         web/unsecure/base_url      1
stores     22        web/unsecure/base_url      1
```

Note the scope type string is **`stores`**, not `store_view`, and scope ids
6, 7, 8, 22 are **store ids**, matching the `store` table above. Reading config
in SQL means implementing the same fallback (`stores` → `websites` → `default`)
that `Magento\Framework\App\Config\ScopeConfigInterface::getValue()` applies
internally. Read config in PHP with `ScopeConfigInterface` instead — see
[queries guide, section 5.5](magento-database-queries.md#55-config-values).

---

## 13. Reading the Schema

### 13.1 The four commands that answer most questions

```sql
-- 1. What columns does this table have?
DESCRIBE sales_order;

-- 2. What exactly is the DDL (including types, defaults, charset)?
SHOW CREATE TABLE catalog_product_entity_varchar;

-- 3. What indexes exist, and in which column order?
SHOW INDEX FROM sales_order;

-- 4. Which tables exist, filtered?
SHOW TABLES LIKE 'sales_invoice%';
```

### 13.2 Information schema — the power queries

```sql
-- All tables of a family, with row estimate
SELECT table_name, table_rows, table_comment
FROM information_schema.tables
WHERE table_schema = DATABASE() AND table_name LIKE 'catalog_product_entity%'
ORDER BY table_name;

-- Find a column by name across every table (indispensable)
SELECT table_name, column_name, column_type, column_key
FROM information_schema.columns
WHERE table_schema = DATABASE() AND column_name = 'order_id'
ORDER BY table_name;

-- Foreign keys on one table
SELECT CONSTRAINT_NAME, COLUMN_NAME, REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'sales_order_item'
  AND REFERENCED_TABLE_NAME IS NOT NULL;

-- Count rows per table (exact, not estimated)
SELECT table_name, table_rows FROM information_schema.tables
WHERE table_schema = DATABASE() ORDER BY table_rows DESC LIMIT 10;
```

### 13.3 Navigating the PHP side

| Question | Where to look |
|---|---|
| Which table does this model use? | `Model/ResourceModel/<Model>.php` → `$this->_init('table_name', 'id_column')` |
| Where is the schema declared? | `vendor/magento/module-*/etc/db_schema.xml` or `app/code/<Vendor>/<Module>/etc/db_schema.xml` |
| How is a name resolved to an id? | `eav_attribute` + `eav_entity_type` |
| Where are indexers declared? | `vendor/magento/module-*/etc/indexer.xml`, state in `indexer_state` |

**Verified** — `indexer_state` currently holds 14 indexers, all `valid`:

```
catalog_product_price, catalog_product_attribute, catalog_product_category,
catalog_category_product, cataloginventory_stock, catalogrule_product,
catalogrule_rule, catalogsearch_fulltext, inventory, customer_grid,
design_config_grid, store_data_exporter,
sales_order_data_exporter, sales_order_status_data_exporter
```

If a query returns data that PHP does not, check `bin/magento indexer:status`
before suspecting the database.

---

## 14. Database Best Practices

### 14.1 Reading

1. **Use `base_*` columns** for any money or quantity aggregation.
2. **Filter `status`/`state` explicitly** — a raw `SELECT * FROM sales_order`
   includes canceled orders and will inflate revenue.
3. **Use the index tables for catalog reads**, not EAV joins.
4. **Add `store_id` to every catalog filter** unless you deliberately want all
   store views.
5. **Run `EXPLAIN` before any query touching `catalog_product_entity_*`** —
   those tables are the largest in the database.

### 14.2 Writing

1. **Never `UPDATE` or `DELETE` a `catalog_*_index*` table.** Reindex instead:
   `bin/magento indexer:reindex catalog_product_price`.
2. **Never `DELETE FROM sales_order` or `sales_order_item`.** Use
   `OrderManagementInterface` or at minimum set `status = 'canceled'`. Deleting
   an order orphans its invoices, credit memos and shipments.
3. **Never hand-edit `sequence_*` tables.**
4. **Do not write to EAV tables by hand** — go through the entity manager, or
   you will bypass cache invalidation and the store fallback.
5. **Always use the `default` resource for DDL** in `db_schema.xml`, except for
   `resource="checkout"` when extending a checkout table (that is legitimate and
   supported).

### 14.3 Schema changes

Schema is declared in `db_schema.xml` (declarative schema), applied with:

```bash
bin/magento setup:db:status          # check if a change is pending
bin/magento setup:db-schema:upgrade   # apply declarative schema changes
bin/magento setup:db-declaration:generate-patch
```

The index and EAV metadata tables (`patch_list`, `flag`) track what has been
applied. **Verified**: `bin/magento setup:db:status` reports
`All modules are up to date.`

See [Queries guide, section 7](magento-database-queries.md#7-creating-a-table)
for a complete `db_schema.xml` example.

---

## 15. Summary

| Concept | Takeaway |
|---|---|
| Two models | Flat for business-critical data, EAV for extensible attributes |
| EAV chain | `eav_entity_type` → `eav_attribute` → `*_entity_*` value table |
| Always scope by entity type | `attribute_code` is unique per entity type, not globally |
| `select` attributes | Store an `option_id`, not a label; join `eav_attribute_option_value` |
| `store_id = 0` | The admin/default EAV value; apply the store fallback in SQL |
| Snapshot columns | `sales_order_item` duplicates SKU/name/price on purpose |
| `base_*` columns | Accounting truth in base currency; use them in reports |
| Foreign keys | 409 exist; cascade on structure, `SET NULL` on identity, none on product/customer links |
| Index tables | Derived data; filter by website_id and customer_group_id |
| MSI join key | `inventory_source_item.source_code`, never a `source_id` |
| `core_config_data` | scope `default` / `websites` / `stores` + `scope_id` |
| Schema changes | `db_schema.xml` + `bin/magento setup:db-schema:upgrade` |

**Continue with** [magento-database-queries.md](magento-database-queries.md) —
how to write and run real queries, with the Collection and Repository APIs.

---

## Official Magento 2 Documentation

| Topic | Link |
|---|---|
| Developer documentation home | [developer.adobe.com/commerce/docs](https://developer.adobe.com/commerce/docs) |
| PHP extensions reference | [developer.adobe.com/commerce/php/](https://developer.adobe.com/commerce/php/) |
| Configure declarative schema | [declarative-schema/configuration](https://developer.adobe.com/commerce/php/development/components/declarative-schema/configuration) |
| Data and schema patches | [declarative-schema/patches](https://developer.adobe.com/commerce/php/development/components/declarative-schema/patches) |
| Website / store / store view scope | [websites-stores-views](https://experienceleague.adobe.com/en/docs/commerce-admin/start/setup/websites-stores-views) |
| Multiple websites or stores | [ms-overview](https://experienceleague.adobe.com/en/docs/commerce-operations/configuration-guide/multi-sites/ms-overview) |
| Database configuration best practices | [database-on-cloud](https://experienceleague.adobe.com/en/docs/commerce-operations/implementation-playbook/best-practices/planning/database-on-cloud) |

---

## Sources

Schema details and all query outputs in this document come from a live
Magento 2.4.8 install on MySQL 8.0.46, with 465 tables, 409 foreign keys and
1 325 indexes. Class references point to `src/vendor/magento/`.

*Last updated: 2026-09-28*
