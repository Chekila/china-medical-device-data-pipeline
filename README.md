# China Medical Device UDI Data Cleaning and EDA

## 1. Project Introduction

This project cleans and analyzes the Unique Device Identification (UDI) data published by China's National Medical Products Administration (NMPA).

The raw data is stored in XML format, distributed across 1,136 compressed file shards, containing approximately 5.67 million medical device records. The goal of this project is to parse, clean, and standardize the raw XML data, then load it into a SQLite database to form a structured, queryable dataset that supports downstream exploratory data analysis.

The cleaning workflow includes XML parsing, English column name standardization, date format unification, cross-file deduplication, and SQLite export. The exploratory data analysis covers dataset overview, time trends, product structure, and registrant distribution analysis.

---

## 2. Data Source

The raw data was downloaded from the official NMPA UDI database download page.

| Item | Description |
|---|---|
| Source organization | National Medical Products Administration (NMPA), China |
| Source URL | `https://udi.nmpa.gov.cn/download.html` |
| Source file | `UDID_FULL_RELEASE_20260501.zip` |
| Data format | ZIP archive containing 1,136 XML files |
| Date accessed | `2026-05-19` |

The raw dataset contains Chinese medical device UDI registration records. Fields include product identification codes, product names, model specifications, registration certificate numbers, registrant information, coding standards, label information, storage conditions, version status, and more.

---

## 3. Data Extraction

The raw data was downloaded as a ZIP archive containing 1,136 XML files. Python's `zipfile` library was used to read the archive contents directly without extracting files to local disk.

Each XML file follows the same structure: the root node contains a `<devices>` node, and each `<device>` child node corresponds to one medical device record. Most fields are plain text nodes, while contact information (`<contactList>`) is a nested structure that requires separate handling.

Example:

```python
import zipfile
import xml.etree.ElementTree as ET

ZIP_PATH = "../data/UDID_FULL_RELEASE_20260501.zip"

with zipfile.ZipFile(ZIP_PATH) as zf:
    xml_names = sorted([n for n in zf.namelist() if n.endswith('.xml')])
    with zf.open(xml_names[0]) as f:
        tree = ET.parse(f)
        root = tree.getroot()
```

A small number of XML files contain invalid characters that cause the standard parser to fail. To handle this, `lxml`'s recovery mode (`recover=True`) was introduced as a fallback parser, ensuring all 1,136 files could be successfully read.

---

## 4. Cleaning Workflow

### 4.1 Imports & Configuration

Python libraries are loaded (pandas, xml, sqlite3, lxml, tqdm, etc.) and two core paths are defined: the raw data ZIP file path and the SQLite database output path.

Two key mappings are configured:

- `column_mapping`: Translates the original XML field names (abbreviated Chinese pinyin, e.g. `zxxsdycpbs`) into English snake_case format (e.g. `udi_code`), covering all fields.
- `date_columns`: Lists the date fields to be reformatted into ISO 8601 format (`YYYY-MM-DD`).

The deduplication key is set to `device_record_key`, which is the unique identifier automatically assigned by the NMPA system to each UDI record. Since the raw data is split across 1,136 XML shards during download, the same record may appear in multiple shards. After all data is written to SQLite, `device_record_key` is used to identify and remove these cross-file duplicates.

---

### 4.2 XML Parse Function

The `parse_xml_to_df()` function is defined to parse a single XML file's root node into a pandas DataFrame, where each `<device>` node becomes one row.

Most fields are extracted directly. Contact information (`<contactList>`) is a nested structure that requires separate handling — fax, email, and phone are extracted as individual columns. All blank strings are converted to `None` at this step, which become `NULL` values when written to the database.

---

### 4.3 Stream All XMLs into SQLite

The `devices` table is dropped and recreated for a clean run. WAL mode is enabled to improve write performance. Each of the 1,136 XML files is then processed in sequence:

1. Attempt to parse using the standard `xml.etree.ElementTree` parser
2. If parsing fails due to invalid characters, automatically fall back to `lxml` recovery mode (`recover=True`) to repair and re-parse, ensuring no file is skipped entirely
3. Call the parse function to generate a DataFrame
4. Rename columns to English using `column_mapping`
5. Add provenance fields (`source_file` records which XML file the row came from; `source_url` records the original download URL)
6. Convert date fields to `YYYY-MM-DD` format
7. Append to SQLite in batches — no deduplication at this stage

---

### 4.4 Global Cross-file Deduplication

After all files are written, a single global deduplication pass is performed at the SQLite level:

1. A `duplicate_flag` column is added with a default value of 0
2. For records sharing the same `device_record_key`, the earliest-written row (lowest `rowid`) is kept and all others are flagged with `duplicate_flag = 1`
3. All rows where `duplicate_flag = 1` are deleted

Deduplication results:

| Stage | Row Count |
|---|---:|
| After initial load (before deduplication) | 5,674,158 |
| Duplicate rows removed | 445 |
| After deduplication (final) | 5,673,713 |

The 445 duplicate records account for just 0.008% of the total, indicating minimal overlap between shards and high source data quality.

