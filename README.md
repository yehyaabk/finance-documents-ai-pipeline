# Finance Documents AI Pipeline on Databricks

An end-to-end data pipeline that turns unstructured **PDF documents** (invoices, purchase orders and receipts) into **structured, queryable tables**, using Databricks AI Functions, Unity Catalog and the **Medallion Architecture**.

---

##  The Idea

Companies receive many financial documents as PDFs: invoices from suppliers, purchase orders, payment receipts. The information inside them (amounts, dates, suppliers, line items) is valuable, but it is locked in files that cannot be queried with SQL.

This project automates the whole process:

1. **Store** the raw PDF files in a Unity Catalog volume.
2. **Parse** each document with AI to understand its layout and content.
3. **Classify** each document as an invoice, a purchase order or a receipt.
4. **Extract** the relevant fields for each document type into structured tables.
5. **Clean and model** the data for analytics and dashboards (next steps).

The result: documents that used to require manual reading become rows and columns that can be analyzed, joined and visualized.

---

##  Architecture

The project follows the **modern Databricks data workflow** and the **Medallion Architecture**, a layered design recommended by Databricks where data becomes cleaner and more valuable at each step.

```
   PDF files                 BRONZE                         SILVER                    GOLD
 (invoices, POs,  ──►  Parsed, classified and   ──►   Cleaned and validated   ──►  Business-ready
    receipts)          extracted documents             structured tables          tables & KPIs
```

| Layer | Purpose | Status |
|---|---|---|
| **Bronze** | Raw files and their first AI-processed version (parsed, classified, extracted) | ✅ Done |
| **Silver** | Cleaned data: correct types, standardized formats, quality checks | 🔜 Next step |
| **Gold** | Aggregated, business-ready tables for dashboards | 🔜 Planned |

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
├── silver_layer                         ← cleaned tables (next step)
└── gold_layer                           ← business tables (planned)
```

---

##  What Has Been Done So Far

All the work below is in the notebook **`Prepare_Bronze_Layer`**, which prepares the data before the cleaning step.

### Step 1: Store the raw documents
The PDF files are uploaded to a **Unity Catalog volume**, the place for unstructured files in Databricks. They are read as binary content with `read_files(..., format => 'binaryFile')`.

### Step 2: Parse the documents
Each PDF is parsed with **`ai_parse_document()`**, which identifies the layout of the document (titles, paragraphs, tables) and returns it as a structured `VARIANT`.

A readable text version of each document is also created by joining the content of all its elements:

```sql
concat_ws('\n',
  transform(
    try_cast(parsed_content:document:elements AS ARRAY<VARIANT>),
    x -> try_cast(x:content AS STRING)
  )
)
```

### Step 3: Classify the documents
Each document is classified with **`ai_classify()`** into one of these categories: invoice, purchase order, receipt or other.

The result is saved in the table **`documents_parsed_classified`**, which contains, for each document:

| Column | Description |
|---|---|
| `path` | Location of the original file |
| `parsed_content` | Full output of `ai_parse_document()` |
| `element_pretty_format` | Readable text version of the document |
| `document_type` | Category assigned by `ai_classify()` |

### Step 4: Extract structured data by document type
Using **`ai_extract()`**, a different extraction schema is applied to each document type:

| Table | Main fields extracted |
|---|---|
| `invoices` | invoice number, invoice date, PO number, seller, seller email, buyer, shipping address, payment method, currency, total amount |
| `purchase_orders` | PO number, PO date, requested ship date, buyer, vendor, currency, subtotal, shipping, total amount |
| `receipts` | receipt number, payment date, seller, amount paid, payment method |
| `invoice_line_items` | invoice number, description, quantity, unit price, line amount |

Line items are extracted as an **array** and turned into one row per item with `EXPLODE`.

---

##  Next Steps

- [ ] **Silver layer:** clean the bronze tables (convert dates to `DATE`, amounts to `DECIMAL`, standardize text, remove duplicates, handle missing values)
- [ ] **Quality checks:** compare the sum of invoice line items with the invoice total, and flag documents with extraction errors or missing fields
- [ ] **Gold layer:** build business tables such as spend by vendor, monthly spend, and unpaid invoices (invoices without a matching receipt)
- [ ] **Dashboard:** visualize the gold tables with a Databricks AI/BI dashboard
- [ ] **Orchestration:** create a Databricks **Job** with a **file arrival trigger**, so the pipeline runs automatically whenever new documents are added to the volume

---

##  Tech Stack

- **Databricks** (notebooks, serverless compute)
- **Unity Catalog** (catalog, schemas, volumes, tables)
- **Databricks SQL** and **AI Functions**: `ai_parse_document()`, `ai_classify()`, `ai_extract()`
- **Delta Lake** tables
- **Databricks Jobs** for orchestration (planned)

---

##  Repository Structure

```
├── README.md
└── notebooks/
    └── Prepare_Bronze_Layer      ← parsing, classification and extraction (bronze layer)
```

---

## Data Privacy

This repository contains **code only**. The documents used for this project are **sample documents** with invented companies and amounts. No real personal or financial data is published.

---

##  What I Learned

- Organizing data with Unity Catalog: catalogs, schemas, volumes and tables
- Designing a pipeline with the Medallion Architecture
- Processing unstructured documents with Databricks AI Functions
- Working with semi-structured `VARIANT` data in SQL (`:`, `::`, `try_cast`, `transform`, `concat_ws`, `EXPLODE`)
- Preparing a pipeline for automation with Databricks Jobs and triggers