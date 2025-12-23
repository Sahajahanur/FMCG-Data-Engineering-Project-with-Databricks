
# FMCG Data Engineering Project with Databricks

## 📋 Project Overview

An end-to-end data engineering solution built on Databricks Free Edition for a merged FMCG company. This project demonstrates how to build a scalable data pipeline that consolidates data from two companies (Atlon - parent company and SportsBar - acquired startup) into a unified analytics platform.

## 🎯 Business Problem

**Atlon**, a leading sports equipment manufacturer, acquired **SportsBar**, a fast-growing energy bars and athletic nutrition startup. The acquisition created significant data challenges:

- Incompatible data formats between companies
- Misaligned reporting cycles
- Missing historical data from SportsBar's hyper-growth phase
- Need for unified supply chain forecasting and inventory planning

**Solution**: Build a reliable, scalable data layer that provides consolidated analytics for both companies.

## 🏗️ Architecture

```
OLTP (SportsBar) → AWS S3 → Bronze → Silver → Gold → BI Dashboard
                                                  ↓
OLTP (Atlon) → Existing Pipeline → Gold ←────────┘
```

### Medallion Architecture Layers

- **Bronze Layer**: Raw data ingestion from S3
- **Silver Layer**: Cleaned and transformed data
- **Gold Layer**: Business-ready aggregated data for analytics

## 🛠️ Technology Stack

- **Platform**: Databricks Free Edition
- **Languages**: Python (PySpark), SQL
- **Cloud Storage**: AWS S3
- **Architecture**: Medallion (Bronze-Silver-Gold)
- **BI Tools**: Databricks Dashboards, Genie (AI Assistant)
- **Orchestration**: Databricks Jobs

## 📊 Data Model

### Star Schema Design

**Fact Table:**
- `fact_orders` - Transaction data with daily/monthly granularity

**Dimension Tables:**
- `dim_customers` - Customer information (B2B clients)
- `dim_products` - Product catalog with categories and variants
- `dim_gross_price` - Pricing information by year/month
- `dim_date` - Date dimension for time-based analysis

## 🚀 Key Features

### 1. Dual Data Pipeline
- **Parent Company (Atlon)**: Established pipeline with historical data till Nov 2025
- **Child Company (SportsBar)**: New pipeline processing 5 months historical data + incremental updates

### 2. Data Processing
- **Full Load (Historical Backfill)**: Batch processing of historical data (July - November 2025)
- **Incremental Load**: Daily processing of new transactions (December 2025 onwards)

### 3. Data Quality & Transformations
- Duplicate removal
- Null value handling
- Date format standardization
- Data type conversions
- String cleaning (trimming, title case)
- Invalid data handling (negative prices, unknown values)
- City name standardization
- Product code generation using SHA hashing

### 4. Advanced Features
- **Change Data Feed (CDF)**: Row-level change tracking
- **Time Travel**: Audit and rollback capabilities
- **Upsert Operations**: Merge logic for incremental updates
- **Window Functions**: Monthly aggregation for parent company alignment
- **Staging Tables**: Efficient incremental processing

## 📁 Project Structure

```
consolidated-pipeline/
├── setup/
│   ├── setup_catalogs_and_tables.ipynb
│   └── dim_date_table_creation.ipynb
├── dimension_data_processing/
│   ├── customer_data_processing.ipynb
│   ├── products_data_processing.ipynb
│   └── gross_price_processing.ipynb
├── fact_data_processing/
│   ├── orders_full_load.ipynb
│   └── orders_incremental_load.ipynb
└── utilities.ipynb
```

## 🔧 Setup Instructions