---

### 4.5 Validation

A final validation step confirms the total row count, verifies that remaining duplicates equal 0, checks the database file size, encoding (UTF-8), and date format (ISO 8601), and previews the first 5 records to confirm all fields were written correctly.

---

## 5. Database Schema

| Column Name | Description | Example Value | Data Type |
|---|---|---|---|
| `udi_code` | UDI code of the minimum sales unit | 06936260855798 | object |
| `coding_system` | UDI coding standard used | GS1 | object |
| `udi_publish_date` | Date the UDI code was published to the database | 2025-08-22 | object |
| `units_per_package` | Number of use units contained in the minimum sales unit | 1 | object |
| `unit_of_use_code` | UDI code of the individual use unit | None | object |
| `barcode_type` | Type of barcode on the label | 二维码 | object |
| `consistent_with_registration` | Whether the label is consistent with the registered sample | 是 | object |
| `registration_product_code` | Product code as shown on the registration certificate | None | object |
| `has_companion_device_code` | Whether a companion device code exists | 否 | object |
| `companion_code_verified` | Whether the companion device code matches the UDI code | None | object |
| `companion_device_code` | UDI code of the companion device | None | object |
| `product_generic_name` | Generic/common name of the product | 聚醚醚酮带线锚钉 | object |
| `trade_name` | Trade or brand name of the product | None | object |
| `model_specification` | Model number or specification of the product | SM400-2500001 | object |
| `is_kit_or_set` | Whether the product is a kit or set | 否 | object |
| `product_description` | Detailed description of the product composition and materials | 产品由锚钉、缝合线... | object |
| `product_catalog_number` | Catalog or part number assigned by the manufacturer | None | object |
| `old_classification_code` | Classification code under the previous regulatory system | None | object |
| `device_category` | Top-level device category | 器械 | object |
| `classification_code` | NMPA product classification code | 13-02-01 | object |
| `unified_social_credit_code` | Unified social credit code of the registrant company | 9133020158396090XB | object |
| `registration_or_filing_number` | Registration certificate number or filing reference number | 国械注准20243131971 | object |
| `registrant_name_cn` | Chinese name of the registrant or filing holder | 宁波华科润生物科技有限公司 | object |
| `registrant_name_en` | English name of the registrant or filing holder | None | object |
| `medical_insurance_code` | National medical insurance classification code | C030102076020020... | object |
| `product_type` | Product type category | 耗材 | object |
| `mr_safety_info` | MRI safety information status | 说明书或标签上面不包含MR安全性信息 | object |
| `is_single_use` | Whether the device is intended for single use only | 是 | object |
| `max_reuse_times` | Maximum number of times the device can be reused | None | object |
| `is_sterile_package` | Whether the device is supplied in sterile packaging | 是 | object |
| `requires_sterilization_before_use` | Whether sterilization is required before use | 否 | object |
| `sterilization_method` | Sterilization method applied | None | object |
| `additional_info_url` | URL linking to additional product information | None | object |
| `special_date` | Special date associated with the product record | None | object |
| `label_includes_lot_number` | Whether the label includes a lot/batch number | 是 | object |
| `label_includes_serial_number` | Whether the label includes a serial number | 是 | object |
| `label_includes_manufacture_date` | Whether the label includes the manufacture date | 否 | object |
| `label_includes_expiry_date` | Whether the label includes the expiry date | 是 | object |
| `special_storage_conditions` | Special storage or handling conditions required | None | object |
| `special_storage_notes` | Additional notes on special storage requirements | None | object |
| `device_record_key` | System-assigned unique identifier for each UDI record | 06936260855798202... | object |
| `version_number` | Version number of this record | 1 | object |
| `version_date` | Date of the most recent version update | 2025-08-22 | object |
| `version_status` | Status of the current record version | 新增 | object |
| `correction_count` | Number of corrections applied to this record | 0 | object |
| `correction_remark` | Description of the correction made | None | object |
| `correction_date` | Date when the correction was applied | None | object |
| `contact_fax` | Fax number of the company contact person | 0574-63729786 | object |
| `contact_email` | Email address of the company contact person | gf@hicren.com | object |
| `contact_phone` | Phone number of the company contact person | 0574-63729797 | object |
| `packingList` | Packaging information | None | object |
| `storageList` | Storage information | None | object |
| `clinicalList` | Clinical information | None | object |
| `source_file` | Source XML filename from which the record was extracted | UDID_FULL_DOWNLOAD_PART1000... | object |
| `source_url` | Original download URL of the source data | https://udi.nmpa.gov.cn/download.html | object |
| `duplicate_flag` | Flag indicating whether the record was identified as a duplicate | 0 | int64 |

---

## 6. EDA Workflow

### 6.1 Dataset Overview

A deduplication summary table is displayed showing row counts before and after deduplication. Missing value ratios are calculated across all columns based on 50,000 randomly sampled rows and sorted by missing ratio. Database metadata is also reported: file size 3.99 GB, column count, source file count (1,136), and total record count (5,673,713).

---

