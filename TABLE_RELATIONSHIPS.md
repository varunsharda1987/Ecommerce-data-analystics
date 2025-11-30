# E-Commerce Analytics - Table Relationships

## Visual Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CORE PRODUCT & INVENTORY                            │
└─────────────────────────────────────────────────────────────────────────────┘

                            ┌──────────────────┐
                            │    PRODUCTS      │
                            │                  │
                            │ PK: sku          │
                            │    product_name  │
                            │    category      │
                            │    brand         │
                            │    unit_cost     │
                            │    current_price │
                            └────────┬─────────┘
                                     │
                   ┌─────────────────┼─────────────────┬──────────────────┐
                   │                 │                 │                  │
                   ▼                 ▼                 ▼                  ▼
         ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐
         │   INVENTORY     │  │ INVENTORY_   │  │  PRICING_    │  │  COMPETITOR_    │
         │                 │  │ TRANSACTIONS │  │  HISTORY     │  │  PRICING        │
         │ PK: inventory_id│  │              │  │              │  │                 │
         │ FK: sku ────────┼──│ FK: sku ─────┼──│ FK: sku ─────┼──│ FK: sku         │
         │    warehouse_id │  │    quantity  │  │    old_price │  │    competitor   │
         │    qty_on_hand  │  │    trans_type│  │    new_price │  │    their_price  │
         └─────────────────┘  └──────────────┘  └──────────────┘  └─────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                       CUSTOMERS & ORDERS FLOW                                │
└─────────────────────────────────────────────────────────────────────────────┘

         ┌──────────────────┐
         │    CUSTOMERS     │
         │                  │
         │ PK: customer_id  │
         │    email         │
         │    total_orders  │
         │    lifetime_val  │
         │    rfm_segment   │
         └────────┬─────────┘
                  │
                  │ 1:N (one customer, many orders)
                  │
                  ▼
         ┌──────────────────┐
         │     ORDERS       │
         │                  │
         │ PK: order_id     │
         │ FK: customer_id ─┼───(links to CUSTOMERS)
         │    order_date    │
         │    total_amount  │
         │    order_status  │
         │    channel       │
         │    utm_campaign  │
         └────────┬─────────┘
                  │
                  │ 1:N (one order, many items)
                  │
                  ▼
         ┌──────────────────┐
         │   ORDER_ITEMS    │
         │                  │
         │ PK: order_item_id│
         │ FK: order_id ────┼───(links to ORDERS)
         │ FK: sku ─────────┼───(links to PRODUCTS)
         │    quantity      │
         │    unit_price    │
         │    line_total    │
         └────────┬─────────┘
                  │
                  │ (can be returned)
                  │
                  ▼
         ┌──────────────────┐
         │   RETURN_ITEMS   │
         │                  │
         │ PK: return_item  │
         │ FK: order_item_id├───(links to ORDER_ITEMS)
         │ FK: return_id    │
         │ FK: sku ─────────┼───(links to PRODUCTS)
         │    reason        │
         │    resaleable    │
         └──────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                          RETURNS PROCESSING                                  │
└─────────────────────────────────────────────────────────────────────────────┘

         ┌──────────────────┐
         │     RETURNS      │
         │                  │
         │ PK: return_id    │
         │ FK: order_id ────┼───(links to ORDERS)
         │ FK: customer_id ─┼───(links to CUSTOMERS)
         │    return_status │
         │    return_type   │
         │    refund_amount │
         └────────┬─────────┘
                  │
                  │ 1:N (one return, many items)
                  │
                  ▼
         ┌──────────────────┐
         │   RETURN_ITEMS   │
         │                  │
         │ PK: return_item  │
         │ FK: return_id ───┼───(links to RETURNS)
         │ FK: order_item_id│
         │ FK: sku          │
         │    qty_returned  │
         │    reason_cat    │
         └──────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                       ADVERTISING CAMPAIGNS                                  │
