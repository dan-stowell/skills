# Gemini Skill Authoring Best Practices (Condensed)

This guide provides a condensed version of best practices for writing effective Skills that an agent can discover and use successfully.

## Core Principles

### 1. Be Concise

The context window is a shared resource. Only add context the agent doesn't already have. Assume the agent is smart.

**Good Example (Concise):**
```markdown
## Extract PDF text

Use pdfplumber for text extraction:

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
```

**Bad Example (Verbose):**
```markdown
## Extract PDF text

PDF (Portable Document Format) files are a common file format... To extract text from a PDF, you'll need to use a library... we recommend pdfplumber...
```

### 2. Set Appropriate Degrees of Freedom

Match the level of specificity to the task's fragility.

*   **High Freedom (Instructions):** For tasks with multiple valid approaches (e.g., code review).
*   **Medium Freedom (Pseudocode/Templates):** When a preferred pattern exists but some variation is acceptable (e.g., report generation).
*   **Low Freedom (Specific Scripts):** For fragile operations where consistency is critical (e.g., database migrations).

### 3. Test with Different Models

Skills behave differently depending on the underlying model. Test your Skill with the range of models you plan to use it with (e.g., both stronger and weaker models) to ensure it provides the right level of guidance for each.

## Skill Structure

A skill is defined in a `SKILL.md` file with YAML frontmatter.

### YAML Frontmatter

```yaml
name: a-unique-skill-name
description: A clear, concise description of what the skill does and when to use it.
```

*   `name`: Max 64 chars, lowercase letters, numbers, and hyphens only.
*   `description`: Max 1024 chars. Should be written in the third person (e.g., "Processes Excel files..."). Be specific and include keywords that would trigger the skill.

### Naming Conventions

Use consistent naming. Gerunds (`verb-ing`) are recommended (e.g., `processing-pdfs`, `analyzing-spreadsheets`).

### Progressive Disclosure

Keep `SKILL.md` as a concise overview that points to more detailed information in other files as needed. This saves context window space.

*   Keep `SKILL.md` body under 500 lines.
*   Split large content into separate files (e.g., `EXAMPLES.md`, `REFERENCE.md`).
*   Link to these files from `SKILL.md`.

**Example `SKILL.md`:**
```markdown
---
name: pdf-processing
description: Extracts text, tables, and fills forms in PDF files.
---

# PDF Processing

## Quick start
Extract text with pdfplumber:
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## Advanced features
- **Form filling**: See [FORMS.md](FORMS.md) for a complete guide.
- **API reference**: See [REFERENCE.md](REFERENCE.md) for all methods.
```

*   **Avoid deep nesting:** All reference files should be linked directly from `SKILL.md`.
*   **Use a table of contents:** For reference files over 100 lines, add a table of contents at the top.

## Workflows and Feedback Loops

### Use Workflows for Complex Tasks

Break complex operations into sequential steps. A checklist helps the agent track progress.

**Example Workflow:**
```markdown
## PDF Form Filling Workflow

Copy this checklist and check off items as you complete them:

```
Task Progress:
- [ ] Step 1: Analyze the form (run analyze_form.py)
- [ ] Step 2: Create field mapping (edit fields.json)
- [ ] Step 3: Validate mapping (run validate_fields.py)
- [ ] Step 4: Fill the form (run fill_form.py)
```

**Step 1: Analyze the form**
Run: `python scripts/analyze_form.py input.pdf`
This extracts form fields and their locations, saving to `fields.json`.

... (details for other steps) ...
```

### Implement Feedback Loops

Use a `validate -> fix -> repeat` pattern to improve output quality. This is especially useful for code.

**Example:**
```markdown
## Document Editing Process

1. Make your edits to `word/document.xml`.
2. **Validate immediately**: `python ooxml/scripts/validate.py unpacked_dir/`
3. If validation fails, review the error, fix the XML, and run validation again.
4. **Only proceed when validation passes.**
5. Rebuild the document.
```

## Content Guidelines

*   **Avoid time-sensitive info:** Don't mention dates or versions that will become outdated. Use an "Old patterns" section for deprecated info.
*   **Use consistent terminology:** Choose one term for a concept and stick to it (e.g., always "API endpoint", not "URL" or "route").

## Common Patterns

*   **Template Pattern:** Provide templates for required output formats.
*   **Examples Pattern:** Provide clear input/output examples, especially for formatting tasks like commit messages.
*   **Conditional Workflow Pattern:** Guide the agent through decision points (e.g., "If creating new content, follow workflow A. If editing, follow workflow B.").

## Evaluation and Iteration

*   **Evaluation-Driven Development:** Before writing a big skill, identify tasks the agent fails at. Create evaluations for these tasks, then write the minimal skill needed to pass them.
*   **Iterate with the Agent:**
    1.  Work with an agent instance to solve a task manually. Note the information you provide.
    2.  Ask the agent to create a Skill based on that interaction.
    3.  Refine the Skill for conciseness and clarity.
    4.  Test the new Skill with a *different* agent instance on similar tasks.
    5.  Observe where the second agent struggles and use those insights to improve the Skill.

## Anti-Patterns to Avoid

*   **Windows-style paths:** Always use forward slashes (`/`) in file paths.
*   **Too many options:** Provide a clear default or recommended tool, not a long list of alternatives.

## Skills with Executable Code

*   **Solve, Don't Punt:** Scripts should handle common errors gracefully instead of failing and letting the agent figure it out.
*   **Provide Utility Scripts:** Pre-made scripts are more reliable, faster, and save tokens compared to generating code from scratch. Make it clear whether a script should be executed or read as a reference.
*   **Verifiable Intermediate Outputs:** For complex, multi-step tasks, have the agent generate a plan (e.g., a JSON file), validate it with a script, and only then execute it.
*   **Package Dependencies:** List required packages (e.g., via `pip` or `npm`) in your instructions. Don't assume tools are pre-installed.
*   **Tool Naming:** When calling tools, use their fully qualified names (e.g., `ServerName:tool_name`) to avoid ambiguity.

## Final Checklist

- [ ] Is the description specific, in the third person, and does it include trigger keywords?
- [ ] Is the `SKILL.md` file concise (< 500 lines)?
- [ ] Are complex details moved to separate, linked files?
- [ ] Are workflows broken into clear steps?
- [ ] Are feedback loops (e.g., validation scripts) included for critical tasks?
- [ ] Are code dependencies listed?
- [ ] Has the skill been tested with different models and real-world scenarios?
