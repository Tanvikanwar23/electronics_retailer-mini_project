
# ELECTRONICS RETAILER MINI-PROJECT DOCUMENTATION
=====================================================

## PROJECT OVERVIEW
This is a complete data engineering project implementing the Medallion Architecture (Bronze-Silver-Gold) 
for an electronics retailer's sales and operational data. The project uses Unity Catalog for data governance 
and Delta Lake format for all tables.

**Catalog Name:** electronics_retailer_clg
**Architecture Pattern:** Medallion (Bronze → Silver → Gold)
**Data Format:** Delta Lake
**Total Tables:** 17 tables across 3 layers

---

## ARCHITECTURE DIAGRAM
```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                              │
│  • Customer Data (CSV)    • Product Data (CSV)                  │
│  • Sales Transactions     • Store Data (CSV)                    │
│  • Exchange Rates (CSV)   • Data Dictionary                     │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                     BRONZE LAYER (Raw Data)                      │
│  Catalog: electronics_retailer_clg | Schema: bronze              │
├─────────────────────────────────────────────────────────────────┤
│  • customers (15,266 rows)      • products (2,517 rows)         │
│  • stores (67 rows)             • sales (62,884 rows)           │
│  • exchange_rates (11,215 rows) • data_dictionary               │
│                                                                  │
│  Purpose: Ingest raw data as-is, minimal transformations        │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SILVER LAYER (Cleaned Data)                   │
│  Catalog: electronics_retailer_clg | Schema: silver              │
├─────────────────────────────────────────────────────────────────┤
│  • customers (15,258 rows)      • products (2,517 rows)         │
│  • customers_data               • stores (67 rows)              │
│  • sales (37,143 rows)          • exchange_rates                │
│                                                                  │
│  Purpose: Data cleaning, deduplication, type conversions        │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                  GOLD LAYER (Business Ready)                     │
│  Catalog: electronics_retailer_clg | Schema: gold                │
├─────────────────────────────────────────────────────────────────┤
│  FACT TABLE:                                                     │
│  • fact_sales (83,311,749 rows) - Denormalized sales fact       │
│                                                                  │
│  DIMENSION TABLES:                                               │
│  • dim_customers (15,258 rows)  • dim_products (2,517 rows)     │
│  • dim_stores (67 rows)         • dim_dates (2,192 rows)        │
│                                                                  │
│  Purpose: Star schema for analytics, aggregated metrics         │
└─────────────────────────────────────────────────────────────────┘
```

---

### BRONZE LAYER - DETAILED TABLE SCHEMAS
Catalog: electronics_retailer_clg | Schema: bronze