### 6.2 Time Trend Analysis

UDI record counts are aggregated by year using the `udi_publish_date` field and visualized as a line chart. 

The data shows that record counts in China's UDI database grew rapidly from 2019, peaking in 2022 at 1,481,655 records. Counts have since stabilized at roughly 800,000 to 1,000,000 records per year. The 2026 count is incomplete as the data snapshot was taken on May 1, 2026.

---

### 6.3 Product Structure Analysis

The number of unique values in `product_generic_name` is counted (approximately 100,000 distinct product types). The Top 20 most frequently registered product types are mapped to English and visualized as a horizontal bar chart. 

Soft hydrophilic contact lenses lead by a wide margin at 1,251,359 records, followed by orthopaedic implants (bone pins, bone plates, bone screws, intramedullary nails) and single-use consumables (infusion sets, syringes).

---

### 6.4 Registrant & Regulatory Analysis

**6.4.1 Top 15 Registrants:** The `registrant_name_cn` field is aggregated and the top 15 registrants are mapped to English, then visualized as a horizontal bar chart and pie chart. 

Jilin Ruierkang Optical Technology Co. leads with 379,507 records. The top 15 registrants together account for approximately 27% of all records.

**6.4.2 Approval Type Distribution:** Approval types are extracted from `registration_or_filing_number` using keyword matching and visualized as a pie chart. 

Domestic registration (注准) accounts for 82.3%, imported registration (注进) for 10.1%, filing (械备) for 4.7%, and the remainder consists of records with non-standard number formats.

**6.4.3 Top 10 Approving Provinces:** The first character of the registration number is extracted as a province code and mapped to province names in English, then visualized as a bar chart and pie chart. 

National-level approvals (NMPA direct) dominate at 4,311,508 records (80.4%), with Jiangsu, Guangdong, and Zhejiang leading among provincial authorities.

---

## 7. Known Limitations, Edge Cases, and Quality Checks

### Known Limitations

- All original field names are abbreviated Chinese pinyin (e.g. `zxxsdycpbs`), which are not human-readable. Column name translation relies on manual interpretation and may contain minor inaccuracies.
- Fields such as `product_generic_name` and `registrant_name_cn` contain Chinese text. English translations in the EDA are manually mapped and only cover the top-frequency values; all other records retain their original Chinese text.
- Several fields (e.g. `contact_fax`, `contact_email`, `contact_phone`, `packingList`, `storageList`, `clinicalList`) are empty in the vast majority of records and have limited analytical value.
- The database is approximately 4 GB. Loading the full dataset into memory is not feasible. Missing value analysis in the EDA is based on a random sample of 50,000 rows and should be treated as an approximation rather than an exact count.

### Edge Cases

- A small number of XML files in the ZIP archive contain invalid characters that cause the standard `xml.etree.ElementTree` parser to fail. `lxml` recovery mode is used as a fallback to preserve data from these files, though recovery mode may silently drop field values with malformed encoding.
- Records where `device_record_key` is `NULL` are excluded from the deduplication logic and retained as-is in the database.
- Approval type classification relies on keyword matching against registration number prefixes. Approximately 2.8% of records do not match any known prefix pattern and are classified as `Other / Unknown`.

### Quality Checks Performed

- Verified total row count before and after deduplication to confirm the deduplication logic executed correctly
- Confirmed that remaining duplicate count equals 0 after deduplication
- Reviewed missing value ratios across all columns using random sampling
- Verified database encoding is UTF-8 and all date fields follow ISO 8601 format (`YYYY-MM-DD`)
- Previewed the first 5 records to manually confirm key fields were written correctly

---

## 8. Reproduction Steps

### Step 1: Download Raw Data

Go to the NMPA UDI database download page and download the full data archive:

```
https://udi.nmpa.gov.cn/download.html
```

Expected file:

```
UDID_FULL_RELEASE_20260501.zip
```

Place the file in the `data/` folder of the project directory.

---

### Step 2: Set Up the Environment

This project uses `uv` to manage dependencies. Instructions will be added once the `uv` environment setup is complete.

---

### Step 3: Run the Cleaning Workflow

Open and run the cleaning notebook:

```
china_cleaning.ipynb
```

This notebook takes `UDID_FULL_RELEASE_20260501.zip` as input and performs XML parsing, column name standardization, date format unification, and cross-file deduplication, then exports the final SQLite database.

Expected output:

```
chinese_medical_devices.db
```

---

### Step 4: Run the EDA Workflow

Open and run the EDA notebook:

```
china_eda.ipynb
```

This notebook takes `chinese_medical_devices.db` as input and performs dataset overview, time trend analysis, product structure analysis, registrant analysis, and UDI coding system analysis. All visualizations are saved to the `outputs/` folder.

---

### Step 5: Review Outputs

Main outputs include:

- SQLite database file
- UDI publish year trend line chart
- Top 20 product types horizontal bar chart
- Top 15 registrants bar chart and pie chart
- Approval type distribution pie chart
- Top 10 approving provinces bar chart and pie chart
- UDI coding system distribution bar chart