└─────────────────────────────────────────────────────────────────────────────┘

         ┌──────────────────┐
         │  AD_CAMPAIGNS    │
         │                  │
         │ PK: campaign_id  │
         │    platform      │
         │    campaign_name │
         │    budget        │
         │    status        │
         │ FK: season_id ───┼───(optional link to SEASONS)
         └────────┬─────────┘
                  │
                  │ 1:N (one campaign, daily performance records)
                  │
                  ▼
         ┌──────────────────┐
         │ AD_PERFORMANCE   │
         │                  │
         │ PK: performance  │
         │ FK: campaign_id ─┼───(links to AD_CAMPAIGNS)
         │    date          │
         │    impressions   │
         │    clicks        │
         │    spend         │
         │    revenue       │
         │    roas          │
         └──────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                       SEASONAL PLANNING                                      │
└─────────────────────────────────────────────────────────────────────────────┘

         ┌──────────────────┐
         │     SEASONS      │
         │                  │
         │ PK: season_id    │
         │    season_name   │
         │    start_date    │
         │    end_date      │
         │    total_budget  │
         │    revenue_target│
         └────────┬─────────┘
                  │
                  ├─────────────────────┬──────────────────────┐
                  │                     │                      │
                  │ 1:N                 │ 1:N                  │ 1:N
                  │                     │                      │
                  ▼                     ▼                      ▼
         ┌────────────────┐   ┌─────────────────┐   ┌─────────────────┐
         │ SEASON_BUDGETS │   │SEASON_MILESTONES│   │  AD_CAMPAIGNS   │
         │                │   │                 │   │                 │
         │ PK: budget_id  │   │ PK: milestone   │   │ PK: campaign_id │
         │ FK: season_id ─┼───│ FK: season_id ──┼───│ FK: season_id   │
         │    category    │   │    name         │   │    platform     │
         │    budgeted    │   │    deadline     │   │    budget       │
         │    actual_spend│   │    status       │   └─────────────────┘
         └────────────────┘   └─────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE RELATIONSHIP MAP                                 │
└─────────────────────────────────────────────────────────────────────────────┘

                                PRODUCTS (Master Table)
                                    │
                ┌───────────────────┼───────────────────┬──────────────────┐
                │                   │                   │                  │
            INVENTORY         PRICING_HISTORY    COMPETITOR_PRICING   INVENTORY_TRANS
                                                                            │
                                                                            │
                                                                       (references
                                                                        products)

    CUSTOMERS ─────────> ORDERS ─────────> ORDER_ITEMS ────────> PRODUCTS
        │                   │                    │
        │                   │                    │
        │                   │                    └──────> RETURN_ITEMS
        │                   │                                  │
        │                   │                                  │
        │                   └──────> RETURNS <─────────────────┘
        │                                │
        └────────────────────────────────┘


    SEASONS ────────┬────────> SEASON_BUDGETS
                    │
                    ├────────> SEASON_MILESTONES
                    │
                    └────────> AD_CAMPAIGNS ────────> AD_PERFORMANCE


## Detailed Relationship Breakdown

### 1. PRODUCTS (Center of Product Data)

**Relationships:**
- **1:N → INVENTORY** (One product can be in multiple warehouses)
- **1:N → INVENTORY_TRANSACTIONS** (One product has many stock movements)
- **1:N → ORDER_ITEMS** (One product sold in many orders)
- **1:N → RETURN_ITEMS** (One product can be returned multiple times)
- **1:N → PRICING_HISTORY** (One product has price change history)
- **1:N → COMPETITOR_PRICING** (One product tracked against competitors)

```
PRODUCTS
   ├── INVENTORY (by warehouse location)
   ├── INVENTORY_TRANSACTIONS (stock movements)
   ├── ORDER_ITEMS (sales)
   ├── RETURN_ITEMS (returns)
   ├── PRICING_HISTORY (price changes)
   └── COMPETITOR_PRICING (competitor prices)
```

---

### 2. CUSTOMERS (Center of Customer Data)

**Relationships:**
- **1:N → ORDERS** (One customer can have many orders)
- **1:N → RETURNS** (One customer can have many returns)

```
CUSTOMERS
   ├── ORDERS (purchase history)
   └── RETURNS (return history)
```

