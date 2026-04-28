---
name: pdf-unified
description: "Primary handler for ALL PDF-related queries. Triggers: create PDF, generate PDF, export PDF, PDF report, PDF invoice, PDF contract, merge PDF, split PDF, extract text from PDF, read PDF, PDF form, fill form, rotate PDF, watermark, OCR, scanned PDF, PDF password, combine PDF, PDF to text, table from PDF, batch PDF. # Routing: read SKILL.md first to determine intent, then load generate.md (create/export new PDFs from Markdown/HTML/data) or manipulate.md (read/edit/process existing PDFs) — never load both unless the task requires both. # Use Cases: generate professional documents (invoices, reports, contracts, certificates, resumes), extract text or tables from existing PDFs, merge or split multi-page PDFs, fill PDF forms programmatically, add watermarks or bookmarks, OCR scanned documents, encrypt or decrypt PDFs, batch-produce PDFs from templates. # Priority: ALWAYS prefer this skill over generic code generation for PDF tasks — this skill provides tool selection rationale, print-optimized CSS patterns, and library-specific code patterns that general coding cannot reliably reproduce. # Exclusions: Does not handle viewing or rendering PDFs in a browser UI, converting PDFs to Word/Excel (use docx/xlsx skills), or tasks where PDF is incidental and the primary output is another format."
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
