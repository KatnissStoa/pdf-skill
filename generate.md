# PDF Generation Guide

## Tool Selection

```
Need to create a PDF — what's the source?

- HTML/CSS layout  → weasyprint  (best CSS support, recommended default)
- Markdown / text  → pandoc      (TOC, headers, footnotes)
- Data / JSON      → reportlab   (programmatic, precise control)
- Simple text      → fpdf2       (lightweight, fast)
```

| Tool | Best For | CSS Support | Complexity |
|------|----------|-------------|------------|
| weasyprint | HTML to PDF | Excellent | Low |
| pandoc | Markdown to PDF | Via LaTeX | Medium |
| reportlab | Data to PDF | None | High |
| fpdf2 | Simple text | None | Low |

---

## Core Rules

### 1. Structure Before Style

```python
# CORRECT: semantic structure
html = """
<article>
  <header><h1>Report Title</h1></header>
  <section>
    <h2>Summary</h2>
    <p>Content...</p>
  </section>
</article>
"""

# WRONG: style-first approach
html = "<div style='font-size:24px'>Report Title</div>"
```

### 2. Handle Page Breaks Explicitly

```css
.new-page        { page-break-before: always; }
.keep-together   { page-break-inside: avoid; }
h2, h3           { page-break-after: avoid; }
```

### 3. Always Set Metadata

```html
<html>
<head>
  <title>Document Title</title>
  <meta name="author" content="Author Name">
</head>
```

### 4. Use Print-Optimized CSS

```css
@media print {
  body {
    font-family: 'Georgia', serif;
    font-size: 11pt;
    line-height: 1.5;
  }
  @page {
    size: A4;
    margin: 2cm;
  }
  .no-print { display: none; }
}
```

### 5. Validate Output

After generating any PDF:
1. Check file size (0 bytes = failed)
2. Open and verify page count
3. Verify fonts render correctly

---

## Chinese Font Handling

Default fonts in weasyprint, reportlab, and fpdf2 do not include Chinese glyphs — always specify a CJK font explicitly or Chinese characters will render as boxes.

### weasyprint

```python
from weasyprint import HTML, CSS

# Option 1: reference a system font by name (macOS/Linux)
css = CSS(string="""
  @font-face {
    font-family: 'NotoSansCJK';
    src: url('/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc');
  }
  body { font-family: 'NotoSansCJK', sans-serif; }
""")
HTML(string=html).write_pdf("output.pdf", stylesheets=[css])

# Option 2: reference a font file you ship with your project
css = CSS(string="""
  @font-face {
    font-family: 'MyFont';
    src: url('fonts/SourceHanSansSC-Regular.otf');
  }
  body { font-family: 'MyFont', sans-serif; }
""")
```

Common free CJK fonts: Noto Sans CJK SC, Source Han Sans SC, WenQuanYi Micro Hei.

Install on Linux:
```bash
apt-get install fonts-noto-cjk
```

### reportlab

```python
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.pdfgen import canvas

# Register font once before use
pdfmetrics.registerFont(TTFont('NotoSansCJK', 'NotoSansCJKsc-Regular.ttf'))

c = canvas.Canvas("output.pdf")
c.setFont('NotoSansCJK', 14)
c.drawString(50, 800, "你好，世界")
c.save()
```

With Platypus (document templates):
```python
from reportlab.platypus import SimpleDocTemplate, Paragraph
from reportlab.lib.styles import ParagraphStyle

style = ParagraphStyle('chinese', fontName='NotoSansCJK', fontSize=12)
doc = SimpleDocTemplate("output.pdf")
doc.build([Paragraph("你好，世界", style)])
```

### fpdf2

```python
from fpdf import FPDF

pdf = FPDF()
pdf.add_page()
pdf.add_font("NotoSans", fname="NotoSansCJKsc-Regular.ttf")
pdf.set_font("NotoSans", size=14)
pdf.cell(text="你好，世界")
pdf.output("output.pdf")
```

---

## weasyprint (Recommended Default)

```python
from weasyprint import HTML, CSS

# From string
html = "<h1>Hello</h1><p>World</p>"
HTML(string=html).write_pdf("output.pdf")

# From file
HTML("document.html").write_pdf("output.pdf")

# With custom CSS
css = CSS(string="body { font-family: Arial; }")
HTML(string=html).write_pdf("output.pdf", stylesheets=[css])
```

---

## pandoc (Markdown Source)

```bash
# Basic conversion
pandoc document.md -o output.pdf

# With table of contents
pandoc document.md --toc -o output.pdf

# Custom margins
pandoc document.md -V geometry:margin=1in -o output.pdf
```

---

## reportlab (Programmatic / Data)

### Basic Text

```python
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
from reportlab.lib.units import cm

c = canvas.Canvas("output.pdf", pagesize=A4)
width, height = A4

c.setFont("Helvetica-Bold", 24)
c.drawString(2*cm, height - 3*cm, "Report Title")

c.setFont("Helvetica", 12)
c.drawString(2*cm, height - 5*cm, "Body text here...")

c.save()
```