### Prerequisites
1. Databricks Free Edition account ([Sign up here](https://www.databricks.com/try-databricks))
2. AWS Account (Free tier eligible)
3. Basic knowledge of Python, SQL, and PySpark

### Step 1: AWS S3 Setup
```bash
# Create S3 bucket
aws s3 mb s3://your-bucket-name

# Upload data
aws s3 cp parent_company/ s3://your-bucket-name/ --recursive
aws s3 cp child_company/ s3://your-bucket-name/ --recursive
```

### Step 2: Databricks Configuration
1. Create catalog and schemas:
```sql
CREATE CATALOG IF NOT EXISTS fmcg;
USE CATALOG fmcg;

CREATE SCHEMA IF NOT EXISTS bronze;
CREATE SCHEMA IF NOT EXISTS silver;
CREATE SCHEMA IF NOT EXISTS gold;
```

2. Establish S3 connection:
   - Navigate to Catalog → External Data → Create Connection
   - Follow AWS Quick Start guide
   - Test connection

### Step 3: Run Notebooks in Order

**Setup Phase:**
1. `setup_catalogs_and_tables.ipynb`
2. `dim_date_table_creation.ipynb`

**Dimension Processing:**
3. `customer_data_processing.ipynb`
4. `products_data_processing.ipynb`
5. `gross_price_processing.ipynb`

**Fact Processing:**
6. `orders_full_load.ipynb` (historical backfill)
7. `orders_incremental_load.ipynb` (daily updates)

### Step 4: Create Orchestration Job
1. Navigate to Workflows → Create Job
2. Add tasks in sequence:
   - Dim Customer Processing
   - Dim Products Processing
   - Dim Gross Price Processing
   - Fact Orders Processing
3. Set dependencies between tasks
4. Schedule job (e.g., daily at 11 PM)

### Step 5: Build Dashboard
1. Create denormalized view:
```sql
CREATE OR REPLACE VIEW fmcg.gold.sales_360_view AS
SELECT 
    f.*,
    d.year, d.quarter, d.month,
    c.customer, c.market, c.platform, c.channel,
    p.division, p.category, p.product, p.variant,
    gp.gross_price_inr,
    f.sold_quantity * gp.gross_price_inr as total_revenue
FROM fmcg.gold.fact_orders f
LEFT JOIN fmcg.gold.dim_date d ON f.date = d.date_key
LEFT JOIN fmcg.gold.dim_customers c ON f.customer_code = c.customer_code
LEFT JOIN fmcg.gold.dim_products p ON f.product_code = p.product_code
LEFT JOIN fmcg.gold.dim_gross_price gp ON f.product_code = gp.product_code;
```

2. Create Databricks Dashboard using the view
3. Add visualizations (KPIs, charts, filters)

## 💡 Key Transformations

### Customer Data
- Remove leading/trailing spaces
- Standardize city names (handle typos)
- Convert customer ID to string (categorical data)
- Create customer column: `customer_name + city`
- Add market, platform, channel columns

### Products Data
- Split product name into product + variant
- Fix spelling errors (e.g., "protien" → "protein")
- Generate surrogate key using SHA hashing
- Handle invalid product IDs
- Add division column based on category mapping

### Gross Price
- Normalize date formats
- Convert negative prices to positive
- Handle "unknown" values → 0
- Add product code from products table
- Aggregate monthly prices to yearly for parent alignment

### Orders (Fact)
- Remove duplicates
- Filter null quantities
- Handle invalid customer IDs
- Standardize date formats
- Monthly aggregation for parent company merge
- Daily granularity for child company

## 📈 Sample Analytics Queries

```sql
-- Top 5 products by revenue
SELECT product, SUM(total_revenue) as revenue
FROM fmcg.gold.sales_360_view
GROUP BY product
ORDER BY revenue DESC
LIMIT 5;

-- Revenue by quarter
SELECT quarter, SUM(total_revenue) as revenue
FROM fmcg.gold.sales_360_view
GROUP BY quarter
ORDER BY quarter;

-- Top customers by sold quantity
SELECT customer, SUM(sold_quantity) as total_quantity
FROM fmcg.gold.sales_360_view
GROUP BY customer
ORDER BY total_quantity DESC
LIMIT 5;
```

## 🤖 Using Genie (AI Assistant)

Ask natural language questions:
- "What are the top 10 products by revenue?"
- "Show me total revenue by quarter"
- "Which customers bought the most products?"

## 📊 Dashboard Features

- **Filters**: Year, Quarter, Month, Channel, Category
- **KPIs**: Total Revenue, Total Quantity Sold
- **Visualizations**:
  - Top 10 products by revenue
  - Revenue share by channel (pie chart)
  - Monthly revenue trend
  - Top customers by quantity
  - Product variant analysis

## 🎓 Learning Outcomes

- Medallion architecture implementation
- PySpark data transformations
- Incremental data processing patterns
- Upsert/merge operations
- Data quality and cleaning techniques
- Orchestration and scheduling
- Dashboard development
- AWS S3 integration with Databricks

## 🐛 Common Issues & Solutions

**Issue**: S3 connection fails
- **Solution**: Verify IAM permissions and regenerate access token

**Issue**: Schema mismatch errors
- **Solution**: Check data types in bronze layer, use explicit casting

**Issue**: Duplicate records in gold layer
- **Solution**: Review upsert conditions and matching keys

**Issue**: Date format inconsistencies
- **Solution**: Use `coalesce` with multiple `try_to_date` formats

## 📚 Resources

- [Databricks Documentation](https://docs.databricks.com)
- [PySpark SQL Functions](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html)
- [Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
- [Delta Lake Features](https://docs.delta.io/latest/delta-intro.html)

## 🤝 Contributing

Feel free to fork this project and submit pull requests for improvements!

## 📝 License

This project is created for educational purposes.

## 👨‍💻 Author

Created as part of a comprehensive data engineering learning project.

## 🙏 Acknowledgments

Special thanks to Databricks for providing the free edition that makes this learning project accessible to everyone.

Codebasics, Dhaval Patel and Hemanand Vadivel for the wonderful free course available in the YouTube. 

---

## 📧 Contact

For questions or feedback, please create an issue in this repository.

**Remember**: This is a learning project. In production environments, implement proper security, error handling, monitoring, and testing practices.
