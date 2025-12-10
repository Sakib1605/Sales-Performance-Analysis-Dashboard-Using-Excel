# Sales Performance Analysis Dashboard Using Excel
Designed an interactive Excel dashboard to analyze sales trends, customer segmentation, product analytics, and geographic insights with time-based analysis for seasonal trends. Implemented VBA macros with event-driven programming to automate data updates, dynamic filtering, and report generation.


## **Objective**

The objective of the **Sales Performance Analysis Dashboard** was to create a comprehensive, interactive, and automated solution for analyzing sales performance across multiple dimensions. This dashboard enables businesses to gain actionable insights by dynamically filtering data, visualizing key metrics, and generating automated reports based on user-defined filters. 

Key goals included:
- Integrate multiple datasets (Customers, Orders, and Sales) into a unified, analytics-ready dataset using SQL for efficient querying and scalable processing.
- Design an intuitive Excel dashboard that offers dynamic insights into sales trends, customer segments, top-performing products, regional patterns, and shipping preferences.
- Implement automation using VBA to streamline data updates, apply user-defined filters, and generate custom PDF reports without manual intervention.

---

## **Screenshot of the Dashboard**

![Sales Performance Analysis Dashboard](dashboard_screenshot.png)

---

## **Steps Taken**

### **1. Data Integration and Transformation**

- **Datasets Used**:
  - **Customers Dataset**: Contained Customer ID, Name, Segment, and other details.
  - **Orders Dataset**: Included Order ID, Order Date, Ship Mode, State, and related attributes.
  - **Sales Dataset**: Provided transactional data like Product, Sales Amount, Quantity, and Profit.

- **Data Cleaning and Preprocessing**:
 To ensure accuracy, scalability, and clean data structure, all raw datasets were first imported into a SQL database and processed through a structured data engineering workflow.
SQL was used to clean and standardize all tables before analysis:

- Remove duplicates
- Fix date formats
- standardize category & segment text fields
- Ensure referential integrity across tables

### SQL Data Cleaning

```sql
-- Standardize date formats
UPDATE orders
SET 
    order_date = CONVERT(date, order_date),
    ship_date  = CONVERT(date, ship_date);

-- Remove duplicate customers
DELETE FROM customers
WHERE customer_id IN (
    SELECT customer_id
    FROM (
        SELECT 
            customer_id,
            ROW_NUMBER() OVER (
                PARTITION BY customer_id 
                ORDER BY customer_name
            ) AS rn
        FROM customers
    ) t
    WHERE rn > 1
);
```
### Integrating the Datasets with SQL JOINs
```sql
CREATE VIEW unified_sales AS
SELECT 
    o.order_id,
    o.order_date,
    o.ship_date,
    o.state,
    o.ship_mode,
    c.customer_id,
    c.customer_name,
    c.segment,
    s.product_name,
    s.category,
    s.sub_category,
    s.sales_amount,
    s.quantity,
    s.profit
FROM sales s
JOIN orders o 
    ON s.order_id = o.order_id
JOIN customers c 
    ON o.customer_id = c.customer_id;

```


---

### **2. SQL Analytics (Answering Business Questions)**

SQL queries were used to derive analytical insights.
#### Business Question: How do monthly sales trend over time?

```sql
SELECT 
    DATEFROMPARTS(YEAR(order_date), MONTH(order_date), 1) AS month_start,
    SUM(sales_amount) AS total_sales
FROM unified_sales
GROUP BY DATEFROMPARTS(YEAR(order_date), MONTH(order_date), 1)
ORDER BY month_start;
```
#### Interpretation: 
This query identifies sales trends by month, showing peaks and dips across the year. The results reveal seasonal patterns, helping businesses forecast demand and optimize inventory and promotional planning.

#### Business Question: How is each product category performing over time, and which categories show accelerating or declining sales momentum?

```sql
WITH category_monthly AS (
    SELECT
        DATEFROMPARTS(YEAR(order_date), MONTH(order_date), 1) AS month_start,
        category,
        SUM(sales_amount) AS monthly_sales
    FROM unified_sales
    GROUP BY 
        DATEFROMPARTS(YEAR(order_date), MONTH(order_date), 1),
        category
),
growth_calc AS (
    SELECT
        month_start,
        category,
        monthly_sales,
        LAG(monthly_sales) OVER (
            PARTITION BY category 
            ORDER BY month_start
        ) AS prev_month_sales
    FROM category_monthly
)
SELECT
    month_start,
    category,
    monthly_sales,
    prev_month_sales,
    (monthly_sales - prev_month_sales) AS sales_change,
    CASE
        WHEN prev_month_sales IS NULL OR prev_month_sales = 0 THEN NULL
        ELSE (monthly_sales - prev_month_sales) * 1.0 / prev_month_sales
    END AS growth_rate
FROM growth_calc
ORDER BY month_start, category;

```
#### Interpretation: 
This analysis provides early-warning signals and growth insights for product planning:

- Positive growth rate → category is gaining traction; consider increasing stock, marketing push, or premium pricing

- Negative growth rate → early signs of decline; investigate causes (competition, seasonality, pricing issues)

- Large month-over-month changes highlight categories experiencing rapid shifts in customer demand

- Tracking categories over time helps leadership prioritize investments, forecast demand, and manage inventory risk




#### Business Question: Which products show early signs of becoming future top performers?

