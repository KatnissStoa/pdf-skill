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

Multi-page with Platypus:

```python
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, PageBreak
from reportlab.lib.styles import getSampleStyleSheet

doc = SimpleDocTemplate("report.pdf", pagesize=letter)
styles = getSampleStyleSheet()
story = []

story.append(Paragraph("Report Title", styles['Title']))
story.append(Spacer(1, 12))
story.append(Paragraph("Body content " * 20, styles['Normal']))
story.append(PageBreak())
story.append(Paragraph("Page 2", styles['Heading1']))

doc.build(story)
```

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
| Missing fonts | Fallback to defaults | Use web-safe fonts |
| Absolute image paths | Images missing | Use relative paths |
| No page size set | Unpredictable layout | Set `@page { size: A4; }` |
| Large images | Huge file size | Compress before use |
