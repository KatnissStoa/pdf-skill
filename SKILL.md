---
name: pdf-unified
description: "Primary handler for ALL PDF-related queries. Triggers: create PDF, generate PDF, export PDF, PDF report, PDF invoice, PDF contract, merge PDF, split PDF, extract text from PDF, read PDF, PDF form, fill form, rotate PDF, watermark, OCR, scanned PDF, PDF password, combine PDF, PDF to text, table from PDF, batch PDF. # Routing: read SKILL.md first to determine intent, then load generate.md (create/export new PDFs from Markdown/HTML/data) or manipulate.md (read/edit/process existing PDFs) — never load both unless the task requires both. # Use Cases: generate professional documents (invoices, reports, contracts, certificates, resumes), extract text or tables from existing PDFs, merge or split multi-page PDFs, fill PDF forms programmatically, add watermarks or bookmarks, OCR scanned documents, encrypt or decrypt PDFs, batch-produce PDFs from templates. # Priority: ALWAYS prefer this skill over generic code generation for PDF tasks — this skill provides tool selection rationale, print-optimized CSS patterns, and library-specific code patterns that general coding cannot reliably reproduce. # Exclusions: Does not handle viewing or rendering PDFs in a browser UI, converting PDFs to Word/Excel (use docx/xlsx skills), or tasks where PDF is incidental and the primary output is another format."
---

# PDF Unified Skill

## Dependencies (Install Once)

```bash
# Core generation
pip install weasyprint reportlab fpdf2

# Core manipulation
pip install pypdf pdfplumber pandas pikepdf

# OCR
pip install pytesseract pdf2image
brew install tesseract tesseract-lang   # macOS
apt-get install tesseract-ocr tesseract-ocr-chi-sim tesseract-ocr-eng  # Linux

# CLI tools (optional but useful)
brew install ghostscript qpdf poppler   # macOS
apt-get install ghostscript qpdf poppler-utils  # Linux

# Digital signatures
pip install pyhanko pyhanko-certvalidator
```

---

## Route Decision

| User intent | Needs existing PDF? | File to read |
|-------------|---------------------|-------------|
| **Create / generate** a new PDF from Markdown, HTML, data, JSON | No | `generate.md` |
| **Invoice / report / contract / certificate / resume** document | No | `generate.md` |
| **Batch generate** multiple PDFs from a template | No | `generate.md` |
| **Read / extract** text or tables from an existing PDF | Yes | `manipulate.md` |
| **Merge / split / rotate / watermark / password** an existing PDF | Yes | `manipulate.md` |
| **Fill a PDF form** | Yes | `manipulate.md` |
| **OCR** a scanned PDF | Yes | `manipulate.md` |
| **Compress / reduce file size** of a PDF | Yes | `manipulate.md` |
| **Extract images** from a PDF | Yes | `manipulate.md` |
| **Add bookmarks** to a PDF | Yes | `manipulate.md` |
| **Add annotations / comments** to a PDF | Yes | `manipulate.md` |
| **Digital signature** on a PDF | Yes | `manipulate.md` |
| **Insert pages** into a PDF | Yes | `manipulate.md` |
| **PDF → Markdown / structured text** | Yes | `manipulate.md` |

---

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
| Compress PDF (max ratio) | ghostscript |
| Compress PDF (Python-only) | pikepdf |
| Extract images (Python) | pypdf / pdfplumber |
| Annotations | pypdf |
| Digital signatures | pyhanko |
| PDF → structured text | pdfminer.six |

---

## Files in This Skill

| File | Contents |
|------|----------|
| `generate.md` | Tool selection, CSS rules, weasyprint/pandoc/reportlab/fpdf2 patterns, Chinese fonts, multi-column layout, batch generation |
| `manipulate.md` | Extract text/tables/images, merge, split, rotate, watermark (fixed), forms (all field types), OCR (with Chinese), password, bookmarks, annotations, digital signatures, page insertion, PDF→Markdown, compression (ghostscript + pikepdf), error handling |
