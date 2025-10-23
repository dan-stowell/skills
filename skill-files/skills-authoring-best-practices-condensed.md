---
name: skills-authoring-best-practices-condensed
description: Provides a concise, model-agnostic reference for writing and refining skill files, covering structure, degrees of freedom, and evaluation checklists. Use when an agent needs quick guidance on skill authoring best practices without vendor-specific dependencies.
---

Here’s a succinct, model-agnostic “Skills Authoring Best Practices” summary that captures the essence of the long file — tuned for any strong or weak LLM or agent:

⸻

Skills Authoring Best Practices (Concise Version)

1. Purpose

A Skill teaches an agent how to perform a reusable task. A good Skill is concise, structured, testable, and efficient with context. It should help the agent act reliably and minimize unnecessary explanation.

⸻

2. Core Principles

Conciseness

Keep Skills short and information-dense.
Ask of every line:
	•	Does the model already know this?
	•	Is this necessary for correct execution?
Favor minimal examples over lengthy exposition.

Degrees of Freedom

Match specificity to task fragility:
	•	High freedom – conceptual guidance for creative work.
	•	Medium freedom – templates or pseudocode with parameters.
	•	Low freedom – exact commands for fragile or critical operations.

Model Adaptation

Different models vary in reasoning power.
Write Skills that weaker models can still follow while avoiding over-explaining for stronger ones. Test with both.

⸻

3. Structure

Frontmatter

Each SKILL.md begins with:

name: lowercase-hyphen-name
description: What the skill does and when to use it

The description must be third-person, specific, and include triggers (keywords or scenarios).

Naming

Prefer gerund or action forms:
processing-pdfs, analyzing-data, writing-docs.

Progressive Disclosure

Keep SKILL.md under ~500 lines.
Link to other files (e.g. FORMS.md, reference.md, examples.md) that are loaded on demand.
References should be one level deep from SKILL.md.

File Organization

Organize by purpose or domain:

pdf/
  SKILL.md
  reference.md
  forms.md
  scripts/

Use descriptive filenames and forward slashes only.

⸻

4. Authoring Patterns

Workflows

Provide clear, numbered steps or checklists.
Use [ ] boxes to help the model track progress.

Feedback Loops

Encourage verify-and-retry cycles (e.g., “Run validator → fix errors → re-validate”).

Templates & Examples

Give output templates for strict formats and input/output examples for stylistic guidance.

Conditional Instructions

Branch by scenario:

“If creating new data, follow section A. If editing existing, follow section B.”

⸻

5. Writing Style & Content
	•	Avoid time-sensitive statements — label “Legacy methods” instead.
	•	Use consistent terminology (“field” not “box”).
	•	Prefer one recommended approach with optional fallback.
	•	Be explicit about dependencies (pip install ...).
	•	Include brief justifications for constants or parameters.

⸻

6. Code and Execution

When Skills include code:
	•	Handle errors explicitly rather than deferring to the model.
	•	Provide small, reusable scripts instead of inline generation.
	•	Clarify when to run vs read a script.
	•	Enable validation before destructive actions (plan-validate-execute).

⸻

7. Evaluation & Iteration

Build from Observed Needs

Start with real examples where the model failed, then write minimal instructions to fix those gaps.

Test, Observe, Refine

Use two roles:
	•	Authoring agent to refine the Skill
	•	Execution agent to apply it in real tasks
Iterate based on observed behavior and feedback.

Evaluation Files

Define simple tests describing expected behaviors and success criteria.

⸻

8. Anti-Patterns

Avoid:
	•	Redundant background explanations
	•	Deeply nested references
	•	Too many alternative methods
	•	Magic numbers or unclear constants
	•	Windows-style \ paths

⸻

9. Final Checklist

✅ Clear description of what and when
✅ Concise, consistent language
✅ Progressive disclosure used
✅ Concrete examples and templates
✅ Validation or feedback loops for fragile tasks
✅ Explicit dependencies and error handling
✅ Tested on both strong and weak models
✅ No time-sensitive or irrelevant content

⸻

This condensed guide captures the mindset:
Be concise, specific, modular, and test-driven.
A good Skill teaches the model exactly what it needs—no more, no less.
