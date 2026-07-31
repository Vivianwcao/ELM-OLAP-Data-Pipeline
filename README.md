# Automated Well Abandonment OLAP Data Pipeline
## Serverless Data Lakehouse Architecture for Operational Analytics

### Tech Stack & Tools
* **Compute & Infrastructure:** AWS Lambda (Python 3.13), AWS SAM (YAML), Docker (`sam build --use-container`), AWS AppFlow, Amazon EventBridge, Amazon SQS

* **Storage & Data Lakehouse:** Amazon S3 (Bronze, Silver, Gold Layers), AWS Glue Data Catalog, Amazon Athena

* **Core Languages & Libraries:** Python 3.13, Pandas, `openpyxl`, PyArrow, Boto3

* **AI & Operational Enrichment:** Claude Haiku on AWS Bedrock

* **Analytics & Visualization:** Amazon QuickSight (SPICE Engine)

---

## Background & Business Problem

ELM manages well abandonment projects in Alberta for several major oil and gas buyers, including companies like Harvest Operations. For every job, ELM coordinates field contractors, tracks daily operational costs, and generates final invoice packages. Over four years of operations, field operators across various job sites filled out more than 750 isolated Excel workbooks, all stored in a shared Google Drive folder.

Extracting value from this dataset presented three core challenges:

1. **Visual form layouts instead of standard spreadsheets:** The workbooks were not built as standard tabular spreadsheets. Instead, they were designed like multi page forms complete with custom formatting, logos, and visual layouts. Extracting information required precise, cell level coordinate parsing.

2. **Dirty and inconsistent human entries:** Because the files were filled out by hand by different operators across different sites over four years, the data entries were messy, inconsistent, and full of small human variations. Important information could not simply be pulled and used directly for analysis.

3. **Zero cross job portfolio visibility:** A project manager could open a single file to check one well, but answering bigger questions across the entire portfolio of 750 abandoned wells was extremely difficult:

* Which wells exceeded their Authorization for Expenditure budget, and by how much?

* What was the total spend per contractor across all jobs?

* How many wells encountered regulatory issues like failed pressure tests, leaky casing, or parted tubing?

* How many plugs were set across the portfolio, at what depths, and did they pass compliance checks?

* Which wells were completed versus those with outstanding work?

To answer these questions and perform proper OLAP analysis across this dataset, opening 750 spreadsheets one by one by hand was functionally impossible. A scalable analysis process was therefore required to consolidate the data into reports comparing actual costs with original budgets and providing contractor-level breakdowns. These reports allow ELM to evaluate performance across wells, identify cost and operational trends, and make better business decisions that affect job profitability and future planning. They also provide buyers such as Harvest Operations with transparent project reporting, helping demonstrate the value of ELM's services and justify its management fee on each job.


---

## Solution & Results

I designed and built an event driven Medallion Data Lakehouse on AWS using Python 3.13, Docker, and AWS SAM defined in a unified SAM template YAML file. The pipeline automatically ingests raw workbooks from Google Drive, parses visual spreadsheet layouts using `openpyxl`, extracts unstructured engineering context using Claude Haiku on Bedrock, and delivers clean datasets to Athena and QuickSight.

### Key Results

* **Complete Portfolio Visibility:** Transformed 750 isolated Excel files into a centralized OLAP data lakehouse, enabling instant portfolio wide cost and regulatory reporting.

* **Automated Budget Reconciliation:** Enabled precise tracking of actual spend versus original authorization budgets across all contractors, protecting ELM's management fee justification.

* **AI Powered Compliance Extraction:** Extracted critical downhole pressure test outcomes and regulatory numbers from unstructured daily field notes using Claude Haiku on AWS Bedrock.

* **Low Cost Interactive Analytics:** Isolated dashboard user queries from raw data files by powering Amazon QuickSight dashboards through the SPICE in memory calculation engine.

---

## Architecture & Data Flow

The pipeline receives workbooks from Google Drive via AppFlow, processes raw files through serverless parsing layers, enriches unstructured text using Bedrock, and loads clean Parquet datasets into Athena for QuickSight visualization.

