# Document Loading in LangChain: Practical Reference Guide

Document loaders convert unstructured or semi-structured data from various sources into LangChain `Document` objects, which contain:

* `page_content` (the text string)

* `metadata` (source, page number, title, coordinates, etc.)

## Quick Summary Table

| Document Type | Recommended Loader | Key Packages | Best For | 
 | ----- | ----- | ----- | ----- | 
| **PDF (Simple)** | `PyPDFLoader` | `pypdf` | Fast, lightweight extraction of digital text | 
| **PDF (Precise Layout)** | `PyMuPDFLoader` / `PDFMinerLoader` | `pymupdf` / `pdfminer.six` | Multi-column text, bounding boxes, precise reading order | 
| **PDF (With Tables)** | `pdfplumber` / `PyMuPDFLoader` | `pdfplumber`, `pymupdf` | Retaining grid alignment, extracting tabular structures | 
| **PDF (Complex / Scanned)** | `UnstructuredPDFLoader` | `unstructured`, `pdf2image`, `tesseract` | Mixed layouts, figures, scans requiring OCR, images | 
| **Plain Text (TXT)** | `TextLoader` | *Built-in* | Raw, unformatted logs and text files | 
| **Markdown (MD)** | `UnstructuredMarkdownLoader` | `unstructured`, `markdown` | Retaining header hierarchy and section semantics | 
| **Single Web Page** | `WebBaseLoader` | `beautifulsoup4`, `urllib3` | HTML scraping, articles, blog posts | 
| **Entire Website** | `RecursiveUrlLoader` | `beautifulsoup4` | Crawling doc sites recursively up to a given depth | 
| **CSV** | `CSVLoader` | *Built-in (`csv`)* | Tabular data where each row becomes a separate chunk | 
| **JSON / JSONL** | `JSONLoader` | `jq` | Nested or key-value structured data using JSONPointer/jq | 

## 1. PDF Loaders

### A. Simple PDF (Linear Digital Text)

* **Loader:** `PyPDFLoader`

* **Install:** `pip install pypdf`

* **Use Case:** Standard text-heavy PDFs with single-column layouts (reports, simple essays).

```
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("documents/linear_report.pdf")
docs = loader.load_and_split()

# Each page maps to one Document with metadata {'source': ..., 'page': int}
print(f"Loaded {len(docs)} pages.")
print(docs[0].page_content[:200])

```

### B. PDF with Precise Layout (Multi-column, Academic Papers)

* **Loader:** `PyMuPDFLoader` (or `PDFMinerLoader`)

* **Install:** `pip install pymupdf`

* **Use Case:** Multi-column magazines, newsletters, and papers where reading order matters and bounding-box coordinates may be needed.

```
from langchain_community.document_loaders import PyMuPDFLoader

# PyMuPDF is extremely fast and parses two-column layouts accurately
loader = PyMuPDFLoader("documents/two_column_paper.pdf")
docs = loader.load()
print(docs[0].metadata)

```

### C. PDF with Tables

* **Loader:** `pdfplumber` (custom wrapper or extraction)

* **Install:** `pip install pdfplumber`

* **Use Case:** Financial reports, invoices, and balance sheets where standard loaders scramble row-and-column data into disjointed strings.

```
import pdfplumber
from langchain_core.documents import Document

def load_pdf_with_tables(file_path: str) -> list[Document]:
    docs = []
    with pdfplumber.open(file_path) as pdf:
        for idx, page in enumerate(pdf.pages):
            text = page.extract_text() or ""
            tables = page.extract_tables()
            
            # Format extracted tables into readable markdown format
            table_strings = []
            for table in tables:
                table_md = "\n".join([" | ".join([cell or "" for cell in row]) for row in table])
                table_strings.append(table_md)
            
            combined_content = f"{text}\n\n" + "\n\n".join(table_strings)
            docs.append(Document(page_content=combined_content, metadata={"source": file_path, "page": idx + 1}))
    return docs

docs = load_pdf_with_tables("documents/financial_report.pdf")

```

