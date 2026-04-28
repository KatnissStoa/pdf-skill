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

## Error Handling

Handle the three most common failure modes before doing anything else with an existing PDF:

```python
from pypdf import PdfReader
from pypdf.errors import PdfReadError

def open_pdf(path, password=None):
    """Open a PDF, handling encrypted, corrupted, and permission-restricted files."""
    try:
        reader = PdfReader(path)
    except PdfReadError as e:
        raise ValueError(f"Corrupted or invalid PDF: {path}") from e

    if reader.is_encrypted:
        if password is None:
            raise PermissionError(f"PDF is encrypted — provide a password")
        ok = reader.decrypt(password)
        if ok == 0:
            raise PermissionError("Wrong password")

    # Check copy permission (bit 5 of permissions flags)
    if reader.metadata and not reader._override_encryption:
        perms = reader._encryption.permissions if hasattr(reader, '_encryption') else None
        # pypdf does not expose permissions cleanly; extraction will raise if blocked

    return reader

# Usage
try:
    reader = open_pdf("document.pdf", password="secret")
except (ValueError, PermissionError) as e:
    print(e)
```

---

## Extract Text

### Basic (pypdf)

```python
from pypdf import PdfReader

def extract_text(pdf_path, password=None):
    reader = PdfReader(pdf_path)
    if reader.is_encrypted:
        reader.decrypt(password or "")
    return "\n".join(page.extract_text() or "" for page in reader.pages)
```

### Layout-Aware (pdfplumber)

```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    for page in pdf.pages:
        print(page.extract_text())
```

---

## PDF → Structured Text / Markdown

Use `pdfminer.six` to preserve heading hierarchy and layout structure:

```python
# pip install pdfminer.six
from pdfminer.high_level import extract_pages
from pdfminer.layout import LTTextBox, LTChar, LTAnon

def pdf_to_markdown(pdf_path):
    lines = []
    for page_layout in extract_pages(pdf_path):
        for element in page_layout:
            if not isinstance(element, LTTextBox):
                continue
            text = element.get_text().strip()
            if not text:
                continue
            # Detect heading by font size of first character
            font_size = 12
            for line in element:
                for char in line:
                    if isinstance(char, LTChar):
                        font_size = char.size
                        break
                break
            if font_size >= 18:
                lines.append(f"# {text}")
            elif font_size >= 14:
                lines.append(f"## {text}")
            else:
                lines.append(text)
            lines.append("")
    return "\n".join(lines)

md = pdf_to_markdown("document.pdf")
with open("output.md", "w") as f:
    f.write(md)
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

## Extract Images

### Python (pypdf)

```python
from pypdf import PdfReader

reader = PdfReader("document.pdf")
for page_num, page in enumerate(reader.pages):
    for img_index, image in enumerate(page.images):
        ext = image.name.split(".")[-1] or "png"
        with open(f"page{page_num+1}_img{img_index+1}.{ext}", "wb") as f:
            f.write(image.data)
```

### Python (pdfplumber — with position info)

```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    for page_num, page in enumerate(pdf.pages):
        for img_index, img in enumerate(page.images):
            print(f"Page {page_num+1} image {img_index+1}: "
                  f"{img['width']}x{img['height']} at ({img['x0']:.0f}, {img['y0']:.0f})")
```

### Command-line (poppler)

```bash
pdfimages -j input.pdf output_prefix
# Output: output_prefix-000.jpg, output_prefix-001.jpg, ...
```

---

## Compress / Reduce File Size

### ghostscript (recommended — best compression ratio)

```bash
# Screen quality (~72 dpi) — smallest file, good for email
gs -dBATCH -dNOPAUSE -q -sDEVICE=pdfwrite \
   -dPDFSETTINGS=/screen \
   -sOutputFile=compressed.pdf input.pdf