<img width="2720" height="2640" alt="well_abandonment_pipeline_v4" src="https://github.com/user-attachments/assets/d15550a8-3c47-4521-9726-95a9ce8b0820" />

### Medallion Data Processing Stages

* **Bronze Layer (`S3 raw/`):** Raw, multi tab `.xlsm` workbooks bulk ingested from Google Drive using AWS AppFlow.

* **Silver Layer (`S3 extracted/` and `S3 clean/`):**
  * **Extraction Sub Stage (`S3 extracted/`):** Lambda 1 parses each `.xlsm` file and generates one single JSON file per workbook, structured with 5 distinct JSON objects corresponding to the 5 workbook tabs.
  * **Clean Parquet Sub Stage (`S3 clean/`):** Lambda 2 runs LLM enrichment on free text notes, enforces schema typing, and writes the output into 5 separate Parquet subfolders (one per tab domain). AWS Glue crawls these Parquet subfolders to build 5 base Athena tables.

* **Gold Layer (Curated Athena Views):** The Gold layer consists of 5 curated SQL views in Athena that join, aggregate, and normalize data across the 5 base Parquet tables. These business ready views serve as the source layer for downstream BI reporting in Amazon QuickSight.

---

## Engineering Challenges & Solutions

### 1. The files are visual forms rather than structured spreadsheets
* **The Challenge:** Field supervisors used Excel as a printed form layout tool rather than a database. Workbooks were built with merged cells, hand aligned labels, and tables that started at different row offsets depending on how much notes the previous supervisor typed. With six tabs per file and subtle undocumented variations across 750 files from different contractors, standard parsing with `pd.read_excel()` failed on file one.

* **The Solution:** Instead of assuming fixed row coordinates, I built an anchor based cell detection parser using `openpyxl`. The script scans for known label strings and reads neighboring cells at relative offsets to where the label was found. Section boundaries for charge tables and repeating line items are detected by keyword terminators rather than row counts. This handles layout shifts across files without breaking silently.

*Main Data & Daily Summary Sheet showing visual layout and repeating line items*
<p float="left">
  <img alt="Main Data Sheet" src="https://github.com/user-attachments/assets/41860b44-6c20-4cfb-a044-5db5d959c72e" width="49%" />
  <img alt="Daily Summary Sheet" src="https://github.com/user-attachments/assets/e5d03409-056d-49fc-8c3b-0f1e231239e0"" width="49%" />
</p>

### 2. Extracting scattered metrics from the Summary of Changes tab
* **The Challenge:** This tab is a regulatory summary filled out by the engineer after job completion, not the field supervisor on site. It contains the most reliable structured data in the whole workbook, including Kelly Bushing elevation, Ground Level, Total Depth, Plug Back Total Depth, and three sub tables listing downhole installations, cement squeezes, and previous perforations. None of it sits in a clean table. It is a visual form spread across a fixed region of the sheet with labels and values scattered across merged cells. Header fields each required individual anchor searches within tightly bounded row and column slices because they sit in different column ranges across adjacent rows.

* **The Solution:** The cement squeeze table presented another hurdle. Engineers typed volumes as `"3.0"`, `"3.0m3"`, `"22.7 Tonne / 17m3"`, or just `"3"` with no unit. I wrote a dedicated `parse_volume()` function to handle all these string variants and split them into separate `volume_m3` and `volume_tonne` float fields so the data could actually be queried downstream.

*Summary of Changes Tab*
<p float="left">
  <img alt="Summary of Changes Sheet1" src="https://github.com/user-attachments/assets/45568571-7ba3-4f63-a2b0-9b43d04a96e8" width="49%" />
  <img alt="Summary of Changes Sheet2" src="https://github.com/user-attachments/assets/2d08d985-9741-4ba4-97b4-4799c75ddd70" width="49%" />
</p>

### 3. Classifying Job & daily supervisor notes with Claude Haiku on Bedrock
* **The Challenge:** The Main sheet and every daily sheet contains a free text section written by the field supervisor. These paragraphs hold the most operationally valuable details in the file, such as plug depths, pressure test outcomes, cement volumes, and equipment failures. Regex can pull numbers, but it cannot determine engineering context. For example, a supervisor note might say the well was pressured to 13 MPa to shear the setting tool, and then describe a separate pressure test on the next line at 7 MPa held for 15 minutes. Both entries contain a pressure value, but only one is a regulatory test result. That distinction is critical for compliance reporting and cannot be made reliably with pattern matching across hundreds of files written by different supervisors.