**Example:**
- Customer ID: CUST-001
  - Order #1234 (Jan 15, 2024)
  - Order #1456 (Feb 20, 2024)
  - Return #R-789 (for Order #1234)

---

### 3. ORDERS (Center of Transaction Flow)

**Relationships:**
- **N:1 → CUSTOMERS** (Many orders belong to one customer)
- **1:N → ORDER_ITEMS** (One order has many line items)
- **1:N → RETURNS** (One order can have multiple returns)

```
CUSTOMERS
    └── ORDERS
           ├── ORDER_ITEMS (line items in the order)
           └── RETURNS (if items are returned)
```

**Example:**
- Order #1234
  - Customer: CUST-001
  - Line Item 1: T-Shirt (SKU: SHIRT-001) × 2
  - Line Item 2: Jeans (SKU: JEANS-001) × 1
  - Return: R-789 (returned the jeans)

---

### 4. ORDER_ITEMS (Transaction Details)

**Relationships:**
- **N:1 → ORDERS** (Many items belong to one order)
- **N:1 → PRODUCTS** (Many order items reference one product)
- **1:1 or 1:N → RETURN_ITEMS** (An order item can be returned once or partially multiple times)

```
ORDERS
   └── ORDER_ITEMS
          ├── (references) PRODUCTS
          └── (can become) RETURN_ITEMS
```

---

### 5. RETURNS (Return Processing)

**Relationships:**
- **N:1 → ORDERS** (Many returns can be from one order - different items at different times)
- **N:1 → CUSTOMERS** (Many returns belong to one customer)
- **1:N → RETURN_ITEMS** (One return contains many items)

```
RETURNS
   ├── (links to) ORDERS
   ├── (links to) CUSTOMERS
   └── RETURN_ITEMS (items being returned)
```

**Example:**
- Return #R-789
  - From Order: #1234
  - Customer: CUST-001
  - Return Item 1: Jeans (SKU: JEANS-001) × 1
  - Reason: Wrong size

---

### 6. RETURN_ITEMS (Return Details)

**Relationships:**
- **N:1 → RETURNS** (Many items in one return)
- **N:1 → ORDER_ITEMS** (Links back to original order item)
- **N:1 → PRODUCTS** (References the product being returned)

```
RETURN_ITEMS
   ├── (links to) RETURNS
   ├── (links to) ORDER_ITEMS (original purchase)
   └── (links to) PRODUCTS
```

---

### 7. INVENTORY (Stock by Location)

**Relationships:**
- **N:1 → PRODUCTS** (Many warehouse locations stock one product)

```
PRODUCTS
   └── INVENTORY
          ├── Warehouse A: 100 units
          ├── Warehouse B: 50 units
          └── Warehouse C: 75 units
```

**Example:**
- Product: T-Shirt (SKU: SHIRT-001)
  - New York Warehouse: 100 units
  - LA Warehouse: 50 units
  - Chicago Warehouse: 75 units

---

### 8. INVENTORY_TRANSACTIONS (Stock Movements)

**Relationships:**
- **N:1 → PRODUCTS** (Many transactions for one product)

```
PRODUCTS
   └── INVENTORY_TRANSACTIONS
          ├── Purchase (received 100 units)
          ├── Sale (sold 10 units)
          ├── Return from customer (received 2 units)
          └── Transfer (moved 20 units to another warehouse)
```

---

### 9. AD_CAMPAIGNS (Marketing Campaigns)

**Relationships:**
- **N:1 → SEASONS** (Optional: Many campaigns can be part of one season)
- **1:N → AD_PERFORMANCE** (One campaign has daily performance records)

```
AD_CAMPAIGNS
   └── AD_PERFORMANCE (daily metrics)
          ├── Jan 1: $100 spend, $500 revenue
          ├── Jan 2: $120 spend, $600 revenue
          └── Jan 3: $110 spend, $550 revenue
```

---

### 10. AD_PERFORMANCE (Daily Metrics)

**Relationships:**
- **N:1 → AD_CAMPAIGNS** (Many daily records for one campaign)

---

### 11. PRICING_HISTORY (Price Changes)

**Relationships:**
- **N:1 → PRODUCTS** (Many price changes for one product)