# Print quality (~300 dpi) — balanced
gs -dBATCH -dNOPAUSE -q -sDEVICE=pdfwrite \
   -dPDFSETTINGS=/printer \
   -sOutputFile=compressed.pdf input.pdf

# Prepress quality — minimal compression, maximum fidelity
gs -dBATCH -dNOPAUSE -q -sDEVICE=pdfwrite \
   -dPDFSETTINGS=/prepress \
   -sOutputFile=compressed.pdf input.pdf
```

`-dPDFSETTINGS` levels: `/screen` < `/ebook` < `/printer` < `/prepress`

### pikepdf (Python-only — no system dependency)

Use when ghostscript is not available. Re-compresses streams and removes redundant objects:

```python
# pip install pikepdf
import pikepdf

with pikepdf.open("input.pdf") as pdf:
    pdf.save(
        "compressed.pdf",
        compress_streams=True,
        object_stream_mode=pikepdf.ObjectStreamMode.generate,
        recompress_flate=True,
    )
```

For aggressive image downsampling (requires Pillow):

```python
import pikepdf
from PIL import Image
import io

def compress_images(pdf_path, output_path, max_dpi=150, quality=75):
    with pikepdf.open(pdf_path) as pdf:
        for page in pdf.pages:
            if "/Resources" not in page:
                continue
            resources = page["/Resources"]
            if "/XObject" not in resources:
                continue
            for name, xobj in resources["/XObject"].items():
                xobj = xobj.with_same_owner_as(pdf)
                if xobj.get("/Subtype") != "/Image":
                    continue
                try:
                    img_data = xobj.read_raw_bytes()
                    img = Image.open(io.BytesIO(img_data)).convert("RGB")
                    buf = io.BytesIO()
                    img.save(buf, format="JPEG", quality=quality)
                    xobj.write(buf.getvalue(), filter=pikepdf.Name("/DCTDecode"))
                except Exception:
                    pass  # skip images that can't be re-compressed
        pdf.save(output_path)
```

### Python (pypdf — lossless only)

pypdf can remove redundant objects but does not re-compress images:

```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("input.pdf")
writer = PdfWriter()

for page in reader.pages:
    page.compress_content_streams()
    writer.add_page(page)

writer.compress_identical_objects(remove_identicals=True, remove_orphans=True)

with open("compressed.pdf", "wb") as f:
    writer.write(f)
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

Command-line:
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

Command-line:
```bash
qpdf input.pdf --pages . 1-5 -- pages1-5.pdf
```

---

## Insert Pages

Insert pages from one PDF into another at a specific position:

```python
from pypdf import PdfReader, PdfWriter

def insert_pages(base_pdf, insert_pdf, position, output_pdf):
    """Insert all pages of insert_pdf into base_pdf before `position` (0-indexed)."""
    base   = PdfReader(base_pdf)
    insert = PdfReader(insert_pdf)
    writer = PdfWriter()

    for page in base.pages[:position]:
        writer.add_page(page)
    for page in insert.pages:
        writer.add_page(page)
    for page in base.pages[position:]:
        writer.add_page(page)

    with open(output_pdf, "wb") as f:
        writer.write(f)

# Insert pages from "cover.pdf" before page 2 (0-indexed = 1)
insert_pages("report.pdf", "cover.pdf", position=1, output_pdf="report_with_cover.pdf")
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

The watermark page must match each source page's size exactly — assuming A4 causes misaligned or cropped watermarks on other page sizes.

```python
from pypdf import PdfReader, PdfWriter
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4
from io import BytesIO

def make_watermark_page(text, page_width, page_height, font="Helvetica",
                        font_size=48, opacity=0.3, color=(0.5, 0.5, 0.5)):
    """Create a single watermark page sized to match the target page."""
    buf = BytesIO()
    c = canvas.Canvas(buf, pagesize=(page_width, page_height))
    c.setFont(font, font_size)
    c.setFillColorRGB(*color, alpha=opacity)
    # Rotate around the page centre so the text is diagonal
    cx, cy = page_width / 2, page_height / 2
    c.saveState()
    c.translate(cx, cy)
    c.rotate(45)
    text_width = c.stringWidth(text, font, font_size)
    c.drawString(-text_width / 2, 0, text)
    c.restoreState()
    c.save()
    buf.seek(0)
    return PdfReader(buf).pages[0]

