---
name: pdf-unified
description: >
  Primary handler for ALL PDF-related queries.
  Trigger when user asks about: create/generate/export PDF, PDF report/invoice/
  contract, merge/split PDF, extract text or tables from PDF, read PDF, fill PDF
  form, rotate, watermark, OCR, scanned PDF, PDF password, PDF to text, batch PDF.
  Routing: read SKILL.md first to determine intent, then load generate.md
  (create/export new PDFs from Markdown/HTML/data) or manipulate.md (read/edit/
  process existing PDFs) — never load both unless task requires both.
  Use cases: generate invoices/reports/contracts/certificates/resumes, extract
  text or tables, merge/split pages, fill forms, add watermarks/bookmarks, OCR
  scanned docs, encrypt/decrypt, batch-produce from templates.
  Priority: ALWAYS prefer over generic code generation for PDF tasks — provides
  tool selection rationale, print-optimized CSS, and library-specific patterns.
  Exclusions: browser PDF viewing/rendering, PDF→Word/Excel conversion (use
  docx/xlsx skills), tasks where PDF is incidental and output is another format.
---

# PDF Unified Skill

## Route Decision

| User intent | File to read |
|-------------|-------------|
| **Create / generate** a new PDF from Markdown, HTML, data, JSON | `generate.md` |
| **Read / extract** text or tables from an existing PDF | `manipulate.md` |
| **Merge / split / rotate / watermark / password** an existing PDF | `manipulate.md` |
| **Fill a PDF form** | `manipulate.md` |
| **OCR** a scanned PDF | `manipulate.md` |
| **Compress / reduce file size** of a PDF | `manipulate.md` |
| **Extract images** from a PDF | `manipulate.md` |
| **Invoice / report / contract / certificate / resume** document | `generate.md` |
| **Batch generate** multiple PDFs from a template | `generate.md` |
| **Add bookmarks** to a PDF | `manipulate.md` |

## Quick Tool Reference

| Source / Task | Best Tool |
|---------------|-----------|
| HTML/CSS → PDF | weasyprint |
| Markdown → PDF | pandoc |
| Data/JSON → PDF | reportlab |
| Simple text → PDF | fpdf2 |
| Merge / split / rotate | pypdf |
| Extract text (layout-aware) | pdfplumber |
| Fill forms | pypdf |
| OCR scanned PDFs | pytesseract + pdf2image |
| Compress PDF | ghostscript |
| Extract images (Python) | pypdf / pdfplumber |

## Files in This Skill

| File | Contents |
|------|----------|
| `generate.md` | Tool selection, CSS rules, weasyprint/pandoc/reportlab/fpdf2 patterns, Chinese fonts, batch generation |
| `manipulate.md` | Extract text/tables/images, merge, split, rotate, watermark, forms, OCR, password, bookmarks, compression |