```
PRODUCTS
   └── PRICING_HISTORY
          ├── Jan 1: $29.99 → $34.99 (price increase)
          ├── Feb 1: $34.99 → $27.99 (promotion)
          └── Mar 1: $27.99 → $32.99 (back to regular)
```

---

### 12. COMPETITOR_PRICING (Market Intelligence)

**Relationships:**
- **N:1 → PRODUCTS** (Many competitor price checks for one product)

```
PRODUCTS
   └── COMPETITOR_PRICING
          ├── Competitor A: $29.99
          ├── Competitor B: $31.99
          └── Competitor C: $28.99
```

---

### 13. SEASONS (Campaign Planning)

**Relationships:**
- **1:N → SEASON_BUDGETS** (One season has multiple budget categories)
- **1:N → SEASON_MILESTONES** (One season has multiple milestones/tasks)
- **1:N → AD_CAMPAIGNS** (One season can have multiple ad campaigns)

```
SEASONS (Black Friday 2024)
   ├── SEASON_BUDGETS
   │      ├── Inventory: $50,000
   │      ├── Marketing: $30,000
   │      └── Operations: $10,000
   │
   ├── SEASON_MILESTONES
   │      ├── Inventory ordered (deadline: Oct 1)
   │      ├── Ads launched (deadline: Nov 15)
   │      └── Website ready (deadline: Nov 20)
   │
   └── AD_CAMPAIGNS
          ├── Google Ads Campaign
          ├── Facebook Ads Campaign
          └── TikTok Ads Campaign
```

---

### 14. SEASON_BUDGETS (Budget Allocation)

**Relationships:**
- **N:1 → SEASONS** (Many budget categories for one season)

---

### 15. SEASON_MILESTONES (Task Tracking)

**Relationships:**
- **N:1 → SEASONS** (Many milestones for one season)
- **Self-referencing** (Optional: One milestone can depend on another)

---

## Key Relationship Patterns

### Pattern 1: Product-Centric (Star Schema)
```
              INVENTORY
                  │
    PRICING ──── PRODUCTS ──── ORDER_ITEMS
                  │
            RETURN_ITEMS
```
**Purpose:** All product-related data radiates from PRODUCTS table

---

### Pattern 2: Customer Journey (Linear Flow)
```
CUSTOMERS → ORDERS → ORDER_ITEMS → RETURN_ITEMS → RETURNS
```
**Purpose:** Track customer lifecycle from purchase to return

---

### Pattern 3: Season Planning (Hierarchical)
```
SEASONS
   ├── SEASON_BUDGETS
   ├── SEASON_MILESTONES
   └── AD_CAMPAIGNS → AD_PERFORMANCE
```
**Purpose:** Organize seasonal campaigns and budgets

---

## Cardinality Summary

| Relationship | Type | Description |
|--------------|------|-------------|
| PRODUCTS → INVENTORY | 1:N | One product in many locations |
| PRODUCTS → ORDER_ITEMS | 1:N | One product sold many times |
| CUSTOMERS → ORDERS | 1:N | One customer, many orders |
| ORDERS → ORDER_ITEMS | 1:N | One order, many items |
| ORDERS → RETURNS | 1:N | One order, multiple returns possible |
| RETURNS → RETURN_ITEMS | 1:N | One return, many items |
| ORDER_ITEMS → RETURN_ITEMS | 1:1 or 1:N | Item can be returned once or partially |
| AD_CAMPAIGNS → AD_PERFORMANCE | 1:N | One campaign, daily records |
| SEASONS → SEASON_BUDGETS | 1:N | One season, many budget categories |
| SEASONS → AD_CAMPAIGNS | 1:N | One season, many campaigns |

---

## Foreign Key Constraints

### Critical Constraints (RESTRICT - Prevent Deletion)
```sql
-- Cannot delete a product if it has orders
ORDER_ITEMS.sku → PRODUCTS.sku (ON DELETE RESTRICT)

-- Cannot delete a customer if they have orders
ORDERS.customer_id → CUSTOMERS.customer_id (ON DELETE RESTRICT)

-- Cannot delete an order if it has returns
RETURNS.order_id → ORDERS.order_id (ON DELETE RESTRICT)
```

