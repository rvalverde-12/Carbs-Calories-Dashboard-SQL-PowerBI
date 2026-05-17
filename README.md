# Starbucks Menu Analysis: Carbs & Calories Dashboard
A data analysis project to find out what's actually inside Starbucks drinks. The main goal here was to stand up a local PostgreSQL database, clean up the raw data, and build a Power BI dashboard that shows the real sugar and calorie outliers—without letting 95 plain coffees and waters (which have 0 carbs) ruin the menu averages.


## The Problem & The Solution
If you take the average of the entire Starbucks menu, the number looks super low because there are tons of plain items (like black coffee or green tea) with zero carbs. That skews the data downstream. 

To fix this, I loaded the data into PostgreSQL, wrote queries to calculate a **true baseline (27.8g of carbs)** that only includes sweetened drinks, and flagged dietary tags like Keto. Now, when you use the dashboard and toggle the **Keto Friendly** filter, the visuals react instantly and show how far above or below a drink is from the real average.

### What's inside the dashboard:
* **Quick Stats (KPIs):** Total available drinks, average calories, and our 27.8g carb benchmark.
* **Scatter Plot:** Shows the relationship between carbs and calories, split into 4 color-coded tiers.
* **Diverging Bar Chart:** A clean list of the top drinks. Bars to the right mean a heavy sugar impact; bars to the left mean carb savings.


## Tech Stack & Connection
* **Database Engine:** PostgreSQL
* **BI & Data Modeling:** Power BI Desktop
* **Database Connection:** Connected Power BI to the local PostgreSQL instance using the native database connector (entering server credentials and pulling the clean SQL views directly into the data model).


## The SQL Codes (PostgreSQL)

### 1. Database Setup & Table Creation
First, we set up the PostgreSQL database and created the core tables to hold the raw Starbucks menu data and the categorical flags.

```sql
-- Creating the main drinks table
CREATE TABLE starbucks_drinks (
    drink_id SERIAL PRIMARY KEY,
    "Drink Name" VARCHAR(255),
    "Calories" INT,
    "Carb. (g)" INT,
    "Protein" INT,
    "Sodium" INT
);
```
```sql
-- Creating the categories table for dietary flags
CREATE TABLE categories (
    drink_id INT REFERENCES starbucks_drinks(drink_id),
    "Drink Name" VARCHAR(255),
    is_keto_friendly VARCHAR(3),
    is_high_protein VARCHAR(3),
    is_low_sodium VARCHAR(3)
);
```
### 2. Flagging Categories (Keto, Protein, Sodium)

```sql
INSERT INTO categories (drink_id, "Drink Name", is_keto_friendly, is_high_protein, is_low_sodium)
SELECT 
    drink_id,
    "Drink Name",
    CASE WHEN "Carb. (g)" <= 10 THEN 'Yes' ELSE 'No' END AS is_keto_friendly,
    CASE WHEN "Protein" >= 10 THEN 'Yes' ELSE 'No' END AS is_high_protein,
    CASE WHEN "Sodium" <= 140 THEN 'Yes' ELSE 'No' END AS is_low_sodium
FROM 
    starbucks_drinks;
```

### 3. Fixing the Average

This query calculates the real average of the menu by ignoring the 95 zero-carb drinks. This gives us a solid baseline to see which prepared drinks actually have too much sugar.

```sql
SELECT 
    "Drink Name",
    "Carb. (g)",
    "Calories",
    AVG("Carb. (g)") OVER() AS true_carb_baseline,
    "Carb. (g)" - AVG("Carb. (g)") OVER() AS variance_from_baseline
FROM 
    starbucks_drinks
WHERE 
    "Carb. (g)" IS NOT NULL 
    AND "Carb. (g)" > 0;
```

### 4. Grouping Drinks by Calories (NTILE)

To make the scatter plot easier to read, I used NTILE(4) to split the menu into 4 even tiers based on calories, sorted from lowest to highest.

```SQL
SELECT 
    "Drink Name",
    "Calories",
    "Carb. (g)",
    NTILE(4) OVER(ORDER BY "Calories" ASC) AS caloric_tier
FROM 
    starbucks_drinks
WHERE 
    "Calories" IS NOT NULL;
```

## Power BI Setup & Data Model

#### The Relationship Fix
The visuals weren't talking to each other at first because of a one-way filter. To fix it, I updated the relationship line in the Model View:

* Tables Connected: Categories Many to one Carb_Variance
* Matching Column: Drink Name
* Cross filter direction: Set to Both. This fixed the filter flow so clicking the Keto slicer updates everything instantly across both PostgreSQL tables.

### Fine-Tuning the UI & UX:
* **Clean & Centered KPIs:** I centered the metrics and turned off the automatic text labels underneath them. **Why:** Power BI puts the column name by default (like "First Drink Name"), which looks messy. Putting a clean title box above them looks way sharper.
* **Modern Floating Cards:** Turned off the default black borders on the KPI cards and added soft, light-gray shadows instead. **Why:** Thick borders look outdated. Shadows give the dashboard a clean, modern "floating app" aesthetic.
* **Fixed Long Text Clipping:** Bumped the Y-axis padding width to **35%**. **Why:** Starbucks has super long drink names. The default settings cut them off (e.g., *"Teavana Shaken Ice..."*), making it impossible to read. Now you can see the full product names clearly.

  <p align="center">
  <img src="Dashboard Starbucks - Keto Friendly.jpg" width="500" alt="Market Gap Analysis: Health Score vs. Consumer Rating"/>
</p>

