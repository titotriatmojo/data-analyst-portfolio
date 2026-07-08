## 🗂️ Data Source

- **Dataset:** Indonesia E-Commerce Sales & Shipping 2023–2025
- **Source:** [Kaggle](https://www.kaggle.com/datasets/bakitacos/indonesia-e-commerce-sales-and-shipping-20232025)
- **Format:** 24 monthly Excel files, consolidated into a single dataset
- **Size:** 49 Rows, 26258 Columns (after merging)
- **Period:** December 2023 – November 2025
- **Fields:** order ID, order date, product category, quantity, shipping fee, discount, payment method, destination province, order status, cancellation reason

## 🛠️ Tools Used

- **Python (pandas)** for combining monthly files, data preparation
- **MySQL Workbench** for querying and analysis (JOIN, GROUP BY, window functions, CASE WHEN, CTEs)
- **Excel** for data validation and pivot cross-checks

## Data Consolidation

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