### Cascading Constraints (CASCADE - Delete Related Data)
```sql
-- Delete order items when order is deleted
ORDER_ITEMS.order_id → ORDERS.order_id (ON DELETE CASCADE)

-- Delete return items when return is deleted
RETURN_ITEMS.return_id → RETURNS.return_id (ON DELETE CASCADE)

-- Delete performance data when campaign is deleted
AD_PERFORMANCE.campaign_id → AD_CAMPAIGNS.campaign_id (ON DELETE CASCADE)

-- Delete season budgets when season is deleted
SEASON_BUDGETS.season_id → SEASONS.season_id (ON DELETE CASCADE)
```

---

## Common Join Queries

### Query 1: Get Order with Customer and Items
```sql
SELECT
    o.order_id,
    c.customer_name,
    c.email,
    oi.product_name,
    oi.quantity,
    oi.unit_price
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_id = 'ORD-001';
```

### Query 2: Get Product with Inventory Across Locations
```sql
SELECT
    p.sku,
    p.product_name,
    i.warehouse_name,
    i.quantity_available
FROM products p
JOIN inventory i ON p.sku = i.sku
WHERE p.sku = 'SHIRT-001';
```

### Query 3: Get Return with Original Order Info
```sql
SELECT
    r.return_id,
    r.return_date,
    o.order_id,
    o.order_date,
    c.customer_name,
    ri.product_name,
    ri.return_reason_category
FROM returns r
JOIN orders o ON r.order_id = o.order_id
JOIN customers c ON r.customer_id = c.customer_id
JOIN return_items ri ON r.return_id = ri.return_id
WHERE r.return_id = 'RET-001';
```

### Query 4: Get Season with All Budgets
```sql
SELECT
    s.season_name,
    s.total_budget,
    sb.budget_category,
    sb.budgeted_amount,
    sb.actual_spend
FROM seasons s
JOIN season_budgets sb ON s.season_id = sb.season_id
WHERE s.season_id = 'BF-2024';
```

### Query 5: Get Campaign with Performance
```sql
SELECT
    c.campaign_name,
    c.platform,
    p.date,
    p.spend,
    p.revenue,
    p.roas
FROM ad_campaigns c
JOIN ad_performance p ON c.campaign_id = p.campaign_id
WHERE c.campaign_id = 'CAMP-001'
ORDER BY p.date DESC;
```

---

## Data Flow Examples

### Example 1: Customer Makes a Purchase
```
1. Customer record exists in CUSTOMERS
2. New record created in ORDERS (links to customer_id)
3. New records created in ORDER_ITEMS (links to order_id and sku)
4. INVENTORY_TRANSACTIONS created for each item (type: sale)
5. INVENTORY updated (quantity_on_hand decreased)
6. PRODUCTS updated (total_sales_count increased)
7. CUSTOMERS updated (total_orders, total_lifetime_value increased)
```

### Example 2: Customer Returns an Item
```
1. RETURN record created (links to order_id and customer_id)
2. RETURN_ITEMS created (links to return_id, order_item_id, sku)
3. INVENTORY_TRANSACTIONS created (type: return_from_customer)
4. If resaleable, INVENTORY updated (quantity_on_hand increased)
5. RETURNS updated with refund_amount
6. CUSTOMERS updated (total_returns increased, return_rate recalculated)
```

### Example 3: Price Change
```
1. New record in PRICING_HISTORY (old_price, new_price)
2. PRODUCTS updated (current_price = new_price)
3. All future ORDER_ITEMS will use new price
4. Historical ORDER_ITEMS retain original price
```

### Example 4: Ad Campaign Run
```
1. AD_CAMPAIGNS record created (linked to season_id if applicable)
2. Daily AD_PERFORMANCE records created (linked to campaign_id)
3. When orders come in with utm_campaign:
   - ORDERS.utm_campaign = campaign_name
   - Revenue attributed to campaign
   - AD_PERFORMANCE.revenue updated
```

---

**Document Version**: 1.0
**Last Updated**: 2024-01-15
