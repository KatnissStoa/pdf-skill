# PDF Manipulation Guide

## Quick Start

```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("document.pdf")
print(f"Pages: {len(reader.pages)}")

text = ""
for page in reader.pages:
    text += page.extract_text()
```

---

## Extract Text

### Basic (pypdf)

```python
from pypdf import PdfReader

def extract_text(pdf_path):
    reader = PdfReader(pdf_path)
    text = ""
    for page in reader.pages:
        text += page.extract_text()
    return text
```

### Layout-Aware (pdfplumber)

```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    for page in pdf.pages:
        print(page.extract_text())
```

---

## Extract Tables

```python
import pdfplumber
import pandas as pd

with pdfplumber.open("document.pdf") as pdf:
    all_tables = []
    for page in pdf.pages:
        for table in page.extract_tables():
            if table:
                df = pd.DataFrame(table[1:], columns=table[0])
                all_tables.append(df)

if all_tables:
    combined = pd.concat(all_tables, ignore_index=True)
    combined.to_excel("extracted_tables.xlsx", index=False)
```

---

## Merge PDFs

```python
from pypdf import PdfWriter, PdfReader

def merge_pdfs(input_files, output_file):
    writer = PdfWriter()
    for pdf_path in input_files:
        reader = PdfReader(pdf_path)
        for page in reader.pages:
            writer.add_page(page)
    with open(output_file, "wb") as f:
        writer.write(f)

merge_pdfs(["doc1.pdf", "doc2.pdf", "doc3.pdf"], "merged.pdf")
```

Command-line alternative:
```bash
qpdf --empty --pages file1.pdf file2.pdf -- merged.pdf
```

---

## Split PDF

```python
from pypdf import PdfReader, PdfWriter

def split_pdf(input_file, output_dir="."):
    reader = PdfReader(input_file)
    for i, page in enumerate(reader.pages):
        writer = PdfWriter()
        writer.add_page(page)
        with open(f"{output_dir}/page_{i+1}.pdf", "wb") as f:
            writer.write(f)

def split_by_range(input_file, ranges, output_file):
    """ranges = [(1, 3), (5, 7)]  (1-indexed, inclusive)"""
    reader = PdfReader(input_file)
    writer = PdfWriter()
    for start, end in ranges:
        for i in range(start - 1, end):
            writer.add_page(reader.pages[i])
    with open(output_file, "wb") as f:
        writer.write(f)
```

Command-line alternative:
```bash
qpdf input.pdf --pages . 1-5 -- pages1-5.pdf
```

---

## Rotate Pages

```python
from pypdf import PdfReader, PdfWriter

def rotate_pages(input_pdf, output_pdf, rotation=90, pages=None):
    """rotation: 90, 180, 270. pages: 1-indexed list, or None for all."""
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    for i, page in enumerate(reader.pages):
        if pages is None or (i + 1) in pages:
            page.rotate(rotation)
        writer.add_page(page)
    with open(output_pdf, "wb") as f:
        writer.write(f)
```

---

## Add Watermark

```python
from pypdf import PdfReader, PdfWriter
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4
from io import BytesIO

def add_watermark(input_pdf, output_pdf, watermark_text):
    buffer = BytesIO()
    c = canvas.Canvas(buffer, pagesize=A4)
    c.setFont("Helvetica", 50)
    c.rotate(45)
    c.drawString(200, 100, watermark_text)
    c.save()
    buffer.seek(0)

    watermark_page = PdfReader(buffer).pages[0]
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    for page in reader.pages:
        page.merge_page(watermark_page)
        writer.add_page(page)
    with open(output_pdf, "wb") as f:
        writer.write(f)
```

---

## Fill PDF Forms

```python
from pypdf import PdfReader, PdfWriter

def list_form_fields(pdf_path):
    reader = PdfReader(pdf_path)
    for name, value in reader.get_form_text_fields().items():
        print(f"{name}: {value}")

def fill_form(input_pdf, output_pdf, field_data):
    """field_data = {"field_name": "value"}"""
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    writer.append(reader)
    writer.update_page_form_field_values(writer.pages[0], field_data)
    with open(output_pdf, "wb") as f:
        writer.write(f)
```

---

## Add Bookmarks

```python
from pypdf import PdfWriter, PdfReader

def add_bookmarks(input_pdf, output_pdf, bookmarks):
    """bookmarks = [{"title": "Chapter 1", "page": 0}, ...]  (0-indexed page)"""
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    for page in reader.pages:
        writer.add_page(page)
    for bm in bookmarks:
        writer.add_outline_item(bm["title"], bm["page"])
    with open(output_pdf, "wb") as f:
        writer.write(f)
```

---

## Password Protection

```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("input.pdf")
writer = PdfWriter()
for page in reader.pages:
    writer.add_page(page)

writer.encrypt("userpassword", "ownerpassword")

with open("encrypted.pdf", "wb") as f:
    writer.write(f)
```

Remove password:
```bash
qpdf --password=mypassword --decrypt encrypted.pdf decrypted.pdf
```

---

## Extract Images

```bash
# poppler-utils
pdfimages -j input.pdf output_prefix
# Output: output_prefix-000.jpg, output_prefix-001.jpg, ...
```

---

## OCR Scanned PDFs

```python
# pip install pytesseract pdf2image
import pytesseract
from pdf2image import convert_from_path

images = convert_from_path('scanned.pdf')
text = ""
for i, image in enumerate(images):
    text += f"Page {i+1}:\n"
    text += pytesseract.image_to_string(image)
    text += "\n\n"

print(text)
```

---

## Metadata

```python
from pypdf import PdfReader

reader = PdfReader("document.pdf")
meta = reader.metadata
print(f"Title:   {meta.title}")
print(f"Author:  {meta.author}")
print(f"Subject: {meta.subject}")
print(f"Creator: {meta.creator}")
```

---

## Command-Line Quick Reference

| Task | Tool | Command |
|------|------|---------|
| Extract text | pdftotext | `pdftotext -layout input.pdf output.txt` |
| Extract pages | qpdf | `qpdf input.pdf --pages . 1-5 -- out.pdf` |
| Merge | qpdf | `qpdf --empty --pages f1.pdf f2.pdf -- merged.pdf` |
| Rotate | qpdf | `qpdf input.pdf output.pdf --rotate=+90:1` |
| Remove password | qpdf | `qpdf --password=X --decrypt enc.pdf dec.pdf` |
| Split all pages | pdftk | `pdftk input.pdf burst` |