### D. Complex / Scanned PDF (OCR & Multi-Modal Elements)

* **Loader:** `UnstructuredPDFLoader`

* **Install:** `pip install "unstructured[all-docs]" pdf2image pytesseract`

* **Use Case:** Scanned archives, PDFs with embedded images, forms, and unpredictable mixed layouts.

```
from langchain_community.document_loaders import UnstructuredPDFLoader

# mode="elements" chunks headers, narrative text, and captions separately
loader = UnstructuredPDFLoader(
    "documents/scanned_archive.pdf",
    mode="elements",
    strategy="hi_res"  # triggers OCR when digital text is unavailable
)
docs = loader.load()

```

## 2. Text & Markdown Loaders

### Plain Text (`.txt`)

* **Loader:** `TextLoader`

* **Install:** No extra packages needed.

```
from langchain_community.document_loaders import TextLoader

loader = TextLoader("data/transcript.txt", encoding="utf-8")
docs = loader.load()

```

### Markdown (`.md`)

* **Loader:** `UnstructuredMarkdownLoader`

* **Install:** `pip install unstructured markdown`

* **Use Case:** Technical documentation; preserves header hierarchy (`#`, `##`, `###`).

```
from langchain_community.document_loaders import UnstructuredMarkdownLoader

loader = UnstructuredMarkdownLoader("docs/README.md")
docs = loader.load()

```

## 3. Web Scraping Loaders

### Single Web Page

* **Loader:** `WebBaseLoader`

* **Install:** `pip install beautifulsoup4`

* **Use Case:** Scraping blog posts, documentation pages, or news articles.

```
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader("https://en.wikipedia.org/wiki/Artificial_intelligence")
docs = loader.load()

```

### Entire Website (Crawl with Depth Limit)

* **Loader:** `RecursiveUrlLoader`

* **Install:** `pip install beautifulsoup4`

* **Use Case:** Crawling documentation domains up to a specified depth limit.

```
from langchain_community.document_loaders import RecursiveUrlLoader
from bs4 import BeautifulSoup

def clean_html(html: str) -> str:
    soup = BeautifulSoup(html, "html.parser")
    return soup.get_text(separator="\n", strip=True)

loader = RecursiveUrlLoader(
    url="https://docs.example.com",
    max_depth=2,
    extractor=clean_html
)
docs = loader.load()

```

## 4. Structured Data (CSV, JSON, JSONL)

### CSV Files

* **Loader:** `CSVLoader`

* **Install:** No extra packages needed.

* **Use Case:** Each row is converted into an independent `Document` object where column headers serve as field labels.

```
from langchain_community.document_loaders import CSVLoader

loader = CSVLoader(
    file_path="data/movies.csv",
    source_column="title"  # sets metadata['source'] to the movie title
)
docs = loader.load()

```

### JSON / JSONL

* **Loader:** `JSONLoader`

* **Install:** `pip install jq`

* **Use Case:** Extracting specific schema fields or array entries using `jq` syntax.

```
from langchain_community.document_loaders import JSONLoader

# Extracts text from the 'plot' field of an array of movie objects
loader = JSONLoader(
    file_path="data/bollywood_movies.json",
    jq_schema=".movies[].plot",
    text_content=True
)
docs = loader.load()

```

## Decision Matrix

```
Is it a PDF?
 ├── Simple text, 1 column? ──────────────> PyPDFLoader (fastest)
 ├── Multi-column / exact layout? ────────> PyMuPDFLoader
 ├── Contains tabular grids? ─────────────> pdfplumber
 └── Scanned or mixed media? ─────────────> UnstructuredPDFLoader (hi_res)

Is it Web Content?
 ├── Single page? ────────────────────────> WebBaseLoader
 └── Entire sub-tree? ────────────────────> RecursiveUrlLoader

Is it Structured / Data?
 ├── Tabular rows? ───────────────────────> CSVLoader
 └── Nested JSON / API responses? ────────> JSONLoader (with jq)

```