┌──────────────────────────────────────────────────────────────────────┐
│ 1. bronze.customers (15,266 rows)                                    │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Raw customer demographic and location data              │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name    │ Data Type │ Description                             │
├────────────────┼───────────┼─────────────────────────────────────────┤
│ CustomerKey    │ INT       │ Unique customer identifier              │
│ Gender         │ STRING    │ Customer gender (2 distinct values)     │
│ Name           │ STRING    │ Customer full name                      │
│ City           │ STRING    │ Customer city (8,598 distinct)          │
│ State_Code     │ STRING    │ State abbreviation code                 │
│ State          │ STRING    │ Full state name (492 distinct)          │
│ Zip_Code       │ STRING    │ Postal code                             │
│ Country        │ STRING    │ Country name (7 distinct)               │
│ Continent      │ STRING    │ Continent (3 distinct)                  │
│ Birthday       │ DATE      │ Customer date of birth                  │
└────────────────┴───────────┴─────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 2. bronze.products (2,517 rows)                                      │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Raw product catalog with pricing and categorization     │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name      │ Data Type │ Description                           │
├──────────────────┼───────────┼───────────────────────────────────────┤
│ ProductKey       │ INT       │ Unique product identifier             │
│ Product_Name     │ STRING    │ Full product name (max 83 chars)      │
│ Brand            │ STRING    │ Product brand (11 distinct brands)    │
│ Color            │ STRING    │ Product color (16 distinct colors)    │
│ Unit_Cost_USD    │ STRING    │ Product cost (510 distinct values)    │
│ Unit_Price_USD   │ STRING    │ Product price (437 distinct values)   │
│ SubcategoryKey   │ INT       │ Product subcategory ID (32 distinct)  │
│ Subcategory      │ STRING    │ Subcategory name (31 distinct)        │
│ CategoryKey      │ INT       │ Product category ID (8 categories)    │
│ Category         │ STRING    │ Category name (8 categories)          │
└──────────────────┴───────────┴───────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 3. bronze.stores (67 rows)                                           │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Raw store location and physical attributes              │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name    │ Data Type │ Description                             │
├────────────────┼───────────┼─────────────────────────────────────────┤
│ StoreKey       │ INT       │ Unique store identifier (0-66)          │
│ Country        │ STRING    │ Store country (8 distinct)              │
│ State          │ STRING    │ Store state (67 distinct)               │
│ Square_Meters  │ INT       │ Store size (245-2,105 sq meters)        │
│ Open_Date      │ DATE      │ Store opening date (2005-2019)          │
└────────────────┴───────────┴─────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 4. bronze.sales (62,884 rows)                                        │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Raw sales transaction records                           │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name     │ Data Type │ Description                            │
├─────────────────┼───────────┼────────────────────────────────────────┤
│ Order_Number    │ INT       │ Unique order identifier                │
│ Line_Item       │ INT       │ Line item number within order          │
│ Order_Date      │ STRING    │ Date order was placed (raw format)     │
│ Delivery_Date   │ STRING    │ Date order was delivered (raw format)  │
│ CustomerKey     │ INT       │ Foreign key to customers               │
│ StoreKey        │ INT       │ Foreign key to stores                  │
│ ProductKey      │ INT       │ Foreign key to products                │
│ Quantity        │ INT       │ Quantity of items ordered              │
│ Currency_Code   │ STRING    │ Transaction currency code              │
└─────────────────┴───────────┴────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 5. bronze.exchange_rates (11,215 rows)                               │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Historical currency exchange rates                      │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name │ Data Type │ Description                                │
├─────────────┼───────────┼────────────────────────────────────────────┤
│ Date        │ DATE      │ Exchange rate effective date              │
│ Currency    │ STRING    │ Currency code                             │
│ Exchange    │ DOUBLE    │ Exchange rate to USD                      │
└─────────────┴───────────┴────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 6. bronze.data_dictionary                                            │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Metadata and data definitions for the project           │
└──────────────────────────────────────────────────────────────────────┘