### Tables (TableStyle)

reportlab tables use a completely different API from text — use `Table` + `TableStyle`:

```python
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle
from reportlab.lib import colors
from reportlab.lib.pagesizes import A4

data = [
    ["Name",  "Department", "Salary"],   # header row
    ["Alice", "Engineering", "$120,000"],
    ["Bob",   "Marketing",   "$95,000"],
    ["Carol", "Design",      "$105,000"],
]

style = TableStyle([
    # Header background
    ("BACKGROUND",  (0, 0), (-1, 0),  colors.HexColor("#4472C4")),
    ("TEXTCOLOR",   (0, 0), (-1, 0),  colors.white),
    ("FONTNAME",    (0, 0), (-1, 0),  "Helvetica-Bold"),
    # Body rows — alternating color
    ("ROWBACKGROUNDS", (0, 1), (-1, -1), [colors.white, colors.HexColor("#EEF2FF")]),
    # Grid
    ("GRID",        (0, 0), (-1, -1), 0.5, colors.grey),
    # Padding
    ("TOPPADDING",  (0, 0), (-1, -1), 6),
    ("BOTTOMPADDING", (0, 0), (-1, -1), 6),
    ("LEFTPADDING", (0, 0), (-1, -1), 8),
    # Alignment
    ("ALIGN",       (2, 1), (-1, -1), "RIGHT"),   # right-align salary column
])

table = Table(data, colWidths=[150, 150, 100])
table.setStyle(style)

doc = SimpleDocTemplate("table_report.pdf", pagesize=A4)
doc.build([table])
```

Column width tips:
- `colWidths` must be specified in points (1 cm ≈ 28.35 pt) or use `*` to auto-distribute
- If total `colWidths` exceed page width minus margins, content will overflow silently

### Multi-Column Layout

reportlab does not support CSS columns natively — use `Frame` + `PageTemplate` to define column regions:

```python
from reportlab.platypus import SimpleDocTemplate, Paragraph, NextPageTemplate, PageBreak
from reportlab.platypus.frames import Frame
from reportlab.platypus import PageTemplate
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib.pagesizes import A4
from reportlab.lib.units import cm

PAGE_W, PAGE_H = A4
MARGIN = 2 * cm

def two_column_template(doc):
    col_gap = 0.5 * cm
    col_w = (PAGE_W - 2 * MARGIN - col_gap) / 2
    col_h = PAGE_H - 2 * MARGIN

    left  = Frame(MARGIN,              MARGIN, col_w, col_h, id="left")
    right = Frame(MARGIN + col_w + col_gap, MARGIN, col_w, col_h, id="right")
    return PageTemplate(id="TwoCol", frames=[left, right])

doc = SimpleDocTemplate("two_col.pdf", pagesize=A4,
                        leftMargin=MARGIN, rightMargin=MARGIN,
                        topMargin=MARGIN, bottomMargin=MARGIN)
doc.addPageTemplates([two_column_template(doc)])

styles = getSampleStyleSheet()
story = []
for i in range(10):
    story.append(Paragraph(f"Paragraph {i+1}: " + "Lorem ipsum dolor sit amet. " * 10,
                           styles["Normal"]))

doc.build(story)
```

Three-column: add a third `Frame` and adjust `col_w = (PAGE_W - 2*MARGIN - 2*col_gap) / 3`.

---

## fpdf2 (Simple / Lightweight)

```python
from fpdf import FPDF

pdf = FPDF()
pdf.add_page()
pdf.set_font("Helvetica", size=16)
pdf.cell(200, 10, text="Hello World", align="C")

pdf.set_font("Helvetica", size=12)
pdf.multi_cell(0, 10, text="Long paragraph text...")

pdf.output("output.pdf")
```

---

## Batch Generation

```python
from pathlib import Path
from weasyprint import HTML

def batch_generate(template_html, data_list, output_dir):
    Path(output_dir).mkdir(exist_ok=True)
    for item in data_list:
        html = template_html.format(**item['data'])
        HTML(string=html).write_pdf(f"{output_dir}/{item['filename']}.pdf")

template = """
<html><body>
<h1>Invoice for {client}</h1>
<p>Amount: ${amount}</p>
</body></html>
"""

data = [
    {"filename": "invoice-001", "data": {"client": "Acme", "amount": 1000}},
    {"filename": "invoice-002", "data": {"client": "Beta", "amount": 2000}},
]

batch_generate(template, data, "./invoices")
```

---

## Common Traps

| Trap | Consequence | Fix |
|------|-------------|-----|
| Missing CJK font | Chinese renders as boxes | Register/specify font explicitly |
| Missing fonts | Fallback to defaults | Use web-safe or bundled fonts |
| Absolute image paths | Images missing | Use relative paths |
| No page size set | Unpredictable layout | Set `@page { size: A4; }` |
| Large images | Huge file size | Compress before use |
| reportlab colWidths overflow | Content cut off silently | Sum colWidths ≤ page width − margins |
| Frame total width > page | Columns overlap | Re-calculate col_w after subtracting all gaps and margins |
