# Copilot Instructions

## Repository Purpose

This is a documentation and guide repository for n8n — an open-source workflow automation tool. It covers self-hosting n8n (locally or on Oracle Cloud VM) and includes hands-on example projects with importable workflow files.

## Structure

```
README.md               # Main guide: n8n overview + Oracle Cloud VM + local setup
Projects/
  projects.md           # Index linking to all example projects
  project{N}/
    readme.md           # Project explanation, concepts, screenshots
    *.json              # Exportable n8n workflow file(s)
.imgs/                  # All workflow screenshots & demo images for every project (central folder, not per-project)
```

## Content Conventions

### Projects
Each project lives in `Projects/project{N}/` and follows this structure:
- `readme.md` — explains the use case, n8n concepts demonstrated, step-by-step setup, and embeds screenshots
- One or more `.json` workflow files — exported directly from n8n (File → Download) for readers to import
- `readme.md` files embed images from the repo-root `.imgs/` folder only — one canonical place for every project’s canvas/screenshots (e.g. `../../.imgs/demo1.png` from `Projects/project1/readme.md`)

### Markdown
- Main `README.md` uses `## table of contents` (lowercase) with anchor links
- Section headers use `##` and `###`; inline commands use fenced code blocks with `bash`
- Notes and warnings use blockquotes (`>`)

### n8n Workflow JSON
- Workflow files are exported from n8n as JSON and committed as-is — do not hand-edit them
- When adding a new project, export the workflow from n8n and save it in the project directory

### Adding a New Project
1. Create `Projects/project{N}/` directory
2. Add the workflow JSON exported from n8n
3. Add screenshots to `.imgs/` following the `demo{N}.png` / `demo{N}_{variant}.png` naming pattern (e.g. `demo1.png`, `demo2.png`, `demo3.png`)
4. Write `Projects/project{N}/readme.md` explaining the use case and concepts
5. Add a link to the new project in `Projects/projects.md`