"""

### SILVER LAYER - DETAILED TABLE SCHEMAS
Catalog: electronics_retailer_clg | Schema: silver

Purpose: Cleaned and deduplicated data with proper data types

┌──────────────────────────────────────────────────────────────────────┐
│ 1. silver.customers (15,258 rows)                                    │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Cleaned customer data with metadata                     │
│ Data Quality: Duplicates removed (15,266 → 15,258 rows)              │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name     │ Data Type │ Description                            │
├─────────────────┼───────────┼────────────────────────────────────────┤
│ customerkey     │ INT       │ Unique customer identifier             │
│ gender          │ STRING    │ Customer gender                        │
│ continent       │ STRING    │ Customer continent                     │
│ ingestion_time  │ TIMESTAMP │ Data ingestion timestamp               │
│ source_file     │ STRING    │ Source file name for lineage tracking  │
└─────────────────┴───────────┴────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 2. silver.customers_data (full customer details)                     │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Extended customer information with all attributes       │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name     │ Data Type │ Description                            │
├─────────────────┼───────────┼────────────────────────────────────────┤
│ customerkey     │ INT       │ Unique customer identifier             │
│ gender          │ STRING    │ Customer gender                        │
│ name            │ STRING    │ Customer full name                     │
│ city            │ STRING    │ Customer city                          │
│ state_code      │ STRING    │ State code                             │
│ state           │ STRING    │ State name                             │
│ zip_code        │ STRING    │ Postal code                            │
│ country         │ STRING    │ Country name                           │
│ continent       │ STRING    │ Continent                              │
│ birthday        │ DATE      │ Customer date of birth                 │
│ ingestion_time  │ TIMESTAMP │ Data ingestion timestamp               │
│ source_file     │ STRING    │ Source file tracking                   │
└─────────────────┴───────────┴────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 3. silver.products (2,517 rows)                                      │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Cleaned product catalog with key attributes             │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name      │ Data Type │ Description                           │
├──────────────────┼───────────┼───────────────────────────────────────┤
│ productkey       │ INT       │ Unique product identifier             │
│ category         │ STRING    │ Product category                      │
│ unit_price_usd   │ DOUBLE    │ Product price (converted to numeric)  │
└──────────────────┴───────────┴───────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 4. silver.stores (67 rows)                                           │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Cleaned store information with channel assignment       │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name │ Data Type │ Description                                │
├─────────────┼───────────┼────────────────────────────────────────────┤
│ storekey    │ INT       │ Unique store identifier                   │
│ country     │ STRING    │ Store country                             │
└─────────────┴───────────┴────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 5. silver.sales (37,143 rows)                                        │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Cleaned sales transactions (deduplicated)               │
│ Data Quality: Aggregated/cleaned (62,884 → 37,143 rows)              │
│ Top Joins: → gold.dim_customers, gold.dim_stores                     │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name     │ Data Type │ Description                            │
├─────────────────┼───────────┼────────────────────────────────────────┤
│ order_number    │ INT       │ Unique order identifier                │
│ order_date      │ DATE      │ Order date (converted from string)     │
│ delivery_date   │ DATE      │ Delivery date (converted from string)  │
│ customerkey     │ INT       │ Foreign key to customers               │
│ storekey        │ INT       │ Foreign key to stores                  │
│ productkey      │ INT       │ Foreign key to products                │
│ quantity        │ INT       │ Quantity ordered                       │
│ currency_code   │ STRING    │ Transaction currency                   │
└─────────────────┴───────────┴────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ 6. silver.exchange_rates                                             │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Cleaned exchange rates for currency conversion          │
└──────────────────────────────────────────────────────────────────────┘

"""

### GOLD LAYER - STAR SCHEMA DESIGN
Catalog: electronics_retailer_clg | Schema: gold

Purpose: Business-ready data in star schema for analytics and reporting
Architecture: 1 Fact Table + 4 Dimension Tables

┌──────────────────────────────────────────────────────────────────────┐
│ FACT TABLE: gold.fact_sales (83,311,749 rows)                        │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Denormalized sales fact table with all dimensions       │
│ Granularity: One row per sales transaction line item                 │
│ Foreign Keys: customerkey, storekey, productkey                      │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name        │ Data Type │ Category  │ Description             │
├────────────────────┼───────────┼───────────┼─────────────────────────┤
│ order_number       │ INT       │ ID        │ Order identifier        │
│ order_date         │ DATE      │ Date      │ Order placement date    │
│ delivery_date      │ DATE      │ Date      │ Delivery date           │
│ delivery_time_days │ INT       │ Metric    │ Days to deliver         │
│ customerkey        │ INT       │ FK        │ → dim_customers         │
│ gender             │ STRING    │ Dimension │ Customer gender         │
│ continent          │ STRING    │ Dimension │ Customer continent      │
│ storekey           │ INT       │ FK        │ → dim_stores            │
│ store_country      │ STRING    │ Dimension │ Store location country  │
│ channel            │ STRING    │ Dimension │ Sales channel           │
│ productkey         │ INT       │ FK        │ → dim_products          │
│ product_category   │ STRING    │ Dimension │ Product category        │
│ unit_price_usd     │ DOUBLE    │ Metric    │ Unit price in USD       │
│ quantity           │ INT       │ Metric    │ Quantity sold           │
│ currency_code      │ STRING    │ Attribute │ Original currency       │
│ exchange_rate      │ DOUBLE    │ Metric    │ Currency conversion rate│
│ revenue_usd        │ DOUBLE    │ Metric    │ Total revenue in USD    │
└────────────────────┴───────────┴───────────┴─────────────────────────┘

