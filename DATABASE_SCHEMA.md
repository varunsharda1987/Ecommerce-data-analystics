# E-Commerce Analytics Tool - Database Schema

## Table of Contents
1. [Entity Relationship Overview](#entity-relationship-overview)
2. [Complete Table Definitions](#complete-table-definitions)
3. [Table Relationships](#table-relationships)
4. [Indexes and Performance](#indexes-and-performance)
5. [Sample Queries](#sample-queries)
6. [Data Dictionary](#data-dictionary)

---

## Entity Relationship Overview

### Core Entities and Their Relationships

```
                                    PRODUCTS
                                        |
                    +-------------------+-------------------+
                    |                   |                   |
                INVENTORY          ORDER_ITEMS         PRICING_HISTORY
                    |                   |                   |
            INVENTORY_TRANS        ORDERS            COMPETITOR_PRICING
                                        |
                    +-------------------+-------------------+
                    |                   |                   |
                CUSTOMERS           RETURNS          AD_CAMPAIGNS
                                        |                   |
                                  RETURN_ITEMS        AD_PERFORMANCE

                                    SEASONS
                                        |
                                  SEASON_BUDGETS
```

---

## Complete Table Definitions

### 1. Products Table
**Purpose**: Master product catalog with all product information

```sql
CREATE TABLE products (
    -- Primary Key
    sku VARCHAR(50) PRIMARY KEY,

    -- Product Information
    product_name VARCHAR(255) NOT NULL,
    description TEXT,
    short_description VARCHAR(500),

    -- Categorization
    category VARCHAR(100),
    sub_category VARCHAR(100),
    brand VARCHAR(100),
    product_type VARCHAR(50), -- physical, digital, service

    -- Supplier Information
    supplier_id VARCHAR(50),
    supplier_name VARCHAR(255),

    -- Pricing Information
    unit_cost DECIMAL(10,2) NOT NULL,
    current_price DECIMAL(10,2) NOT NULL,
    list_price DECIMAL(10,2), -- MSRP/original price
    minimum_price DECIMAL(10,2), -- price floor
    maximum_price DECIMAL(10,2), -- price ceiling

    -- Inventory Management
    reorder_point INTEGER DEFAULT 0,
    safety_stock_level INTEGER DEFAULT 0,
    economic_order_quantity INTEGER,
    lead_time_days INTEGER DEFAULT 0,

    -- Product Attributes
    weight_kg DECIMAL(8,2),
    dimensions_cm VARCHAR(50), -- LxWxH
    color VARCHAR(50),
    size VARCHAR(50),
    material VARCHAR(100),

    -- Product Lifecycle
    launch_date DATE,
    discontinue_date DATE,
    status VARCHAR(20) DEFAULT 'active', -- active, discontinued, seasonal, out_of_stock

    -- SEO and Marketing
    product_url VARCHAR(500),
    image_url VARCHAR(500),
    meta_title VARCHAR(255),
    meta_description TEXT,

    -- Analytics
    total_sales_count INTEGER DEFAULT 0,
    total_revenue DECIMAL(12,2) DEFAULT 0,
    average_rating DECIMAL(3,2),
    review_count INTEGER DEFAULT 0,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Indexes for performance
    INDEX idx_category (category),
    INDEX idx_brand (brand),
    INDEX idx_status (status),
    INDEX idx_supplier (supplier_id)
);
```

---

### 2. Inventory Table
**Purpose**: Track current inventory levels by location

```sql
CREATE TABLE inventory (
    -- Primary Key
    inventory_id SERIAL PRIMARY KEY,

    -- Foreign Keys
    sku VARCHAR(50) NOT NULL,

    -- Location Information
    warehouse_id VARCHAR(50) NOT NULL,
    warehouse_name VARCHAR(255),
    warehouse_location VARCHAR(255), -- city, state, country

    -- Inventory Quantities
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0, -- allocated to orders
    quantity_available INTEGER NOT NULL DEFAULT 0, -- on_hand - reserved
    quantity_in_transit INTEGER DEFAULT 0, -- incoming stock
    quantity_damaged INTEGER DEFAULT 0,
    quantity_on_hold INTEGER DEFAULT 0, -- quality check, etc.

    -- Valuation
    total_value DECIMAL(12,2), -- quantity_on_hand * unit_cost
    average_cost DECIMAL(10,2), -- weighted average cost

    -- Location Details
    bin_location VARCHAR(50), -- specific warehouse location
    zone VARCHAR(20), -- warehouse zone

    -- Timestamps
    last_counted_date DATE, -- last physical count
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (sku) REFERENCES products(sku) ON DELETE CASCADE,
    UNIQUE KEY unique_sku_warehouse (sku, warehouse_id),

    -- Indexes
    INDEX idx_sku (sku),
    INDEX idx_warehouse (warehouse_id),
    INDEX idx_available (quantity_available)
);
```

---

### 3. Inventory Transactions Table
**Purpose**: Track all inventory movements

```sql
CREATE TABLE inventory_transactions (
    -- Primary Key
    transaction_id SERIAL PRIMARY KEY,

    -- Foreign Keys
    sku VARCHAR(50) NOT NULL,
    warehouse_id VARCHAR(50) NOT NULL,

    -- Transaction Details
    transaction_type VARCHAR(30) NOT NULL,
    -- Types: purchase, sale, return_from_customer, return_to_supplier,
    --        transfer_in, transfer_out, adjustment_increase, adjustment_decrease,
    --        damage, theft, loss

    quantity INTEGER NOT NULL,
    unit_cost DECIMAL(10,2),
    total_cost DECIMAL(12,2),

    -- Reference Information
    reference_type VARCHAR(30), -- order, purchase_order, transfer, adjustment
    reference_id VARCHAR(100), -- order_id, po_id, transfer_id, etc.

    -- Transfer Details (if applicable)
    from_warehouse_id VARCHAR(50),
    to_warehouse_id VARCHAR(50),

    -- Additional Information
    reason VARCHAR(100), -- reason for adjustment, damage, etc.
    notes TEXT,
    performed_by VARCHAR(100), -- user who performed the transaction

    -- Timestamps
    transaction_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (sku) REFERENCES products(sku) ON DELETE CASCADE,

    -- Indexes
    INDEX idx_sku (sku),
    INDEX idx_transaction_type (transaction_type),
    INDEX idx_transaction_date (transaction_date),
    INDEX idx_reference (reference_type, reference_id),
    INDEX idx_warehouse (warehouse_id)
);
```

---

### 4. Customers Table
**Purpose**: Store customer information and metrics

```sql
CREATE TABLE customers (
    -- Primary Key
    customer_id VARCHAR(50) PRIMARY KEY,

    -- Personal Information
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),

    -- Account Information
    registration_date DATE NOT NULL,
    account_status VARCHAR(20) DEFAULT 'active', -- active, inactive, suspended, deleted
    customer_type VARCHAR(20) DEFAULT 'B2C', -- B2C, B2B, wholesale

    -- Demographic Information
    date_of_birth DATE,
    gender VARCHAR(20),

    -- Location
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(100),

    -- Customer Segmentation
    customer_segment VARCHAR(50), -- VIP, loyal, regular, at_risk, lost, etc.
    acquisition_channel VARCHAR(50), -- organic, paid_search, social, referral, etc.
    acquisition_campaign VARCHAR(100),

    -- RFM Scores (for segmentation)
    rfm_recency_score INTEGER, -- 1-5
    rfm_frequency_score INTEGER, -- 1-5
    rfm_monetary_score INTEGER, -- 1-5
    rfm_segment VARCHAR(50), -- champions, loyal, at_risk, etc.

    -- Customer Metrics
    total_orders INTEGER DEFAULT 0,
    total_lifetime_value DECIMAL(12,2) DEFAULT 0,
    total_items_purchased INTEGER DEFAULT 0,
    average_order_value DECIMAL(10,2) DEFAULT 0,
    total_returns INTEGER DEFAULT 0,
    return_rate DECIMAL(5,2) DEFAULT 0, -- percentage

    -- Purchase Behavior
    first_purchase_date DATE,
    last_purchase_date DATE,
    days_since_last_purchase INTEGER,
    purchase_frequency DECIMAL(8,2), -- average days between purchases

    -- Marketing Preferences
    email_subscribed BOOLEAN DEFAULT TRUE,
    sms_subscribed BOOLEAN DEFAULT FALSE,
    marketing_consent BOOLEAN DEFAULT FALSE,

    -- Loyalty Program
    loyalty_points INTEGER DEFAULT 0,
    loyalty_tier VARCHAR(50), -- bronze, silver, gold, platinum

    -- Risk Indicators
    fraud_risk_score DECIMAL(5,2), -- 0-100
    high_return_risk BOOLEAN DEFAULT FALSE,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP,

    -- Indexes
    INDEX idx_email (email),
    INDEX idx_customer_segment (customer_segment),
    INDEX idx_rfm_segment (rfm_segment),
    INDEX idx_last_purchase (last_purchase_date),
    INDEX idx_registration_date (registration_date)
);
```

---

### 5. Orders Table
**Purpose**: Store order header information

```sql
CREATE TABLE orders (
    -- Primary Key
    order_id VARCHAR(50) PRIMARY KEY,

    -- Foreign Keys
    customer_id VARCHAR(50) NOT NULL,

    -- Order Timing
    order_date TIMESTAMP NOT NULL,
    order_time TIME,
    completed_date TIMESTAMP,
    shipped_date TIMESTAMP,
    delivered_date TIMESTAMP,

    -- Order Status
    order_status VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- Status values: pending, confirmed, processing, shipped, delivered,
    --                completed, cancelled, refunded, partially_refunded

    fulfillment_status VARCHAR(30), -- unfulfilled, partially_fulfilled, fulfilled
    payment_status VARCHAR(30), -- pending, paid, failed, refunded, partially_refunded

    -- Financial Information
    subtotal DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    discount_code VARCHAR(50),
    discount_percentage DECIMAL(5,2),

    tax_amount DECIMAL(10,2) DEFAULT 0,
    tax_rate DECIMAL(5,2),

    shipping_amount DECIMAL(10,2) DEFAULT 0,

    total_amount DECIMAL(10,2) NOT NULL,

    -- Cost Information (for margin calculation)
    total_cost_of_goods DECIMAL(10,2),
    gross_profit DECIMAL(10,2),
    gross_margin_percent DECIMAL(5,2),

    -- Payment Information
    payment_method VARCHAR(50), -- credit_card, paypal, apple_pay, etc.
    payment_processor VARCHAR(50),
    transaction_id VARCHAR(100),

    -- Shipping Information
    shipping_method VARCHAR(50), -- standard, express, overnight
    shipping_carrier VARCHAR(50), -- UPS, FedEx, USPS, DHL
    tracking_number VARCHAR(100),

    shipping_address_line1 VARCHAR(255),
    shipping_address_line2 VARCHAR(255),
    shipping_city VARCHAR(100),
    shipping_state VARCHAR(100),
    shipping_postal_code VARCHAR(20),
    shipping_country VARCHAR(100),

    -- Billing Information
    billing_address_line1 VARCHAR(255),
    billing_address_line2 VARCHAR(255),
    billing_city VARCHAR(100),
    billing_state VARCHAR(100),
    billing_postal_code VARCHAR(20),
    billing_country VARCHAR(100),

    -- Channel and Source
    channel VARCHAR(50), -- web, mobile_app, marketplace, retail, phone
    platform VARCHAR(50), -- shopify, amazon, ebay, etc.
    device_type VARCHAR(20), -- desktop, mobile, tablet

    -- Attribution
    traffic_source VARCHAR(50), -- organic, paid_search, social, email, direct
    utm_source VARCHAR(100),
    utm_medium VARCHAR(100),
    utm_campaign VARCHAR(100),
    utm_term VARCHAR(100),
    utm_content VARCHAR(100),

    referring_url TEXT,
    landing_page TEXT,

    -- Order Characteristics
    is_first_order BOOLEAN DEFAULT FALSE,
    items_count INTEGER,
    unique_items_count INTEGER,

    -- Customer Service
    notes TEXT,
    customer_notes TEXT,
    gift_message TEXT,
    is_gift BOOLEAN DEFAULT FALSE,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT,

    -- Indexes
    INDEX idx_customer (customer_id),
    INDEX idx_order_date (order_date),
    INDEX idx_order_status (order_status),
    INDEX idx_channel (channel),
    INDEX idx_traffic_source (traffic_source),
    INDEX idx_utm_campaign (utm_campaign),
    INDEX idx_payment_status (payment_status)
);
```

---

### 6. Order Items Table
**Purpose**: Store individual items within each order

```sql
CREATE TABLE order_items (
    -- Primary Key
    order_item_id SERIAL PRIMARY KEY,

    -- Foreign Keys
    order_id VARCHAR(50) NOT NULL,
    sku VARCHAR(50) NOT NULL,

    -- Product Information (snapshot at time of order)
    product_name VARCHAR(255) NOT NULL,
    category VARCHAR(100),
    brand VARCHAR(100),

    -- Quantity and Pricing
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    list_price DECIMAL(10,2), -- original price before discount

    -- Discounts
    discount_amount DECIMAL(10,2) DEFAULT 0,
    discount_percentage DECIMAL(5,2),

    -- Totals
    line_total DECIMAL(10,2) NOT NULL, -- (unit_price * quantity) - discount

    -- Cost Information
    unit_cost DECIMAL(10,2),
    total_cost DECIMAL(10,2), -- unit_cost * quantity
    gross_profit DECIMAL(10,2), -- line_total - total_cost
    gross_margin_percent DECIMAL(5,2),

    -- Fulfillment
    fulfillment_status VARCHAR(30), -- pending, fulfilled, cancelled
    warehouse_id VARCHAR(50),

    -- Product Attributes (at time of purchase)
    variant_options JSON, -- {"size": "L", "color": "Blue"}

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (sku) REFERENCES products(sku) ON DELETE RESTRICT,

    -- Indexes
    INDEX idx_order (order_id),
    INDEX idx_sku (sku),
    INDEX idx_product_name (product_name),
    INDEX idx_category (category)
);
```

---

### 7. Returns Table
**Purpose**: Store return/refund header information

```sql
CREATE TABLE returns (
    -- Primary Key
    return_id VARCHAR(50) PRIMARY KEY,

    -- Foreign Keys
    order_id VARCHAR(50) NOT NULL,
    customer_id VARCHAR(50) NOT NULL,

    -- Return Timing
    return_request_date TIMESTAMP NOT NULL,
    return_received_date TIMESTAMP,
    return_processed_date TIMESTAMP,
    refund_processed_date TIMESTAMP,

    -- Return Status
    return_status VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- Status: pending, approved, rejected, received, inspected,
    --         refunded, exchanged, completed

    -- Return Type
    return_type VARCHAR(30) NOT NULL, -- refund, exchange, store_credit

    -- Financial Information
    total_return_value DECIMAL(10,2), -- original purchase value
    total_refund_amount DECIMAL(10,2), -- actual refund (may differ due to fees)
    restocking_fee DECIMAL(10,2) DEFAULT 0,
    return_shipping_cost DECIMAL(10,2) DEFAULT 0,
    processing_cost DECIMAL(10,2) DEFAULT 0,

    -- Exchange Information (if applicable)
    exchange_order_id VARCHAR(50),
    additional_payment DECIMAL(10,2), -- if exchange item costs more

    -- Return Method
    return_shipping_carrier VARCHAR(50),
    return_tracking_number VARCHAR(100),
    return_label_cost DECIMAL(10,2),

    -- Approval Information
    approved_by VARCHAR(100),
    approval_date TIMESTAMP,
    rejection_reason TEXT,

    -- Total Impact
    net_loss DECIMAL(10,2), -- total financial impact of return
    items_restocked_value DECIMAL(10,2),
    items_disposed_value DECIMAL(10,2),

    -- Notes
    customer_notes TEXT,
    internal_notes TEXT,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE RESTRICT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT,

    -- Indexes
    INDEX idx_order (order_id),
    INDEX idx_customer (customer_id),
    INDEX idx_return_date (return_request_date),
    INDEX idx_status (return_status),
    INDEX idx_return_type (return_type)
);
```

---

### 8. Return Items Table
**Purpose**: Store individual items being returned

```sql
CREATE TABLE return_items (
    -- Primary Key
    return_item_id SERIAL PRIMARY KEY,

    -- Foreign Keys
    return_id VARCHAR(50) NOT NULL,
    order_item_id INTEGER NOT NULL,
    sku VARCHAR(50) NOT NULL,

    -- Product Information
    product_name VARCHAR(255),

    -- Return Quantity
    quantity_returned INTEGER NOT NULL,
    original_quantity INTEGER, -- from the order

    -- Return Reason
    return_reason_category VARCHAR(50) NOT NULL,
    -- Categories: defective, wrong_item, size_fit, quality,
    --            changed_mind, late_delivery, damaged, other

    return_reason_subcategory VARCHAR(100),
    return_reason_detail TEXT, -- free text from customer

    -- Item Condition
    condition_received VARCHAR(30),
    -- Conditions: new_sealed, new_opened, used_good, used_fair, damaged, defective

    resaleable BOOLEAN DEFAULT FALSE,
    requires_inspection BOOLEAN DEFAULT TRUE,

    -- Inspection Results
    inspected_by VARCHAR(100),
    inspection_date TIMESTAMP,
    inspection_notes TEXT,
    quality_issue_confirmed BOOLEAN,

    -- Financial
    unit_price DECIMAL(10,2), -- price paid by customer
    refund_amount DECIMAL(10,2), -- actual refund for this item
    restocked_value DECIMAL(10,2), -- value if restocked
    disposal_cost DECIMAL(10,2), -- cost to dispose if not resaleable

    -- Restocking
    restocked BOOLEAN DEFAULT FALSE,
    restocked_date TIMESTAMP,
    restocked_warehouse_id VARCHAR(50),

    -- Disposition
    disposition VARCHAR(30),
    -- Dispositions: restocked, liquidated, donated, destroyed, returned_to_supplier

    -- Analytics
    days_from_purchase INTEGER, -- days between order and return

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (return_id) REFERENCES returns(return_id) ON DELETE CASCADE,
    FOREIGN KEY (order_item_id) REFERENCES order_items(order_item_id) ON DELETE RESTRICT,
    FOREIGN KEY (sku) REFERENCES products(sku) ON DELETE RESTRICT,

    -- Indexes
    INDEX idx_return (return_id),
    INDEX idx_sku (sku),
    INDEX idx_reason_category (return_reason_category),
    INDEX idx_condition (condition_received),
    INDEX idx_resaleable (resaleable)
);
```

---

### 9. Ad Campaigns Table
**Purpose**: Store advertising campaign information

```sql
CREATE TABLE ad_campaigns (
    -- Primary Key
    campaign_id VARCHAR(100) PRIMARY KEY,

    -- Platform Information
    platform VARCHAR(50) NOT NULL,
    -- Platforms: google_ads, facebook, instagram, tiktok, amazon, pinterest, linkedin

    platform_campaign_id VARCHAR(100), -- ID in the ad platform

    -- Campaign Details
    campaign_name VARCHAR(255) NOT NULL,
    campaign_type VARCHAR(50), -- search, display, shopping, video, social

    -- Campaign Status
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Status: active, paused, ended, draft, archived

    -- Budget Information
    budget_type VARCHAR(30), -- daily, lifetime, monthly
    budget_amount DECIMAL(10,2),
    currency VARCHAR(3) DEFAULT 'USD',

    -- Timing
    start_date DATE NOT NULL,
    end_date DATE,

    -- Campaign Objective
    objective VARCHAR(50),
    -- Objectives: awareness, traffic, engagement, leads, conversions, sales, app_installs

    -- Targeting (stored as JSON for flexibility)
    targeting_settings JSON,
    -- Example: {"age_min": 18, "age_max": 65, "genders": ["male", "female"],
    --           "locations": ["US", "CA"], "interests": ["sports", "fitness"]}

    -- Bidding
    bidding_strategy VARCHAR(50), -- manual_cpc, auto_cpc, target_cpa, target_roas
    target_cpa DECIMAL(10,2),
    target_roas DECIMAL(5,2),

    -- Associated Season (if applicable)
    season_id VARCHAR(50),

    -- Campaign Manager
    managed_by VARCHAR(100),

    -- Notes
    notes TEXT,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Indexes
    INDEX idx_platform (platform),
    INDEX idx_status (status),
    INDEX idx_start_date (start_date),
    INDEX idx_season (season_id)
);
```

---

### 10. Ad Performance Table
**Purpose**: Store daily advertising performance metrics

```sql
CREATE TABLE ad_performance (
    -- Primary Key
    performance_id SERIAL PRIMARY KEY,

    -- Foreign Keys
    campaign_id VARCHAR(100) NOT NULL,

    -- Date and Dimensions
    date DATE NOT NULL,

    -- Ad Set/Group Level (optional, for more granular tracking)
    ad_set_id VARCHAR(100),
    ad_set_name VARCHAR(255),
    ad_id VARCHAR(100),
    ad_name VARCHAR(255),

    -- Keyword Level (for search ads)
    keyword VARCHAR(255),
    match_type VARCHAR(20), -- exact, phrase, broad

    -- Dimensions
    device_type VARCHAR(20), -- desktop, mobile, tablet
    placement VARCHAR(100), -- facebook_feed, instagram_stories, search, display
    age_range VARCHAR(20),
    gender VARCHAR(20),
    location VARCHAR(100), -- country or city

    -- Impression and Click Metrics
    impressions INTEGER DEFAULT 0,
    clicks INTEGER DEFAULT 0,
    ctr DECIMAL(5,2), -- click-through rate %

    -- Cost Metrics
    spend DECIMAL(10,2) DEFAULT 0,
    cpc DECIMAL(10,2), -- cost per click
    cpm DECIMAL(10,2), -- cost per thousand impressions

    -- Conversion Metrics
    conversions INTEGER DEFAULT 0,
    conversion_rate DECIMAL(5,2),
    cost_per_conversion DECIMAL(10,2),

    -- Revenue Metrics
    revenue DECIMAL(10,2) DEFAULT 0,
    roas DECIMAL(8,2), -- return on ad spend (revenue / spend)

    -- Engagement Metrics (social platforms)
    engagements INTEGER DEFAULT 0, -- likes, comments, shares
    engagement_rate DECIMAL(5,2),
    video_views INTEGER DEFAULT 0,
    video_view_rate DECIMAL(5,2),

    -- E-commerce Specific Metrics
    add_to_cart INTEGER DEFAULT 0,
    add_to_cart_rate DECIMAL(5,2),
    purchases INTEGER DEFAULT 0,
    purchase_rate DECIMAL(5,2),

    -- Assisted Conversions
    view_through_conversions INTEGER DEFAULT 0,
    assisted_conversions INTEGER DEFAULT 0,

    -- Quality Metrics (Google Ads)
    quality_score DECIMAL(3,1), -- 1-10
    impression_share DECIMAL(5,2), -- percentage

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (campaign_id) REFERENCES ad_campaigns(campaign_id) ON DELETE CASCADE,
    UNIQUE KEY unique_performance (campaign_id, date, ad_id, keyword, device_type, location),

    -- Indexes
    INDEX idx_campaign (campaign_id),
    INDEX idx_date (date),
    INDEX idx_device (device_type),
    INDEX idx_keyword (keyword)
);
```

---

### 11. Pricing History Table
**Purpose**: Track all price changes over time

```sql
CREATE TABLE pricing_history (
    -- Primary Key
    pricing_id SERIAL PRIMARY KEY,

    -- Foreign Keys
    sku VARCHAR(50) NOT NULL,

    -- Price Change
    old_price DECIMAL(10,2),
    new_price DECIMAL(10,2) NOT NULL,
    price_change_amount DECIMAL(10,2), -- new - old
    price_change_percentage DECIMAL(5,2), -- (new - old) / old * 100

    -- Change Reason
    change_reason VARCHAR(100),
    -- Reasons: cost_increase, cost_decrease, competitive_pricing, promotion,
    --          clearance, dynamic_pricing, manual_adjustment, seasonal

    change_type VARCHAR(30), -- increase, decrease

    -- Promotion Details (if applicable)
    is_promotion BOOLEAN DEFAULT FALSE,
    promotion_name VARCHAR(255),
    promotion_start_date DATE,
    promotion_end_date DATE,

    -- Effective Date
    effective_date TIMESTAMP NOT NULL,
    scheduled_date TIMESTAMP, -- if scheduled for future

    -- Approval
    created_by VARCHAR(100) NOT NULL,
    approved_by VARCHAR(100),
    approval_status VARCHAR(20), -- pending, approved, rejected, auto_approved

    -- Impact Tracking
    expected_revenue_impact DECIMAL(12,2),
    expected_volume_impact INTEGER,
    actual_revenue_impact DECIMAL(12,2), -- calculated after some time
    actual_volume_impact INTEGER,

    -- Notes
    notes TEXT,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (sku) REFERENCES products(sku) ON DELETE CASCADE,

    -- Indexes
    INDEX idx_sku (sku),
    INDEX idx_effective_date (effective_date),
    INDEX idx_change_reason (change_reason)
);
```

---

### 12. Competitor Pricing Table
**Purpose**: Track competitor prices for price comparison

```sql
CREATE TABLE competitor_pricing (
    -- Primary Key
    competitor_price_id SERIAL PRIMARY KEY,

    -- Competitor Information
    competitor_id VARCHAR(50) NOT NULL,
    competitor_name VARCHAR(255) NOT NULL,

    -- Product Matching
    sku VARCHAR(50), -- our SKU
    competitor_product_id VARCHAR(100),
    competitor_product_name VARCHAR(255),
    competitor_sku VARCHAR(100),

    -- Pricing Information
    competitor_price DECIMAL(10,2) NOT NULL,
    competitor_list_price DECIMAL(10,2), -- if on sale
    discount_amount DECIMAL(10,2),
    discount_percentage DECIMAL(5,2),

    -- Availability
    in_stock BOOLEAN DEFAULT TRUE,
    stock_level VARCHAR(30), -- in_stock, low_stock, out_of_stock, unknown

    -- Price Check Details
    price_check_date DATE NOT NULL,
    price_check_time TIME,

    -- Source
    source_type VARCHAR(30), -- manual, web_scraping, api, third_party_tool
    product_url TEXT,

    -- Comparison
    our_price DECIMAL(10,2), -- our price at time of check
    price_difference DECIMAL(10,2), -- our_price - competitor_price
    price_difference_percentage DECIMAL(5,2),
    price_position VARCHAR(30), -- lower, same, higher

    -- Shipping Information
    competitor_shipping_cost DECIMAL(10,2),
    competitor_free_shipping BOOLEAN,

    -- Rating and Reviews
    competitor_rating DECIMAL(3,2),
    competitor_review_count INTEGER,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (sku) REFERENCES products(sku) ON DELETE CASCADE,

    -- Indexes
    INDEX idx_sku (sku),
    INDEX idx_competitor (competitor_id),
    INDEX idx_check_date (price_check_date),
    INDEX idx_price_position (price_position)
);
```

---

### 13. Seasons Table
**Purpose**: Store seasonal events and campaigns

```sql
CREATE TABLE seasons (
    -- Primary Key
    season_id VARCHAR(50) PRIMARY KEY,

    -- Season Information
    season_name VARCHAR(255) NOT NULL,
    season_type VARCHAR(50) NOT NULL,
    -- Types: holiday, sale_event, product_launch, clearance,
    --        promotional_event, quarterly_event, custom

    -- Timing
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    peak_start_date DATE, -- when is peak expected to start
    peak_end_date DATE,

    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'planning',
    -- Status: planning, approved, active, completed, cancelled

    -- Financial Targets
    total_budget DECIMAL(12,2) NOT NULL,
    revenue_target DECIMAL(12,2) NOT NULL,
    profit_target DECIMAL(12,2),
    target_roi_percentage DECIMAL(5,2),

    -- Actual Performance
    actual_revenue DECIMAL(12,2) DEFAULT 0,
    actual_profit DECIMAL(12,2) DEFAULT 0,
    actual_spend DECIMAL(12,2) DEFAULT 0,
    actual_roi_percentage DECIMAL(5,2),

    -- Volume Targets
    target_orders INTEGER,
    target_units_sold INTEGER,
    target_new_customers INTEGER,

    -- Actual Volume
    actual_orders INTEGER DEFAULT 0,
    actual_units_sold INTEGER DEFAULT 0,
    actual_new_customers INTEGER DEFAULT 0,

    -- Season Manager
    managed_by VARCHAR(100),
    team_members JSON, -- ["user1", "user2", "user3"]

    -- Year (for comparisons)
    year INTEGER NOT NULL,

    -- Related Seasons
    previous_season_id VARCHAR(50), -- same season from previous year

    -- Notes and Learnings
    planning_notes TEXT,
    execution_notes TEXT,
    lessons_learned TEXT,
    recommendations TEXT,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Indexes
    INDEX idx_year (year),
    INDEX idx_status (status),
    INDEX idx_start_date (start_date),
    INDEX idx_season_type (season_type)
);
```

---

### 14. Season Budgets Table
**Purpose**: Track budget allocation and spending by category for each season

```sql
CREATE TABLE season_budgets (
    -- Primary Key
    budget_id SERIAL PRIMARY KEY,

    -- Foreign Keys
    season_id VARCHAR(50) NOT NULL,

    -- Budget Category
    budget_category VARCHAR(50) NOT NULL,
    -- Categories: inventory, marketing, operations, logistics, technology, contingency

    sub_category VARCHAR(100),
    -- Marketing: google_ads, facebook_ads, influencer, email, content
    -- Operations: staffing, warehouse, customer_service
    -- Logistics: shipping, packaging, returns

    -- Budget Allocation
    budgeted_amount DECIMAL(10,2) NOT NULL,
    budget_percentage DECIMAL(5,2), -- percentage of total season budget

    -- Spending
    actual_spend DECIMAL(10,2) DEFAULT 0,
    committed_spend DECIMAL(10,2) DEFAULT 0, -- committed but not yet spent
    remaining_budget DECIMAL(10,2),

    -- Variance
    budget_variance DECIMAL(10,2), -- actual - budget
    budget_variance_percentage DECIMAL(5,2),

    -- Pacing
    expected_spend DECIMAL(10,2), -- expected spend by now based on timeline
    pacing_status VARCHAR(30), -- on_track, ahead, behind, overspent

    -- Performance
    revenue_generated DECIMAL(12,2), -- revenue attributed to this budget category
    roi DECIMAL(8,2), -- revenue / spend

    -- Approval
    approved_by VARCHAR(100),
    approval_date DATE,
    approval_status VARCHAR(20), -- pending, approved, rejected

    -- Notes
    notes TEXT,
    justification TEXT, -- why this budget is needed

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (season_id) REFERENCES seasons(season_id) ON DELETE CASCADE,
    UNIQUE KEY unique_season_budget (season_id, budget_category, sub_category),

    -- Indexes
    INDEX idx_season (season_id),
    INDEX idx_category (budget_category),
    INDEX idx_sub_category (sub_category)
);
```

---

### 15. Season Milestones Table (Additional Table)
**Purpose**: Track important milestones and tasks for each season

```sql
CREATE TABLE season_milestones (
    -- Primary Key
    milestone_id SERIAL PRIMARY KEY,

    -- Foreign Keys
    season_id VARCHAR(50) NOT NULL,

    -- Milestone Information
    milestone_name VARCHAR(255) NOT NULL,
    milestone_type VARCHAR(50),
    -- Types: planning, inventory, marketing, website, operations, review

    description TEXT,

    -- Timing
    deadline DATE NOT NULL,
    completed_date DATE,

    -- Status
    status VARCHAR(30) NOT NULL DEFAULT 'not_started',
    -- Status: not_started, in_progress, completed, overdue, cancelled

    priority VARCHAR(20), -- low, medium, high, critical

    -- Assignment
    responsible_party VARCHAR(100),
    assigned_team VARCHAR(100),

    -- Dependencies
    depends_on_milestone_id INTEGER, -- if this milestone depends on another
    blocking_milestone BOOLEAN DEFAULT FALSE, -- if other milestones depend on this

    -- Progress
    progress_percentage INTEGER DEFAULT 0, -- 0-100

    -- Notes
    notes TEXT,
    completion_notes TEXT,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    FOREIGN KEY (season_id) REFERENCES seasons(season_id) ON DELETE CASCADE,
    FOREIGN KEY (depends_on_milestone_id) REFERENCES season_milestones(milestone_id),

    -- Indexes
    INDEX idx_season (season_id),
    INDEX idx_deadline (deadline),
    INDEX idx_status (status),
    INDEX idx_responsible (responsible_party)
);
```

---

## Table Relationships

### Relationship Diagram Details

#### 1. Products Relationships
- **products** → **inventory** (1:N) - One product can be in multiple warehouses
- **products** → **inventory_transactions** (1:N) - One product has many inventory movements
- **products** → **order_items** (1:N) - One product can be in many orders
- **products** → **pricing_history** (1:N) - One product has price change history
- **products** → **competitor_pricing** (1:N) - One product has multiple competitor price checks
- **products** → **return_items** (1:N) - One product can be returned multiple times

#### 2. Customers Relationships
- **customers** → **orders** (1:N) - One customer can have many orders
- **customers** → **returns** (1:N) - One customer can have many returns

#### 3. Orders Relationships
- **orders** → **order_items** (1:N) - One order has multiple line items
- **orders** → **returns** (1:N) - One order can have multiple returns (different items)

#### 4. Returns Relationships
- **returns** → **return_items** (1:N) - One return has multiple items
- **order_items** → **return_items** (1:1 or 1:N) - One order item can be partially or fully returned

#### 5. Campaign Relationships
- **ad_campaigns** → **ad_performance** (1:N) - One campaign has daily performance records
- **ad_campaigns** → **seasons** (N:1) - Multiple campaigns can belong to one season

#### 6. Season Relationships
- **seasons** → **season_budgets** (1:N) - One season has multiple budget categories
- **seasons** → **season_milestones** (1:N) - One season has multiple milestones
- **seasons** → **ad_campaigns** (1:N) - One season can have multiple campaigns

---

## Indexes and Performance

### Primary Indexes (Already Defined in Tables)
All tables have primary keys and foreign key indexes defined.

### Additional Recommended Indexes for Performance

```sql
-- Products Table
CREATE INDEX idx_products_price_range ON products(current_price);
CREATE INDEX idx_products_status_category ON products(status, category);

-- Inventory Table
CREATE INDEX idx_inventory_low_stock ON inventory(sku, quantity_available)
WHERE quantity_available < 10;

-- Orders Table
CREATE INDEX idx_orders_date_status ON orders(order_date, order_status);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
CREATE INDEX idx_orders_channel_date ON orders(channel, order_date);

-- Order Items Table
CREATE INDEX idx_order_items_sku_date ON order_items(sku, created_at);
CREATE INDEX idx_order_items_category_date ON order_items(category, created_at);

-- Returns Table
CREATE INDEX idx_returns_date_status ON returns(return_request_date, return_status);

-- Return Items Table
CREATE INDEX idx_return_items_reason_date ON return_items(return_reason_category, created_at);

-- Ad Performance Table
CREATE INDEX idx_ad_perf_date_campaign ON ad_performance(date DESC, campaign_id);
CREATE INDEX idx_ad_perf_campaign_date ON ad_performance(campaign_id, date DESC);

-- Composite indexes for common queries
CREATE INDEX idx_orders_analytics ON orders(order_date, order_status, channel, traffic_source);
```

### Partitioning Strategy (for large datasets)

```sql
-- Partition orders table by year
CREATE TABLE orders_2024 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Partition ad_performance by month
CREATE TABLE ad_performance_2024_01 PARTITION OF ad_performance
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

---

## Sample Queries

### 1. Inventory Analysis Queries

#### Low Stock Alert
```sql
SELECT
    p.sku,
    p.product_name,
    p.category,
    i.quantity_available,
    p.reorder_point,
    p.safety_stock_level,
    (p.reorder_point - i.quantity_available) AS shortage
FROM products p
JOIN inventory i ON p.sku = i.sku
WHERE i.quantity_available < p.reorder_point
ORDER BY shortage DESC;
```

#### Inventory Turnover Ratio
```sql
SELECT
    p.sku,
    p.product_name,
    SUM(oi.quantity * oi.unit_cost) AS cogs_last_90_days,
    AVG(i.quantity_on_hand * p.unit_cost) AS avg_inventory_value,
    (SUM(oi.quantity * oi.unit_cost) / NULLIF(AVG(i.quantity_on_hand * p.unit_cost), 0)) * 4 AS annual_turnover_ratio
FROM products p
JOIN order_items oi ON p.sku = oi.sku
JOIN orders o ON oi.order_id = o.order_id
JOIN inventory i ON p.sku = i.sku
WHERE o.order_date >= CURRENT_DATE - INTERVAL '90 days'
    AND o.order_status = 'completed'
GROUP BY p.sku, p.product_name
ORDER BY annual_turnover_ratio DESC;
```

#### Dead Stock Report
```sql
SELECT
    p.sku,
    p.product_name,
    p.category,
    i.quantity_on_hand,
    (i.quantity_on_hand * p.unit_cost) AS dead_stock_value,
    MAX(o.order_date) AS last_sale_date,
    CURRENT_DATE - MAX(o.order_date) AS days_since_last_sale
FROM products p
JOIN inventory i ON p.sku = i.sku
LEFT JOIN order_items oi ON p.sku = oi.sku
LEFT JOIN orders o ON oi.order_id = o.order_id AND o.order_status = 'completed'
WHERE i.quantity_on_hand > 0
GROUP BY p.sku, p.product_name, p.category, i.quantity_on_hand, p.unit_cost
HAVING MAX(o.order_date) < CURRENT_DATE - INTERVAL '90 days'
    OR MAX(o.order_date) IS NULL
ORDER BY dead_stock_value DESC;
```

---

### 2. Sales Analysis Queries

#### Daily Sales Summary
```sql
SELECT
    DATE(order_date) AS sale_date,
    COUNT(DISTINCT order_id) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value,
    SUM(gross_profit) AS total_gross_profit,
    AVG(gross_margin_percent) AS avg_margin_percent
FROM orders
WHERE order_status = 'completed'
    AND order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE(order_date)
ORDER BY sale_date DESC;
```

#### Top Products by Revenue
```sql
SELECT
    oi.sku,
    oi.product_name,
    oi.category,
    COUNT(DISTINCT oi.order_id) AS order_count,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.line_total) AS total_revenue,
    AVG(oi.unit_price) AS avg_selling_price,
    SUM(oi.gross_profit) AS total_profit,
    AVG(oi.gross_margin_percent) AS avg_margin_percent
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_status = 'completed'
    AND o.order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY oi.sku, oi.product_name, oi.category
ORDER BY total_revenue DESC
LIMIT 50;
```

#### Customer RFM Segmentation
```sql
WITH customer_metrics AS (
    SELECT
        customer_id,
        CURRENT_DATE - MAX(order_date) AS recency_days,
        COUNT(order_id) AS frequency,
        SUM(total_amount) AS monetary
    FROM orders
    WHERE order_status = 'completed'
    GROUP BY customer_id
),
rfm_scores AS (
    SELECT
        customer_id,
        recency_days,
        frequency,
        monetary,
        NTILE(5) OVER (ORDER BY recency_days ASC) AS recency_score,
        NTILE(5) OVER (ORDER BY frequency DESC) AS frequency_score,
        NTILE(5) OVER (ORDER BY monetary DESC) AS monetary_score
    FROM customer_metrics
)
SELECT
    customer_id,
    recency_days,
    frequency,
    monetary,
    recency_score,
    frequency_score,
    monetary_score,
    CASE
        WHEN recency_score >= 4 AND frequency_score >= 4 AND monetary_score >= 4 THEN 'Champions'
        WHEN recency_score >= 3 AND frequency_score >= 3 THEN 'Loyal Customers'
        WHEN recency_score >= 4 AND frequency_score <= 2 THEN 'Promising'
        WHEN recency_score >= 3 AND monetary_score >= 4 THEN 'Big Spenders'
        WHEN recency_score <= 2 AND frequency_score >= 3 THEN 'At Risk'
        WHEN recency_score <= 2 AND frequency_score <= 2 THEN 'Lost'
        ELSE 'Regular'
    END AS rfm_segment
FROM rfm_scores
ORDER BY monetary DESC;
```

---

### 3. Return Analysis Queries

#### Return Rate by Product
```sql
SELECT
    p.sku,
    p.product_name,
    p.category,
    COUNT(DISTINCT oi.order_id) AS total_orders,
    SUM(oi.quantity) AS total_units_sold,
    COUNT(DISTINCT ri.return_id) AS return_count,
    SUM(ri.quantity_returned) AS total_units_returned,
    (SUM(ri.quantity_returned)::DECIMAL / NULLIF(SUM(oi.quantity), 0) * 100) AS return_rate_percent,
    SUM(ri.refund_amount) AS total_refund_amount
FROM products p
JOIN order_items oi ON p.sku = oi.sku
LEFT JOIN return_items ri ON oi.order_item_id = ri.order_item_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '90 days'
    AND o.order_status = 'completed'
GROUP BY p.sku, p.product_name, p.category
HAVING SUM(oi.quantity) > 10  -- only products with significant sales
ORDER BY return_rate_percent DESC
LIMIT 50;
```

#### Return Reason Analysis
```sql
SELECT
    return_reason_category,
    return_reason_subcategory,
    COUNT(*) AS return_count,
    SUM(quantity_returned) AS units_returned,
    SUM(refund_amount) AS total_refund,
    AVG(refund_amount) AS avg_refund,
    (COUNT(*)::DECIMAL / (SELECT COUNT(*) FROM return_items) * 100) AS percentage_of_returns
FROM return_items
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY return_reason_category, return_reason_subcategory
ORDER BY return_count DESC;
```

---

### 4. Advertising Analysis Queries

#### Campaign Performance Summary
```sql
SELECT
    c.campaign_id,
    c.campaign_name,
    c.platform,
    SUM(p.impressions) AS total_impressions,
    SUM(p.clicks) AS total_clicks,
    AVG(p.ctr) AS avg_ctr,
    SUM(p.spend) AS total_spend,
    AVG(p.cpc) AS avg_cpc,
    SUM(p.conversions) AS total_conversions,
    AVG(p.conversion_rate) AS avg_conversion_rate,
    SUM(p.revenue) AS total_revenue,
    (SUM(p.revenue) / NULLIF(SUM(p.spend), 0)) AS roas,
    (SUM(p.spend) / NULLIF(SUM(p.conversions), 0)) AS cpa
FROM ad_campaigns c
JOIN ad_performance p ON c.campaign_id = p.campaign_id
WHERE p.date >= CURRENT_DATE - INTERVAL '30 days'
    AND c.status = 'active'
GROUP BY c.campaign_id, c.campaign_name, c.platform
ORDER BY total_revenue DESC;
```

#### Daily Ad Performance Trend
```sql
SELECT
    date,
    SUM(spend) AS daily_spend,
    SUM(revenue) AS daily_revenue,
    (SUM(revenue) / NULLIF(SUM(spend), 0)) AS daily_roas,
    SUM(conversions) AS daily_conversions,
    (SUM(spend) / NULLIF(SUM(conversions), 0)) AS daily_cpa
FROM ad_performance
WHERE date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY date
ORDER BY date DESC;
```

---

### 5. Pricing Analysis Queries

#### Margin Analysis by Category
```sql
SELECT
    category,
    COUNT(DISTINCT sku) AS product_count,
    AVG(current_price) AS avg_price,
    AVG(unit_cost) AS avg_cost,
    AVG(((current_price - unit_cost) / NULLIF(current_price, 0)) * 100) AS avg_margin_percent,
    SUM(total_revenue) AS total_revenue,
    SUM(total_revenue - (total_sales_count * unit_cost)) AS total_profit
FROM products
WHERE status = 'active'
GROUP BY category
ORDER BY total_revenue DESC;
```

#### Price Change Impact Analysis
```sql
SELECT
    ph.sku,
    p.product_name,
    ph.old_price,
    ph.new_price,
    ph.price_change_percentage,
    ph.effective_date,
    ph.change_reason,
    ph.expected_revenue_impact,
    ph.actual_revenue_impact,
    CASE
        WHEN ph.actual_revenue_impact > ph.expected_revenue_impact THEN 'Exceeded'
        WHEN ph.actual_revenue_impact < ph.expected_revenue_impact THEN 'Underperformed'
        ELSE 'Met Expectation'
    END AS performance
FROM pricing_history ph
JOIN products p ON ph.sku = p.sku
WHERE ph.effective_date >= CURRENT_DATE - INTERVAL '90 days'
    AND ph.actual_revenue_impact IS NOT NULL
ORDER BY ph.effective_date DESC;
```

#### Competitor Price Comparison
```sql
SELECT
    p.sku,
    p.product_name,
    p.current_price AS our_price,
    AVG(cp.competitor_price) AS avg_competitor_price,
    MIN(cp.competitor_price) AS lowest_competitor_price,
    MAX(cp.competitor_price) AS highest_competitor_price,
    p.current_price - AVG(cp.competitor_price) AS price_difference,
    ((p.current_price - AVG(cp.competitor_price)) / NULLIF(AVG(cp.competitor_price), 0) * 100) AS price_difference_percent,
    CASE
        WHEN p.current_price < AVG(cp.competitor_price) THEN 'Below Market'
        WHEN p.current_price > AVG(cp.competitor_price) THEN 'Above Market'
        ELSE 'At Market'
    END AS price_position
FROM products p
JOIN competitor_pricing cp ON p.sku = cp.sku
WHERE cp.price_check_date >= CURRENT_DATE - INTERVAL '7 days'
    AND p.status = 'active'
GROUP BY p.sku, p.product_name, p.current_price
ORDER BY price_difference_percent DESC;
```

---

### 6. Season Planning Queries

#### Season Budget vs Actual
```sql
SELECT
    s.season_name,
    s.start_date,
    s.end_date,
    s.status,
    s.total_budget,
    s.revenue_target,
    s.profit_target,
    SUM(sb.budgeted_amount) AS total_budgeted,
    SUM(sb.actual_spend) AS total_spent,
    SUM(sb.remaining_budget) AS total_remaining,
    s.actual_revenue,
    s.actual_profit,
    (s.actual_revenue / NULLIF(s.revenue_target, 0) * 100) AS revenue_achievement_percent,
    (s.actual_profit / NULLIF(s.profit_target, 0) * 100) AS profit_achievement_percent
FROM seasons s
LEFT JOIN season_budgets sb ON s.season_id = sb.season_id
WHERE s.year = EXTRACT(YEAR FROM CURRENT_DATE)
GROUP BY s.season_id, s.season_name, s.start_date, s.end_date, s.status,
         s.total_budget, s.revenue_target, s.profit_target, s.actual_revenue, s.actual_profit
ORDER BY s.start_date;
```

#### Budget by Category
```sql
SELECT
    sb.budget_category,
    sb.sub_category,
    SUM(sb.budgeted_amount) AS budgeted,
    SUM(sb.actual_spend) AS spent,
    SUM(sb.remaining_budget) AS remaining,
    SUM(sb.budget_variance) AS variance,
    AVG(sb.budget_variance_percentage) AS avg_variance_percent,
    SUM(sb.revenue_generated) AS revenue,
    (SUM(sb.revenue_generated) / NULLIF(SUM(sb.actual_spend), 0)) AS roi
FROM season_budgets sb
JOIN seasons s ON sb.season_id = s.season_id
WHERE s.status = 'active'
GROUP BY sb.budget_category, sb.sub_category
ORDER BY spent DESC;
```

#### Year-over-Year Season Comparison
```sql
SELECT
    s1.season_name,
    s1.year AS current_year,
    s1.actual_revenue AS current_revenue,
    s1.actual_profit AS current_profit,
    s2.year AS previous_year,
    s2.actual_revenue AS previous_revenue,
    s2.actual_profit AS previous_profit,
    ((s1.actual_revenue - s2.actual_revenue) / NULLIF(s2.actual_revenue, 0) * 100) AS revenue_growth_percent,
    ((s1.actual_profit - s2.actual_profit) / NULLIF(s2.actual_profit, 0) * 100) AS profit_growth_percent
FROM seasons s1
JOIN seasons s2 ON s1.previous_season_id = s2.season_id
WHERE s1.status = 'completed'
    AND s2.status = 'completed'
ORDER BY s1.start_date DESC;
```

---

## Data Dictionary

### Common Field Types and Conventions

| Field Type | Format | Example | Description |
|------------|--------|---------|-------------|
| IDs | VARCHAR(50) | ORD-2024-00001 | Unique identifiers |
| SKU | VARCHAR(50) | SHIRT-BLU-L-001 | Product identifier |
| Dates | DATE | 2024-01-15 | Date only (YYYY-MM-DD) |
| Timestamps | TIMESTAMP | 2024-01-15 14:30:00 | Date and time |
| Currency | DECIMAL(10,2) | 1234.56 | Money values |
| Percentages | DECIMAL(5,2) | 15.50 | Percentage (stored as decimal) |
| Status | VARCHAR(20-30) | active, pending | Enumerated values |
| Boolean | BOOLEAN | TRUE/FALSE | Yes/No values |
| JSON | JSON | {"key": "value"} | Flexible structured data |

### Status Field Values

#### Order Status
- `pending` - Order placed, awaiting confirmation
- `confirmed` - Order confirmed, awaiting processing
- `processing` - Order being prepared
- `shipped` - Order shipped to customer
- `delivered` - Order delivered
- `completed` - Order finalized
- `cancelled` - Order cancelled
- `refunded` - Order refunded
- `partially_refunded` - Partial refund issued

#### Return Status
- `pending` - Return requested, awaiting approval
- `approved` - Return approved
- `rejected` - Return rejected
- `received` - Return received at warehouse
- `inspected` - Return inspected
- `refunded` - Refund processed
- `exchanged` - Exchange processed
- `completed` - Return fully processed

#### Campaign Status
- `active` - Campaign running
- `paused` - Campaign paused
- `ended` - Campaign completed
- `draft` - Campaign in planning
- `archived` - Campaign archived

#### Season Status
- `planning` - In planning phase
- `approved` - Budget and plan approved
- `active` - Season currently running
- `completed` - Season ended
- `cancelled` - Season cancelled

---

## Database Maintenance

### Regular Maintenance Tasks

#### Daily
```sql
-- Update materialized views (if used)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_inventory_summary;

-- Analyze tables for query optimization
ANALYZE orders;
ANALYZE order_items;
ANALYZE ad_performance;
```

#### Weekly
```sql
-- Vacuum to reclaim storage
VACUUM ANALYZE orders;
VACUUM ANALYZE inventory_transactions;

-- Update statistics
ANALYZE VERBOSE;
```

#### Monthly
```sql
-- Archive old data (example: archive orders older than 2 years)
INSERT INTO orders_archive
SELECT * FROM orders
WHERE order_date < CURRENT_DATE - INTERVAL '2 years';

DELETE FROM orders
WHERE order_date < CURRENT_DATE - INTERVAL '2 years';

-- Rebuild indexes
REINDEX TABLE orders;
REINDEX TABLE order_items;
```

---

## Security Considerations

### Row-Level Security Example
```sql
-- Enable row-level security on orders table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create policy: users can only see their own orders
CREATE POLICY customer_orders_policy ON orders
FOR SELECT
USING (customer_id = current_user_id());

-- Create policy: admins can see all orders
CREATE POLICY admin_orders_policy ON orders
FOR ALL
TO admin_role
USING (true);
```

### Sensitive Data Encryption
```sql
-- Encrypt sensitive customer data
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Example: Encrypt email addresses
UPDATE customers
SET email = pgp_sym_encrypt(email, 'encryption_key');

-- Decrypt when needed
SELECT pgp_sym_decrypt(email::bytea, 'encryption_key') AS email
FROM customers;
```

---

## Backup Strategy

### Full Backup (Daily)
```bash
pg_dump -U username -h localhost -F c -b -v -f "backup_$(date +%Y%m%d).dump" ecommerce_analytics
```

### Table-Specific Backup
```bash
pg_dump -U username -h localhost -t orders -t order_items -F c -f "orders_backup.dump" ecommerce_analytics
```

### Restore
```bash
pg_restore -U username -h localhost -d ecommerce_analytics -v "backup_20240115.dump"
```

---

**Document Version**: 1.0
**Last Updated**: 2024-01-15
**Database Version**: PostgreSQL 14+
**Author**: Development Team