```sql
WITH monthly_product_sales AS (
    SELECT
        DATEFROMPARTS(YEAR(order_date), MONTH(order_date), 1) AS month_start,
        product_name,
        SUM(sales_amount) AS total_sales
    FROM unified_sales
    GROUP BY DATEFROMPARTS(YEAR(order_date), MONTH(order_date), 1), product_name
),
growth_calc AS (
    SELECT
        product_name,
        month_start,
        total_sales,
        LAG(total_sales) OVER (PARTITION BY product_name ORDER BY month_start) AS prev_sales
    FROM monthly_product_sales
)
SELECT
    product_name,
    month_start,
    total_sales,
    prev_sales,
    (total_sales - prev_sales) AS change_in_sales,
    CASE 
        WHEN prev_sales = 0 OR prev_sales IS NULL THEN NULL
        ELSE (total_sales - prev_sales) * 1.0 / prev_sales
    END AS growth_rate
FROM growth_calc
WHERE month_start >= DATEADD(month, -3, (SELECT MAX(month_start) FROM monthly_product_sales))
ORDER BY growth_rate DESC;

```
#### Interpretation: 
This identifies rising products based on recent growth trends:
- Helps detect products that may become next quarter’s top sellers

- Supports demand forecasting and stocking decisions

- Allows early investment in products showing early momentum

#### Business Question 3: Which geographic regions generate the highest order volume?

```sql
SELECT TOP 5 
    state,
    COUNT(order_id) AS total_orders
FROM unified_sales
GROUP BY state
ORDER BY total_orders DESC;


```
#### Interpretation: 
Highlighting top-performing states helps identify geographic demand clusters. These insights can be used for: Regional sales prioritization & Optimizing inventory distribution.

#### Business Question 5: Which product categories generate the highest sales, and where should inventory be optimized?

```sql
SELECT 
    category,
    SUM(sales_amount) AS total_sales,
    SUM(quantity) AS total_units_sold
FROM unified_sales
GROUP BY category
ORDER BY total_sales DESC;

```
#### Interpretation: 
This identifies which product categories drive revenue and volume. Businesses use this insight to:

- Prioritize high-performing categories

- Optimize stock levels

- Plan category-specific promotions

- Identify underperforming categories needing review


### **3. Dashboard Design**

The dashboard was created in Excel with a focus on interactivity, clarity, and ease of use.

#### **Key Visualizations**:
- **Line Chart**: Tracks total sales over time, segmented by product categories.
- **Bar Chart**: Highlights contributions of sales across products.
- **Pie Chart**: Visualizes order distribution by shipping modes.
- **Horizontal Bar Chart**: Displays the top 5 states by the total number of orders.
- **Column Chart**: Examines average shipping time across shipping modes and regions.

#### **Key Metrics (KPIs)**:
- **Total Orders**: Total number of orders processed.
- **Total Sales**: Cumulative sales amount across all transactions.
- **Average Sales Per Order**: Average revenue generated per order.

#### **Interactive Features**:
- **Slicers**:
  - Filter data by date (month/year), ship mode, product category, sub-category, state, and customer segment.
- **Dynamic Updates**:
  - All visualizations and KPIs update automatically based on slicer selections.

---

### **3. Automation Using VBA**

To enhance usability, VBA was implemented to automate report generation.

#### **Custom Report Generation**:
- A button labeled **"Click to Generate Report"** was added to the dashboard.
- The VBA macro:
  - Consolidates filtered data from PivotTables affected by slicers.
  - Creates a new worksheet with the report structured as:
    - **Section Headers**: "Analysis -> Sheet Name."
    - **Data with Colorful Headers**: Filtered data is organized for clarity.
  - Exports the report as a **PDF** with the current date in the filename.

#### **Key VBA Features**:
- **Event-Driven Execution**: Macro executes only when the button is clicked.
- **Formatting Automation**:
  - Automatically formats the report (e.g., column auto-fit, borders, colorful headers).
- **Dynamic Data Handling**:
  - Loops through PivotTables to extract only filtered, visible data.

---

## **Results Achieved**

### **1. Unified Dataset**
- Successfully combined the **Customers**, **Orders**, and **Sales** datasets into a single structured table, enabling seamless analysis and visualization.

### **2. Key Insights Derived from the Dashboard**:
- **Sales Trends**:
  - Sales peaked during specific months, driven by **Technology** and **Furniture** categories.
  - **Office Supplies** showed consistent, moderate growth across the timeline.
- **Shipping Mode Analysis**:
  - **Standard Class** accounted for 60% of all orders, making it the most popular shipping mode.
- **State Performance**:
  - **California** had the highest number of orders among all states.
- **KPI Highlights**:
  - **Total Orders**: **9800**
  - **Total Sales**: **$2,257,623**
  - **Average Sales Per Order**: **$230.4**

### **3. Automated Report Generation**:
- Users can generate on-demand reports by applying slicers and clicking the **Generate Report** button.
- Reports are formatted professionally with section headers and colorful headers.
- Reports are exported as PDFs for easy sharing and distribution.


## **Conclusion**

The **Sales Performance Analysis Dashboard** is a comprehensive tool that enables stakeholders to:
- Track and analyze key sales metrics.
- Gain insights into customer segments, product performance, and shipping preferences.
- Generate professional, shareable reports at the click of a button.

By leveraging advanced Excel techniques and VBA automation, this dashboard significantly improves efficiency and enhances data-driven decision-making. The ability to export reports as PDFs ensures that insights are easily shareable across teams, making this solution an invaluable asset for any organization.

---

