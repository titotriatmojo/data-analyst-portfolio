## 🗂️ Data Source

- **Dataset:** Indonesia E-Commerce Sales & Shipping 2023–2025
- **Source:** [Kaggle](https://www.kaggle.com/datasets/bakitacos/indonesia-e-commerce-sales-and-shipping-20232025)
- **Format:** 24 monthly Excel files, consolidated into a single dataset
- **Size:** 49 Columns, 26,258 Rows (after merging)
- **Period:** December 2023 – November 2025
- **Fields:** order ID, order date, product category, quantity, shipping fee, discount, payment method, destination province, order status, cancellation reason

## 🛠️ Tools Used

- **Python (pandas)** for combining monthly files, data preparation
- **MySQL Workbench** for querying and analysis (JOIN, GROUP BY, window functions, CASE WHEN, CTEs)
- **Excel** for data validation and pivot cross-checks

## Step 1: Data Consolidation

The raw dataset was provided as 24 separate monthly Excel files (one per month, 
Dec 2023–Nov 2025), stored in Google Drive. Since all files share an identical 
column structure, they were programmatically combined into a single dataset 
using Python (pandas) rather than manually merging them in Excel, a more 
scalable and reproducible approach for datasets with many files.

**Process:**
1. Used `glob` to detect and count all `.xlsx` files in the raw data folder
2. Loaded each file into a dataframe and verified the file count matched 
   the expected 24 months
3. Concatenated all monthly dataframes into a single unified dataset 
   using `pd.concat()`
4. Exported the combined result as `combined_data.csv` for the next stage 
   (data cleaning)

**Code:**
```
import glob
import pandas as pd

#Count the number of files
path_pattern = "/content/drive/MyDrive/Dataset/id_e_commerce_sales_shipping_23-25/raw_public_data/*.xlsx"
files = glob.glob(path_pattern)

#Print the total number of files found
print(f"Number of files found: {len(files)}")

# Combine all into a single dataframe
df_list = [pd.read_excel(f) for f in files]
combined_df = pd.concat(df_list, ignore_index=True)

# Save the combined results
combined_df.to_csv("/content/drive/MyDrive/Dataset/id_e_commerce_sales_shipping_23-25/processed_public_data/combined_data.csv", index=False)

print("File successfully saved to your Google Drive!")
```

**Result:** 24 files successfully combined into a single dataset 
with 49 total rows, ready for cleaning and analysis.

---

## Step 2: Data Cleaning

Initial inspection revealed several data quality issues that needed to be 
addressed before analysis:

1. **Fully empty columns**  
   `SKU Induk`, `Nomor Referensi SKU`, and `Catatan` 
   contained no data across all 26,258 rows and were removed.

2. **Personally Identifiable Information (PII)**  
   Raw export (a standard Shopee Seller Center format) included customer-identifying fields such as 
   recipient name, phone number, and shipping address. These were dropped 
   entirely, both for privacy/ethical reasons and because they carry no 
   analytical value at the aggregate business-insight level this analysis 
   operates on cancellation rate, category performance, and regional trends 
   are all derived from grouped transaction attributes, not individual identity.

3. **Mixed granularity**  
   Dataset contained two levels of detail in one 
   table: order-level fields (populated for ±26,000 rows) and item/SKU-level 
   fields (populated for only ±1,470 rows). This analysis retains only 
   order-level fields to ensure consistent, complete data across the full 
   dataset.

4. **Data type correction**  
   Several numeric fields (e.g., total payment, 
   total weight) were stored as text due to currency formatting ("Rp", 
   thousand separators). These were converted to proper numeric types to 
   enable aggregation and calculation.

5. **Consistency check**  
   Cross-validated that the count of orders with a 
   "cancelled" status aligned with the count of rows containing a filled 
   cancellation reason, to confirm the cancellation-related fields could be 
   reliably used for Case Study 1.

**Result:** dataset reduced from 49 to 41 relevant columns, retaining 
all 26,258 order-level rows with corrected data types, ready for SQL analysis.

**Code:**
```
# 1. Drop columns with 100% missing values
fully_empty_cols = ['SKU Induk', 'Nomor Referensi SKU', 'Catatan']
combined_df = combined_df.drop(columns=fully_empty_cols)

# 2. Drop PII (Personally Identifiable Information) columns
# These are excluded for privacy reasons and have no analytical value
# at the aggregate business-insight level this project focuses on
pii_cols = ['Username (Pembeli)', 'Nama Penerima', 'No. Telepon', 
            'Alamat Pengiriman', 'Catatan dari Pembeli']
combined_df = combined_df.drop(columns=pii_cols)

# 3. Separate order-level data from incomplete SKU-level data
# Columns 0-22 are populated for nearly all 26,258 orders (order-level summary)
# Columns 23+ are only populated for about 1,470 rows (item/SKU-level detail)
# This analysis focuses on order-level data for consistency and completeness
order_level_cols = combined_df.columns[:22].tolist()  
df_orders = combined_df[order_level_cols].copy()

# 4. Fix data types — columns stored as text but should be numeric
money_cols = ['Total Berat', 'Total Pembayaran'] 
for col in money_cols:
    df_orders[col] = (
        df_orders[col]
        .astype(str)
        .str.replace('Rp', '', regex=False)
        .str.replace('.', '', regex=False)
        .str.replace(',', '.', regex=False)
        .str.strip()
    )
    df_orders[col] = pd.to_numeric(df_orders[col], errors='coerce')

# 5. Cross-check: does cancellation reason count match cancelled order count?
cancelled_orders = df_orders[df_orders['Status Pesanan'].str.contains('Batal', case=False, na=False)]
print(f"Orders with 'cancelled' status: {len(cancelled_orders)}")
print(f"Orders with a cancellation reason filled: {df_orders['Alasan Pembatalan'].notna().sum()}")

# Save the clean csv
combined_df.to_csv("/content/drive/MyDrive/Dataset/id_e_commerce_sales_shipping_23-25/clean_public_data/clean_data.csv", index=False)

print("File successfully saved")
```

## Step 3: Exploratory Data Analysis (SQL)

- **Input:** `cleaned_order_level_data.csv` imported into MySQL as the `sales_shipping` table (database: `e_commerce_23_25`)
- **Tool:** MySQL Workbench
- **Note:** Original column names (Bahasa Indonesia, from the raw Shopee export) were retained as-is rather than renamed, to preserve the dataset's original structure
- **Output:** full query file, with results exported as `.csv` per question

#### Loading Data into MySQL  
The cleaned dataset was loaded into MySQL using `LOAD DATA INFILE`, chosen 
over the GUI Import Wizard for faster performance with a dataset of this 
size (26,258 rows) and more explicit control over field/line terminators.

```sql
LOAD DATA LOCAL INFILE 'clean_data_final.csv'
INTO TABLE e_commerce_23_25.sales_shipping
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES;
```

#### A Note on NULL Values

Two distinct types of NULL appear in the query results below, with different implications:

- **`Alasan Pembatalan` (cancellation reason) NULL — expected by design.** 22,733 of 26,258 orders (86.6%) have no cancellation reason simply because they were never cancelled. This is not a data quality issue.
- **`order_month` NULL — a genuine data gap.** 2,434 orders (9.3% of total) are missing a value in `Waktu Pesanan Dibuat`, so they cannot be assigned to a month. Their cancellation rate (12.86%) is close to the platform-wide average (13.43%), suggesting this gap does not meaningfully bias the trend analysis. These rows were excluded from the month-by-month trend chart but are included in all other (non-time-based) queries.

---

#### Q1: Monthly Trend of Orders & Cancellation Rate

```sql
SELECT 
    DATE_FORMAT(STR_TO_DATE(`Waktu Pesanan Dibuat`, '%Y-%m-%d'), '%Y-%m') AS order_month,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) AS cancelled_orders,
    ROUND(SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS cancellation_rate_pct
FROM e_commerce_23_25.sales_shipping
GROUP BY order_month
ORDER BY order_month;
```

**Result (selected months, full data in `q1_monthly_trend.csv`):**
| order_month | total_orders | cancelled_orders | cancellation_rate_pct |
|---|---|---|---|
| 2023-12 | 591 | 127 | 21.49 |
| 2024-04 | 664 | 129 | 19.43 |
| 2024-07 | 1,644 | 208 | 12.65 |
| 2025-03 | 844 | 166 | 19.67 |
| 2025-08 | 1,075 | 99 | 9.21 |
| 2025-11 | 1,337 | 121 | 9.05 |

*(Note: 2024-12 and 2025-07 are absent from the dataset — a data gap worth flagging rather than a zero-order month.)*

**Insight:** Cancellation rate declined from **21.49% in December 2023 to 9.05% by November 2025**, nearly halving over the 24-month period, with the lowest rates concentrated in the final quarter of the dataset (Aug–Nov 2025, all under 10%). **So What:** whatever operational changes occurred over this period, platform policy, seller performance, or logistics improvements, appear to have meaningfully reduced cancellation. **Recommendation:** investigate what changed operationally between early 2024 and late 2025 (seller onboarding standards, payment options added, etc.) to formally document and replicate the practices driving this improvement.

---

#### Q2: Cancellation Rate by Product Category

```sql
SELECT 
    product_category,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) AS cancelled_orders,
    ROUND(SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS cancellation_rate_pct
FROM e_commerce_23_25.sales_shipping
GROUP BY product_category
ORDER BY cancellation_rate_pct DESC;
```

**Result (top cancellation categories with meaningful sample size):**
| product_category | total_orders | cancelled_orders | cancellation_rate_pct |
|---|---|---|---|
| Pengolah Bumbu / Sayur | 177 | 57 | 32.20 |
| Rak / Rak Serbaguna | 681 | 201 | 29.52 |
| Pisau / Alat Potong | 189 | 48 | 25.40 |
| Mangkok | 48 | 11 | 22.92 |

**Result (highest-volume / best-selling categories, for comparison):**
| product_category | total_orders | cancelled_orders | cancellation_rate_pct |
|---|---|---|---|
| Celengan | 6,999 | 770 | 11.00 |
| Mangkok Sambal / Saus | 4,389 | 459 | 10.46 |
| Aksesoris Pintu | 3,385 | 308 | 9.10 |

**Insight:** The platform's highest-volume categories (Celengan, Mangkok Sambal/Saus, Aksesoris Pintu) all perform **below the platform-wide cancellation rate (13.43%)**, at 9–11%. In contrast, mid-volume kitchen/tool categories, Rak/Rak Serbaguna (681 orders, 29.52%), Pengolah Bumbu/Sayur (177 orders, 32.20%), and Pisau/Alat Potong (189 orders, 25.40%), show cancellation rates more than double the platform average, despite large enough sample sizes to be meaningful rather than statistical noise. **So What:** the core best-selling categories are healthy; the problem is concentrated in a specific cluster of kitchen/organizational tool categories. **Recommendation:** investigate these categories specifically for common root causes, product description accuracy, sizing/expectation mismatches, or quality complaints, rather than applying a platform-wide fix.

---

#### Q3: Cancellation Rate by Payment Method

```sql
SELECT 
    `Metode Pembayaran`,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) AS cancelled_orders,
    ROUND(SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS cancellation_rate_pct
FROM e_commerce_23_25.sales_shipping
GROUP BY `Metode Pembayaran`
ORDER BY cancellation_rate_pct DESC;
```

**Result (major payment methods by volume):**
| Metode Pembayaran | total_orders | cancelled_orders | cancellation_rate_pct |
|---|---|---|---|
| COD (Bayar di Tempat) | 13,153 | 1,728 | 13.14 |
| Saldo ShopeePay | 5,246 | 480 | 9.15 |
| Online Payment | 4,350 | 800 | 18.39 |
| SPayLater | 2,197 | 289 | 13.15 |
| SeaBank Bayar Instan | 887 | 149 | 16.80 |

**Insight:** Among the two largest digital payment methods with comparable volume, **Online Payment shows a 18.39% cancellation rate versus just 9.15% for Saldo ShopeePay**, nearly double, despite both being "already-paid" digital methods. **So What:** this gap points to friction in the online payment gateway itself (failed transactions, pending payment timeouts) rather than a buyer-intent issue, since ShopeePay balance payments are instant and rarely fail. **Recommendation:** audit the Online Payment gateway's failure/timeout rate as a likely direct contributor to unnecessary cancellations.

---

#### Q4: Revenue and Order Volume by Province

```sql
SELECT 
    Provinsi,
    COUNT(*) AS total_orders,
    ROUND(SUM(`Total Pembayaran`), 0) AS total_revenue,
    ROUND(AVG(`Total Pembayaran`), 0) AS avg_order_value,
    SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) AS cancelled_orders,
    ROUND(SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS cancellation_rate_pct
FROM e_commerce_23_25.sales_shipping
GROUP BY Provinsi
ORDER BY total_revenue DESC;
```

**Result (top revenue provinces, Java):**
| Provinsi | total_orders | total_revenue | avg_order_value | cancellation_rate_pct |
|---|---|---|---|---|
| JAWA BARAT | 8,141 | 317,293 | 39 | 11.14 |
| BANTEN | 4,888 | 241,771 | 49 | 12.44 |
| DKI JAKARTA | 3,729 | 187,337 | 50 | 12.15 |
| JAWA TENGAH | 1,789 | 81,679 | 46 | 12.41 |

**Result (outer-island provinces with elevated cancellation):**
| Provinsi | total_orders | cancellation_rate_pct |
|---|---|---|
| SULAWESI UTARA | 50 | 32.00 |
| MALUKU | 26 | 30.77 |
| SULAWESI TENGAH | 73 | 30.14 |
| GORONTALO | 77 | 28.57 |
| KEPULAUAN RIAU | 102 | 25.49 |
| KALIMANTAN TIMUR | 169 | 25.44 |

**Insight:** Java-based provinces, which account for the large majority of both order volume and revenue, consistently show cancellation rates **below the platform average (11–12.4% vs. 13.43%)**. Outer-island provinces, while much lower in volume, show cancellation rates **2–2.5x higher** (25–32%). **So What:** this geographic pattern likely connects to the shipping-related cancellation reasons found in Q6 ("Pengiriman gagal", "Penjual gagal mengirimkan pesanan tepat waktu"), longer shipping distances to eastern Indonesia plausibly increase delivery failure and timeout-driven cancellations. **Recommendation:** review shipping SLA and courier options specifically for outer-island destinations rather than treating cancellation as a uniform national problem.

---

#### Q5: Discount Tier vs. Order Volume & Cancellation

```sql
SELECT 
    CASE 
        WHEN `Total Diskon` = 0 THEN '0 - No Discount'
        WHEN `Total Diskon` BETWEEN 1 AND 20000 THEN '1 - Low (Rp1-20K)'
        WHEN `Total Diskon` BETWEEN 20001 AND 50000 THEN '2 - Medium (Rp20K-50K)'
        ELSE '3 - High (>Rp50K)'
    END AS discount_tier,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) AS cancelled_orders,
    ROUND(SUM(CASE WHEN `Status Pesanan` LIKE '%Batal%' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS cancellation_rate_pct
FROM e_commerce_23_25.sales_shipping
GROUP BY discount_tier
ORDER BY discount_tier;
```

**Result:**
| discount_tier | total_orders | cancelled_orders | cancellation_rate_pct |
|---|---|---|---|
| 0 - No Discount | 26,099 | 3,501 | 13.41 |
| 1 - Low (Rp1-20K) | 159 | 24 | 15.09 |

**Insight / Limitation:** 99.4% of all orders (26,099 of 26,258) carry no discount at all, with no orders falling into the medium or high discount tiers. This means the dataset does not contain enough variation in discounting to draw a reliable conclusion about its relationship with cancellation. **Rather than forcing an insight from this result, it is reported here as a data limitation**: discounting does not appear to be actively used as a sales lever on this store, which is itself worth noting for a business audience, but doesn't support a cause-effect claim about discount size and cancellation behavior.

---

#### Q6: Top Cancellation Reasons

```sql
SELECT 
    `Alasan Pembatalan`,
    COUNT(*) AS total_cases,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM e_commerce_23_25.sales_shipping WHERE `Alasan Pembatalan` IS NOT NULL), 2) AS pct_of_cancellations
FROM e_commerce_23_25.sales_shipping
WHERE `Alasan Pembatalan` IS NOT NULL
GROUP BY `Alasan Pembatalan`
ORDER BY total_cases DESC;
```

**Result (top reasons, grouped by initiator):**
| Alasan Pembatalan | total_cases | pct_of_cancellations |
|---|---|---|
| Dibatalkan oleh Pembeli. Alasan: Lainnya/ berubah pikiran | 729 | 20.68%* |
| Dibatalkan oleh Pembeli. Alasan: Ubah Pesanan yang Ada | 704 | 19.97%* |
| Dibatalkan secara otomatis oleh sistem. Alasan: Pesanan belum dibayar | 585 | 16.60%* |
| Dibatalkan oleh Pembeli. Alasan: Need to change delivery address | 400 | 11.35%* |
| Dibatalkan secara otomatis oleh sistem. Alasan: Pengiriman gagal | 297 | 8.42%* |

*percentages recalculated as share of the 3,525 total cancellations (not share of all orders)*

**Summary by cancellation initiator (of 3,525 total cancellations):**
| Initiator | Cases | Share |
|---|---|---|
| Buyer-initiated | 2,363 | 67.0% |
| System auto-cancelled | 1,147 | 32.5% |
| Seller-initiated | 15 | 0.4% |

**Insight:** Two-thirds of all cancellations are buyer-initiated, and within that group, the dominant driver is not dissatisfaction with the product, it's the need to **modify an existing order**. Combining "Ubah Pesanan yang Ada," "Need to change delivery address," "Perlu mengubah alamat pengiriman," "Perlu mengubah pesanan," and "Perlu mengubah Voucher" accounts for **1,443 cancellations = 41% of all cancellations platform-wide**. Separately, 585 cancellations (16.6%) are system auto-cancellations for unpaid orders, pointing to checkout/payment-completion friction. **So What:** the largest share of cancellations isn't a product or pricing problem, it's the absence of a post-checkout order-editing capability. Buyers who simply need to fix an address or add an item are forced to cancel and re-purchase entirely. **Recommendation:** this is the strongest, most actionable finding in this analysis and directly informs the proposed solution in the BRD : a **post-checkout order editing feature** (address changes, item modification within a limited time window before fulfillment begins) could plausibly address a significant share of the 41% "order modification" cancellation cluster.

---

### Summary of Key Findings

1. Cancellation rate nearly halved over 24 months (21.5% → 9.1%), suggesting positive but undocumented operational improvement.
2. A small cluster of kitchen/tool categories show 2x+ the platform's average cancellation rate: a targeted, not platform-wide, problem.
3. Online Payment shows double the cancellation rate of ShopeePay balance payments: likely a payment gateway reliability issue.
4. Outer-island provinces cancel 2–2.5x more often than Java, plausibly linked to shipping/delivery failure.
5. **41% of all cancellations stem from buyers needing to modify an existing order** : the single largest, most actionable finding, and the foundation for this project's BRD.