Key Metrics:
  • revenue_usd: Calculated as (unit_price_usd × quantity × exchange_rate)
  • delivery_time_days: Calculated as (delivery_date - order_date)


┌──────────────────────────────────────────────────────────────────────┐
│ DIMENSION: gold.dim_customers (15,258 rows)                          │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Customer dimension with demographics                    │
│ Primary Key: customerkey                                             │
│ Joins To: fact_sales, silver.sales                                   │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name │ Data Type │ Description                                │
├─────────────┼───────────┼────────────────────────────────────────────┤
│ customerkey │ INT       │ Unique customer identifier (PK)           │
│ gender      │ STRING    │ Customer gender                           │
│ continent   │ STRING    │ Geographic continent                      │
└─────────────┴───────────┴────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ DIMENSION: gold.dim_products (2,517 rows)                            │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Product dimension with category and pricing             │
│ Primary Key: productkey                                              │
│ Joins To: fact_sales, silver.sales                                   │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name      │ Data Type │ Description                           │
├──────────────────┼───────────┼───────────────────────────────────────┤
│ productkey       │ INT       │ Unique product identifier (PK)        │
│ category         │ STRING    │ Product category                      │
│ unit_price_usd   │ DOUBLE    │ Standard unit price in USD            │
└──────────────────┴───────────┴───────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ DIMENSION: gold.dim_stores (67 rows)                                 │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Store dimension with location and channel               │
│ Primary Key: storekey                                                │
│ Joins To: fact_sales, silver.sales                                   │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name │ Data Type │ Description                                │
├─────────────┼───────────┼────────────────────────────────────────────┤
│ storekey    │ INT       │ Unique store identifier (PK)              │
│ country     │ STRING    │ Store country location                    │
│ channel     │ STRING    │ Sales channel (Online/In-store)           │
└─────────────┴───────────┴────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ DIMENSION: gold.dim_dates (2,192 rows)                               │
├──────────────────────────────────────────────────────────────────────┤
│ Description: Date dimension for time-based analysis                  │
│ Primary Key: date                                                    │
│ Coverage: Multi-year date range                                      │
├──────────────────────────────────────────────────────────────────────┤
│ Column Name  │ Data Type │ Description                               │
├──────────────┼───────────┼───────────────────────────────────────────┤
│ date         │ TIMESTAMP │ Full date timestamp (PK)                  │
│ year         │ INT       │ Year component                            │
│ month        │ INT       │ Month number (1-12)                       │
│ quarter      │ INT       │ Quarter (1-4)                             │
│ month_name   │ STRING    │ Full month name                           │
│ day_name     │ STRING    │ Full day name                             │
│ day_of_week  │ INT       │ Day of week number                        │
└──────────────┴───────────┴───────────────────────────────────────────┘