def add_watermark(input_pdf, output_pdf, watermark_text,
                  font="Helvetica", font_size=48, opacity=0.3):
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    for page in reader.pages:
        w = float(page.mediabox.width)
        h = float(page.mediabox.height)
        wm_page = make_watermark_page(watermark_text, w, h,
                                      font=font, font_size=font_size, opacity=opacity)
        page.merge_page(wm_page)
        writer.add_page(page)
    with open(output_pdf, "wb") as f:
        writer.write(f)
```

Chinese watermark — register a CJK font first:

```python
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont

pdfmetrics.registerFont(TTFont("NotoSansCJK", "NotoSansCJKsc-Regular.ttf"))
add_watermark("input.pdf", "output.pdf", "机密文件", font="NotoSansCJK", font_size=48)
```

---

## Fill PDF Forms

### List all fields first

```python
from pypdf import PdfReader

def list_form_fields(pdf_path):
    reader = PdfReader(pdf_path)
    fields = reader.get_fields()
    for name, field in (fields or {}).items():
        ft   = field.get("/FT", "unknown")    # /Tx=text, /Btn=button, /Ch=choice
        val  = field.get("/V", "")
        opts = field.get("/Opt", [])
        print(f"{name!r:30s}  type={ft}  value={val!r}  options={opts}")
```

### Fill text fields

```python
from pypdf import PdfReader, PdfWriter

def fill_text_fields(input_pdf, output_pdf, field_data):
    """field_data = {"FieldName": "value"}"""
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    writer.append(reader)
    writer.update_page_form_field_values(writer.pages[0], field_data)
    with open(output_pdf, "wb") as f:
        writer.write(f)
```

### Fill checkboxes

```python
from pypdf import PdfReader, PdfWriter
from pypdf.generic import NameObject

def fill_checkboxes(input_pdf, output_pdf, checkbox_data):
    """checkbox_data = {"FieldName": True/False}"""
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    writer.append(reader)

    for page in writer.pages:
        if "/Annots" not in page:
            continue
        for annot in page["/Annots"]:
            annot = annot.get_object()
            field_name = annot.get("/T")
            if field_name and field_name in checkbox_data:
                checked = checkbox_data[field_name]
                # Determine the On value from the field's appearance states
                ap = annot.get("/AP", {})
                on_value = "/Yes"
                if "/N" in ap:
                    states = ap["/N"].keys()
                    on_value = next((s for s in states if s != "/Off"), "/Yes")
                annot.update({
                    NameObject("/V"):  NameObject(on_value if checked else "/Off"),
                    NameObject("/AS"): NameObject(on_value if checked else "/Off"),
                })

    with open(output_pdf, "wb") as f:
        writer.write(f)
```

### Fill dropdowns / list boxes

```python
from pypdf import PdfReader, PdfWriter
from pypdf.generic import NameObject, createStringObject

def fill_dropdown(input_pdf, output_pdf, dropdown_data):
    """dropdown_data = {"FieldName": "selected_value"}"""
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    writer.append(reader)

    for page in writer.pages:
        if "/Annots" not in page:
            continue
        for annot in page["/Annots"]:
            annot = annot.get_object()
            field_name = annot.get("/T")
            if field_name and field_name in dropdown_data:
                value = dropdown_data[field_name]
                annot.update({
                    NameObject("/V"):  createStringObject(value),
                    NameObject("/DV"): createStringObject(value),
                })

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

## Annotations

### Read existing annotations

