# Finance Documents AI Pipeline on Databricks

**Project by Yehya ABOU KHECHFE**

An **automated, end-to-end data pipeline** that turns unstructured **PDF documents** (invoices, purchase orders and receipts) into **structured, business-ready tables**, using Databricks AI Functions, Unity Catalog, the **Medallion Architecture** and **Databricks Jobs**.

Drop a PDF into the raw storage volume, and the pipeline does the rest: it parses, classifies and extracts the document, cleans the data, and refreshes the business tables, with no manual step.

---

## The Idea

Companies receive many financial documents as PDFs: invoices from suppliers, purchase orders, payment receipts. The information inside them (amounts, dates, vendors, line items) is valuable, but it is locked in files that cannot be queried with SQL, and reading them by hand is slow and error-prone.

This project automates the whole process:

1. **Store** the raw PDF files in a Unity Catalog volume.
2. **Parse** each document with AI to understand its layout and content.
3. **Classify** each document as an invoice, a purchase order or a receipt.
4. **Extract** the relevant fields for each document type into structured tables.
5. **Clean** the extracted data so it can be trusted.
6. **Build business tables** that answer real questions: how much we spend, with whom, and whether invoices match purchase orders.
7. **Automate** everything with a Databricks Job that runs as soon as new files arrive.

The result: documents that used to require manual reading become rows and columns that can be analyzed, joined and visualized, automatically.

---

## Tech Stack

- **Databricks** (notebooks, serverless compute)
- **Unity Catalog** (catalog, schemas, volumes, tables)
- **Databricks SQL** and **AI Functions**: `ai_parse_document()`, `ai_classify()`, `ai_extract()`
- **Delta Lake** tables
- **Databricks Jobs** for orchestration, with a file arrival trigger

---

## Architecture

The project follows the **modern Databricks data workflow** and the **Medallion Architecture**, a layered design recommended by Databricks where data becomes cleaner and more valuable at each step.

```
   PDF files                 BRONZE                     SILVER                  GOLD
 (invoices, POs,  ──►  Parsed, classified and  ──►  Cleaned, typed and  ──►  Business-ready
    receipts)          extracted documents          validated tables         tables & KPIs
       ▲
       └── a new file in the raw volume triggers the whole pipeline automatically
```

| Layer | Purpose | Notebook |
|---|---|---|
| **Bronze** | Raw files and their first AI-processed version (parsed, classified, extracted) | [1_Prepare_Bronze_Layer](notebooks/1_Prepare_Bronze_Layer.ipynb) |
| **Silver** | Cleaned data: correct types, standardized formats, quality checks | [2_Bronze_To_Silver_Transformations](notebooks/2_Bronze_To_Silver_Transformations.ipynb) |
| **Gold** | Aggregated, business-ready tables for dashboards | [3_Silver_To_Gold](notebooks/3_Silver_To_Gold.ipynb) |
| **Orchestration** | Databricks Job running the three notebooks in order, triggered by new files | — |

### Data organization in Unity Catalog

```
medical_finance                          ← catalog
├── bronze_layer                         ← schema
│   ├── (volume)                         ← raw PDF files
│   ├── documents_parsed_classified      ← every document, parsed and classified
│   ├── invoices                         ← one row per invoice
│   ├── purchase_orders                  ← one row per purchase order
│   ├── receipts                         ← one row per receipt
│   └── invoice_line_items               ← one row per product/service line of each invoice
├── silver_layer                         ← cleaned tables
└── gold_layer                           ← business tables
```

---

## Orchestration: the Databricks Job

The three notebooks are orchestrated by a **Databricks Job**, so the pipeline runs **automatically, in the right order**.

![Databricks Job with three tasks: Prepare_Bronze, Prepare_Silver_Tables and Ready_To_Use_Table](https://github.com/user-attachments/assets/58743db5-5e8e-4e17-bb9b-6a9dcecaf426)

### Tasks

| Order | Task | Notebook | What it does |
|---|---|---|---|
| 1 | **Prepare_Bronze** | [1_Prepare_Bronze_Layer](notebooks/1_Prepare_Bronze_Layer.ipynb) | Parses, classifies and extracts the new documents |
| 2 | **Prepare_Silver_Tables** | [2_Bronze_To_Silver_Transformations](notebooks/2_Bronze_To_Silver_Transformations.ipynb) | Cleans the extracted data |
| 3 | **Ready_To_Use_Table** | [3_Silver_To_Gold](notebooks/3_Silver_To_Gold.ipynb) | Builds the business-ready gold tables |

### Dependency chain

```
Prepare_Bronze  ──►  Prepare_Silver_Tables  ──►  Ready_To_Use_Table
```

Each task starts **only after the previous one has succeeded**. This guarantees that the silver tables are never built from incomplete bronze data, and that the gold tables are never built from data that has not been cleaned yet. If a task fails, the next ones do not run.

### Trigger: file arrival

The Job uses a **file arrival trigger** (an event-based trigger) on the Unity Catalog **volume that contains the raw files**.

- When new PDF files are dropped into the volume, the Job **starts automatically**.
- No fixed schedule is needed: the pipeline runs **only when there is new data**, which avoids useless runs and saves compute.

### Compute

All tasks run on **serverless compute**: Databricks manages and scales the resources automatically, with no cluster to configure.

---

## What I Learned

- Organizing data with Unity Catalog: catalogs, schemas, volumes and tables
- Designing a pipeline with the Medallion Architecture
- Processing unstructured documents with Databricks AI Functions
- Working with semi-structured `VARIANT` data in SQL (`:`, `::`, `try_cast`, `transform`, `concat_ws`, `EXPLODE`)
- Orchestrating a pipeline with Databricks Jobs, task dependencies and event-based triggers