"""

## DATA FLOW AND TRANSFORMATIONS

### Bronze → Silver Transformations
1. **Data Type Conversions**
   • Order_Date, Delivery_Date: STRING → DATE
   • Unit_Price_USD, Unit_Cost_USD: STRING → DOUBLE
   
2. **Data Quality**
   • Deduplication: customers (15,266 → 15,258 rows)
   • Null handling and validation
   • Key normalization (lowercase column names)
   
3. **Metadata Addition**
   • ingestion_time: Timestamp of data load
   • source_file: Lineage tracking

### Silver → Gold Transformations
1. **Star Schema Creation**
   • Extract dimensions from normalized tables
   • Create dimension tables (dim_customers, dim_products, dim_stores, dim_dates)
   • Build denormalized fact table with all dimensions
   
2. **Business Logic**
   • Calculate delivery_time_days: delivery_date - order_date
   • Calculate revenue_usd: unit_price_usd × quantity × exchange_rate
   • Assign sales channels based on store attributes
   
3. **Data Enrichment**
   • Join exchange rates for multi-currency support
   • Add customer demographics to fact table
   • Include product categories for analysis

---

## PROJECT STRUCTURE

Workspace: /Users/kanwartanvi93@gmail.com/electronics-retailer_mini-project/

```
electronics-retailer_mini-project/
├── Bronze_Layer/
│   ├── customers_bronze_nb          (Loads customer data)
│   ├── products_bronze_nb           (Loads product catalog)
│   ├── stores_bronze_nb             (Loads store information)
│   ├── sales_bronze_nb              (Loads sales transactions)
│   ├── exchange_rates_bronze_nb     (Loads currency rates)
│   └── data_dictionary_bronze_nb    (Metadata definitions)
│
├── Silver_Layer/
│   └── (Transformation notebooks - to be added)
│
└── Gold_Layer/
    └── (Analytics notebooks - to be added)
```

---

## KEY RELATIONSHIPS (STAR SCHEMA)

```
                    ┌─────────────────┐
                    │  dim_dates      │
                    │  (2,192 rows)   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼────────┐   ┌──────▼──────────┐   ┌────▼──────────┐
│ dim_customers  │   │   fact_sales    │   │ dim_products  │
│ (15,258 rows)  │◄──┤ (83,311,749)    │──►│ (2,517 rows)  │
└────────────────┘   │                 │   └───────────────┘
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │  dim_stores     │
                     │  (67 rows)      │
                     └─────────────────┘
```

Relationships:
• fact_sales.customerkey → dim_customers.customerkey
• fact_sales.productkey → dim_products.productkey
• fact_sales.storekey → dim_stores.storekey
• fact_sales.order_date, delivery_date → dim_dates.date

---

## CONTACT AND MAINTENANCE

**Project Owner:** kanwartanvi93@gmail.com

## QUICK ACCESS LINKS

### Browse Tables in Databricks:
• Bronze Layer: electronics_retailer_clg.bronze
• Silver Layer: electronics_retailer_clg.silver  
• Gold Layer: electronics_retailer_clg.gold

### Key Analytics Tables:
• Fact Table: electronics_retailer_clg.gold.fact_sales
• Customer Dimension: electronics_retailer_clg.gold.dim_customers
• Product Dimension: electronics_retailer_clg.gold.dim_products
• Store Dimension: electronics_retailer_clg.gold.dim_stores
• Date Dimension: electronics_retailer_clg.gold.dim_dates


## SUMMARY STATISTICS

Data Volume by Layer:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Bronze:   91,949 rows (raw data ingestion)
Silver:   70,203+ rows (cleaned & deduplicated)
Gold:     83,351,783 rows (analytics-ready)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Largest Tables:
1. gold.fact_sales:        83,311,749 rows ⭐ Primary Fact Table
2. bronze.sales:           62,884 rows
3. silver.sales:           37,143 rows
4. bronze.customers:       15,266 rows
5. silver.customers:       15,258 rows

Dimension Tables:
• dim_customers:           15,258 unique customers
• dim_products:            2,517 products across 8 categories
• dim_stores:              67 stores in 8 countries
• dim_dates:               2,192 date records

---

## DATA COVERAGE

Geographic Coverage:
• Countries: 8 (Store locations)
• Continents: 3 (Customer base)
• States/Regions: 67 unique store locations

Product Coverage:
• Categories: 8
• Subcategories: 31
• Brands: 11
• Total Products: 2,517
• Colors Available: 16

Customer Base:
• Total Customers: 15,258
• Geographic Spread: 3 continents, 7 countries
• Cities: 8,598 unique cities

Transaction Data:
• Total Sales Records: 83+ million rows
• Currencies Supported: Multiple (with USD conversion)
• Exchange Rate History: 11,215+ records

---

📊 MEDALLION ARCHITECTURE IMPLEMENTATION COMPLETE 📊

This project successfully implements a production-grade medallion architecture
with proper data governance, lineage tracking, and a star schema optimized
for analytical queries and business intelligence.
""")