```python
from pypdf import PdfReader

def read_annotations(pdf_path):
    reader = PdfReader(pdf_path)
    results = []
    for page_num, page in enumerate(reader.pages):
        for annot in page.annotations or []:
            annot = annot.get_object()
            results.append({
                "page": page_num + 1,
                "type": annot.get("/Subtype", ""),
                "content": annot.get("/Contents", ""),
                "rect": list(annot.get("/Rect", [])),
            })
    return results
```

### Add a text annotation (sticky note)

```python
from pypdf import PdfReader, PdfWriter
from pypdf.generic import (
    AnnotationBuilder, ArrayObject, FloatObject, NameObject,
    NumberObject, createStringObject, DictionaryObject,
)

def add_text_annotation(input_pdf, output_pdf, page_num, rect, content, author=""):
    """rect = (x1, y1, x2, y2) in PDF user space (bottom-left origin)."""
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    writer.append(reader)

    annot = DictionaryObject({
        NameObject("/Type"):     NameObject("/Annot"),
        NameObject("/Subtype"):  NameObject("/Text"),
        NameObject("/Rect"):     ArrayObject([FloatObject(v) for v in rect]),
        NameObject("/Contents"): createStringObject(content),
        NameObject("/T"):        createStringObject(author),
        NameObject("/Open"):     NameObject("/false"),
    })
    writer.add_annotation(page_number=page_num, annotation=annot)

    with open(output_pdf, "wb") as f:
        writer.write(f)

# Add a note at position (50, 700, 200, 750) on page 0
add_text_annotation("input.pdf", "output.pdf", page_num=0,
                     rect=(50, 700, 200, 750), content="Review this section", author="Alice")
```

### Add a highlight annotation

```python
from pypdf import PdfReader, PdfWriter
from pypdf.generic import (
    ArrayObject, FloatObject, NameObject, NumberObject,
    DictionaryObject, createStringObject,
)

def add_highlight(input_pdf, output_pdf, page_num, rect, color=(1, 1, 0)):
    """rect = (x1, y1, x2, y2). color = RGB tuple 0-1."""
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    writer.append(reader)

    annot = DictionaryObject({
        NameObject("/Type"):    NameObject("/Annot"),
        NameObject("/Subtype"): NameObject("/Highlight"),
        NameObject("/Rect"):    ArrayObject([FloatObject(v) for v in rect]),
        NameObject("/QuadPoints"): ArrayObject([
            FloatObject(rect[0]), FloatObject(rect[3]),
            FloatObject(rect[2]), FloatObject(rect[3]),
            FloatObject(rect[0]), FloatObject(rect[1]),
            FloatObject(rect[2]), FloatObject(rect[1]),
        ]),
        NameObject("/C"): ArrayObject([FloatObject(c) for c in color]),
    })
    writer.add_annotation(page_number=page_num, annotation=annot)

    with open(output_pdf, "wb") as f:
        writer.write(f)
```

---

## Digital Signatures

Use `pyhanko` — the only actively maintained pure-Python PDF signing library.

```bash
pip install pyhanko pyhanko-certvalidator
```

### Self-signed certificate (for testing)

```bash
# Generate a self-signed cert + key (requires openssl)
openssl req -x509 -newkey rsa:2048 -keyout signer.key \
  -out signer.crt -days 365 -nodes \
  -subj "/CN=Test Signer/O=My Org"
# Bundle into PKCS#12
openssl pkcs12 -export -out signer.p12 \
  -inkey signer.key -in signer.crt -passout pass:secret
```

### Sign a PDF

