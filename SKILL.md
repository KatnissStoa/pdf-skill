---
name: pdf-unified
description: Unified PDF skill — generate new PDFs from Markdown/HTML/data/templates, or manipulate existing PDFs (extract text/tables, merge, split, rotate, watermark, fill forms, OCR, password). Routes to the right approach based on the user's intent.
---

# PDF Unified Skill

## Route Decision

Read the user's request and route to the appropriate reference file:

| User intent | File to read |
|-------------|-------------|
| **Create / generate** a new PDF from Markdown, HTML, data, JSON, template | `generate.md` |
| **Read / extract** text or tables from an existing PDF | `manipulate.md` |
| **Merge / split / rotate / watermark / password** an existing PDF | `manipulate.md` |
| **Fill a PDF form** | `manipulate.md` |
| **OCR** a scanned PDF | `manipulate.md` |
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

## Files in This Skill

| File | Contents |
|------|----------|
| `generate.md` | Tool selection, CSS rules, weasyprint/pandoc/reportlab/fpdf2 patterns, batch generation |
| `manipulate.md` | Extract text/tables, merge, split, rotate, watermark, forms, OCR, password, bookmarks |
