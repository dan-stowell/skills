---
name: skills-authoring-essentials
description: Provides practical guidance for writing effective skills files that AI agents can discover and use successfully. Use when creating or improving skills files, or when the user needs help with skills authoring best practices, structure, or patterns.
---

# Skills Authoring Essentials

A practical guide for writing effective skills files that AI agents can discover and use successfully.

## Core Principles

### 1. Be Concise

Context windows are shared resources. Your skill competes with conversation history, other skills, and the actual request.

**Key question**: Does the AI really need this information, or can you assume it already knows?

**Good** (50 tokens):
```markdown
## Extract PDF text

Use pdfplumber for text extraction:

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
```

**Bad** (150 tokens):
```markdown
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text, images, and other content. To extract text from a PDF, you'll need to
use a library. There are many libraries available...
```

### 2. Match Specificity to Task Fragility

**High Freedom** (text instructions):
- Use when multiple approaches are valid
- Example: Code reviews, content analysis

```markdown
1. Analyze the code structure
2. Check for potential bugs
3. Suggest improvements for readability
```

**Medium Freedom** (pseudocode with parameters):
- Use when a preferred pattern exists
- Example: Report generation with options

```python
def generate_report(data, format="markdown", include_charts=True):
    # Process data
    # Generate output in specified format
```

**Low Freedom** (exact scripts):
- Use when operations are fragile or order-critical
- Example: Database migrations, system configurations

```bash
python scripts/migrate.py --verify --backup
# Do not modify the command or add additional flags
```

**Think of it as**: Narrow bridge (low freedom) vs. open field (high freedom).

### 3. Test with Target Models

Stronger models may need less explanation; weaker models may need more detail. Test your skill with all models you plan to use and aim for instructions that work across your target range.

## Structure

### Metadata

Your skill needs a clear name and description:

**Name**:
- Use descriptive, action-oriented names
- Good: `processing-pdfs`, `analyzing-spreadsheets`, `managing-databases`
- Avoid: `helper`, `utils`, `tools`

**Description**:
- Include both WHAT it does and WHEN to use it
- Write in third person
- Be specific with key terms
- Good: "Processes Excel files and generates formatted reports. Use when analyzing spreadsheet data or creating business summaries."
- Bad: "I can help you with Excel files"

### Content Organization

**Progressive Disclosure**: Structure information so details are loaded only when needed.

**Simple case** (everything in one file):
```markdown
## Data Analysis

Use pandas for data processing:
[core instructions here]
```

**Complex case** (split across files):
```markdown
## Data Analysis

For standard operations, see the core workflow below.
For advanced features, see `reference/advanced-operations.md`.

### Core Workflow
[essential instructions here]
```

**Guidelines**:
- Keep main file under 500 lines
- Split reference material into separate files
- Use descriptive filenames: `form_validation_rules.md` not `doc2.md`
- Organize by domain: `reference/finance.md`, `reference/sales.md`

## Effective Patterns

### Provide Working Scripts

Don't ask the AI to write validation logic—provide it.

**Good**:
```markdown
Run the validation script:
```bash
python scripts/validate_form.py input.json
```

Creates: `validation_report.txt` with any errors found
```

**Bad**:
```markdown
Write a script to validate the form data
```

### Use Concrete Examples

**Good**:
```markdown
## Field Mapping

Map spreadsheet columns to database fields:
- Column "Customer Name" → field `customer_name`
- Column "Order Total" → field `order_total`
```

**Bad**:
```markdown
## Field Mapping

Map spreadsheet columns to appropriate database fields
```

### Create Verifiable Workflows

For complex operations, use a plan-validate-execute pattern:

```markdown
## Batch Update Process

1. Analyze current state: `python analyze.py data.xlsx`
2. Generate change plan: Creates `changes.json`
3. Validate plan: `python validate.py changes.json`
4. Execute changes: `python apply.py changes.json`
5. Verify results: `python verify.py output/`
```

**Why this works**: Catches errors before changes are applied, provides clear debugging, allows iteration on plans.

### Handle Dependencies Explicitly

Don't assume packages are installed:

**Good**:
```markdown
Install required package:
```bash
pip install pypdf
```

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```
```

**Bad**:
```markdown
Use the pdf library to process the file.
```

### Leverage Visual Analysis

When inputs can be rendered as images (PDFs, diagrams, forms), have the AI analyze them visually to understand layout and structure.

### Avoid Time-Sensitive Information

Don't include API versions, URLs, or patterns that will become outdated. If you must, clearly mark them as potentially outdated and provide verification steps.

## Quality Checklist

Before finalizing a skill:

**Core Quality**
- [ ] Description includes both what and when
- [ ] Main file under 500 lines
- [ ] Assumes AI intelligence—no over-explaining
- [ ] Examples are concrete, not abstract
- [ ] Consistent terminology throughout

**Code & Scripts**
- [ ] Scripts solve problems (don't punt to AI)
- [ ] Error messages are specific and helpful
- [ ] Required packages explicitly listed
- [ ] Critical operations have validation steps
- [ ] No unexplained "magic numbers"

**Testing**
- [ ] Tested with all target models
- [ ] Verified with real usage scenarios
- [ ] Error cases handled gracefully

## Common Pitfalls

1. **Over-explaining**: The AI likely knows what a PDF is
2. **Under-specifying critical operations**: "Be careful with migrations" isn't enough
3. **Assuming tool availability**: Always state dependencies
4. **Vague examples**: Show specific input/output, not abstractions
5. **Mixed point-of-view**: Stay in third person
6. **Monolithic files**: Split large skills into digestible pieces

## Key Takeaway

Write skills files as if you're leaving instructions for a highly capable colleague who might not know your specific domain. Be concise where they'd naturally understand, be specific where mistakes are costly, and provide the actual tools they need rather than asking them to create everything from scratch.