```python
from pyhanko.sign import signers, fields
from pyhanko.sign.fields import SigFieldSpec
from pyhanko import stamp
from pyhanko_certvalidator import CertificateValidator
from pyhanko.pdf_utils.incremental_writer import IncrementalPdfFileWriter
from pyhanko.sign.signers.pdf_signer import PdfSignatureMetadata

def sign_pdf(input_pdf, output_pdf, p12_path, p12_password):
    signer = signers.SimpleSigner.load_pkcs12(p12_path, passphrase=p12_password.encode())

    with open(input_pdf, "rb") as f:
        writer = IncrementalPdfFileWriter(f)
        # Add a visible signature field if none exists
        fields.append_signature_field(writer, SigFieldSpec("Signature", on_page=0))
        meta = PdfSignatureMetadata(field_name="Signature")
        out = signers.sign_pdf(writer, meta, signer=signer)

    with open(output_pdf, "wb") as f:
        f.write(out.getvalue())

sign_pdf("document.pdf", "signed.pdf", "signer.p12", "secret")
```

### Verify a signature

```python
from pyhanko.sign.validation import validate_pdf_signature
from pyhanko.pdf_utils.reader import PdfFileReader

def verify_signatures(pdf_path):
    with open(pdf_path, "rb") as f:
        reader = PdfFileReader(f)
        for sig in reader.embedded_signatures:
            status = validate_pdf_signature(sig)
            print(f"Field: {sig.field_name}")
            print(f"  Intact:    {status.intact}")
            print(f"  Valid:     {status.valid}")
            print(f"  Signer CN: {status.signing_cert.subject.human_friendly}")
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

## OCR Scanned PDFs

### English (default)

```python
# pip install pytesseract pdf2image
import pytesseract
from pdf2image import convert_from_path

images = convert_from_path("scanned.pdf")
text = ""
for i, image in enumerate(images):
    text += f"Page {i+1}:\n"
    text += pytesseract.image_to_string(image)
    text += "\n\n"

print(text)
```

### Chinese (Simplified)

```bash
# Install Chinese language data first
apt-get install tesseract-ocr-chi-sim        # Linux
brew install tesseract-lang                  # macOS (includes all languages)
```

```python
import pytesseract
from pdf2image import convert_from_path

images = convert_from_path("scanned_chinese.pdf")
text = ""
for i, image in enumerate(images):
    text += f"Page {i+1}:\n"
    # lang='chi_sim' for Simplified Chinese; 'chi_tra' for Traditional; 'chi_sim+eng' for mixed
    text += pytesseract.image_to_string(image, lang="chi_sim")
    text += "\n\n"

print(text)
```

### Output searchable PDF (PDF/A with text layer)

```python
import pytesseract
from pdf2image import convert_from_path
from pathlib import Path

def ocr_to_searchable_pdf(input_pdf, output_pdf, lang="chi_sim+eng"):
    images = convert_from_path(input_pdf, dpi=300)
    pages_pdf = []
    for i, image in enumerate(images):
        page_path = f"/tmp/ocr_page_{i}.pdf"
        pdf_bytes = pytesseract.image_to_pdf_or_hocr(image, lang=lang, extension="pdf")
        with open(page_path, "wb") as f:
            f.write(pdf_bytes)
        pages_pdf.append(page_path)

    # Merge page PDFs
    from pypdf import PdfWriter, PdfReader
    writer = PdfWriter()
    for p in pages_pdf:
        writer.append(PdfReader(p))
    with open(output_pdf, "wb") as f:
        writer.write(f)

ocr_to_searchable_pdf("scanned.pdf", "searchable.pdf")
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
| Extract images | pdfimages | `pdfimages -j input.pdf prefix` |
| Compress | ghostscript | `gs ... -dPDFSETTINGS=/ebook -sOutputFile=out.pdf in.pdf` |
| Extract pages | qpdf | `qpdf input.pdf --pages . 1-5 -- out.pdf` |
| Merge | qpdf | `qpdf --empty --pages f1.pdf f2.pdf -- merged.pdf` |
| Rotate | qpdf | `qpdf input.pdf output.pdf --rotate=+90:1` |
| Remove password | qpdf | `qpdf --password=X --decrypt enc.pdf dec.pdf` |
| Split all pages | pdftk | `pdftk input.pdf burst` |