* **The Solution:** I integrated Claude Haiku on AWS Bedrock inside Lambda 2. A structured prompt sends the daily free text summaries alongside the parsed Summary of Changes data. The model returns clean JSON per file containing classified well events, pressure test results with pass or fail flags, converted cement volumes, and AER regulatory numbers. Prompt engineering was the main effort here, specifically teaching the model the field specific distinction between tool shear pressure and a post set pressure test.

*Daily sheet note for Day 1 (left) and Main sheet note for all days (right)*
<p float="left">
  <img alt="Daily Notes" src="https://github.com/user-attachments/assets/ae025dcd-e1be-4049-bf05-23bb0cd2bd20" width="49%" />
  <img alt="Main Notes" src="https://github.com/user-attachments/assets/8352b986-fa69-4f78-a7a0-db565b498972" width="49%" />
</p>


### 4. Handling 750 files arriving simultaneously without rate failures
* **The Challenge:** AWS AppFlow transfers all files from Google Drive in bulk. Without rate control, dropping 750 files into S3 simultaneously would spawn hundreds of concurrent Lambda invocations and hit AWS account concurrency limits, causing silent failures with no visibility into lost data.

* **The Solution:** I decoupled the pipeline using two SQS queues. SQS Queue 1 buffers raw file transfers between S3 and Lambda 1, while SQS Queue 2 buffers intermediate JSON files between S3 and Lambda 2. Both queues use dedicated Dead Letter Queues (DLQs). If a file fails processing after maximum retries, it lands in the DLQ, triggers a CloudWatch depth alarm, and sends an SNS email alert. Failed files can be analyzed, fixed, and redriven directly out of the DLQ without re uploading source files from Google Drive.

---

## Analytics and Downstream BI Layer

### Athena Base Tables and Gold SQL Views
AWS Glue catalogs the Silver Parquet datasets into 5 base tables in Athena. To normalize inconsistent formatting and join related operational data across tables, 5 Gold SQL views were created:

1. **Cost by Supplier View:** Normalizes unit prices, item descriptions, and vendor names for cost tracking.

2. **Load Fluid Detail View:** Tracks fluid types, volumes, and disposal metrics across jobs.

3. **Master Well Summary View:** Joins high level well identifiers, AER regulatory numbers, and execution dates.

4. **Material Transfer Detail View:** Tracks equipment and material movements between well locations.

5. **Plug and Cement Detail View:** Isolates regulatory compliance metrics, plug depths, and pressure test outcomes extracted by the LLM.

### QuickSight Reporting
Amazon QuickSight connects to the Gold Athena SQL views through the SPICE in memory calculation engine on a periodic refresh schedule. This isolates dashboard user traffic when filtering bar charts, cost trends, and compliance metrics from executing live S3 queries in Athena, keeping query costs low and load times fast.

[![Watch Demo](https://img.shields.io/badge/▶️_Watch_the_B2B_QuickSight_Dashboard_Video_Demo-94f2a8?style=for-the-badge)](https://youtu.be/aFxhm_Saku0)

<p align="center">
  <a href="https://youtu.be/aFxhm_Saku0" target="_blank">
    <img src="https://img.youtube.com/vi/aFxhm_Saku0/maxresdefault.jpg" alt="B2B QuickSight Dashboard Demo" width="100%" />
  </a>
</p>

---

## Monitoring and Error Handling

* **Dead Letter Queues (DLQs):** SQS Queue 1 and Queue 2 use DLQs to catch payloads that fail processing after maximum retry attempts.

* **Alerting:** A CloudWatch metric alarm monitors DLQ depth and triggers an SNS email alert if a message lands in the DLQ.

* **Redrive Strategy:** Failed files can be analyzed, fixed, and redriven directly out of the DLQ without re uploading source files from Google Drive.
