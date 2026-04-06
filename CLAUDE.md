# Tutorial Guidelines

The `tutorials/` folder contains step-by-step material that serves two purposes:
1. **Learning resource** — students read and follow the tutorials to understand how the project is built
2. **Build instructions** — when helping a student build the project, Claude should follow the tutorials as the source of truth for what to build and how

---

## For Students

Work through the tutorials in order. Each tutorial builds on the previous one. Do not skip ahead — later steps assume the earlier ones are complete.

---

## For Claude

### When helping a student build
- TBD

### When writing a new tutorial
- Name the file `XX-<topic>.md` (e.g., `01-project-setup.md`, `02-database-models.md`)
- Use sequential numbering and lowercase kebab-case
- Each tutorial must follow this structure:
  1. **Title & goal** — what the student will have built by the end
  2. **Prerequisites** — what must be completed first
  3. **Concepts introduced** — brief list of new ideas covered
  4. **Step-by-step content** — the main body
  5. **Summary** — recap and what comes next
- Write for students with basic Python knowledge but no prior FastAPI/SQLAlchemy experience
- Explain the *why* behind every decision, not just the *what*
- Introduce concepts before showing code
- Use `> **Note:**` blockquotes for important details or common mistakes
- Assume the code doesn't exist yet, and don't refer them to existing code files, as students will produce the code by themselves (Tutorials will be shared with students separate from the rest of the project.)
- Code in tutorials must match the actual project code exactly
- Try make the tutorial focused and concise to some extent without compromising on important details and information. 
