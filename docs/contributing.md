# Publishing SOPs to the Website

This guide explains how to add new SOPs or update existing ones so they appear on the live site at [evilliers.github.io/agrp-sops](https://evilliers.github.io/agrp-sops).

Every time you push changes to GitHub, the site rebuilds automatically — no manual deployment needed.

---

## Prerequisites

- Git installed on your machine
- SSH access to the repository (`git@github.com:evilliers/agrp-sops.git`)
- The repository cloned locally at `~/work/SOPS_hosting/`

---

## Folder Structure

All SOP content lives inside the `docs/` folder, organised by topic:

```
docs/
├── index.md                      # Home page
├── about.md
├── hpc/                          # HPC-related SOPs
├── genome-assembly/              # Genome assembly SOPs
├── edna-metabarcoding/           # eDNA SOPs
└── data-management/              # Data management SOPs
```

Place new SOPs in the most relevant subfolder. Create a new subfolder if a new topic area is needed.

---

## Workflow A — Updating an Existing SOP

If you are editing a file that already appears on the site:

**Step 1 — Edit the file**

Open the `.md` file in your text editor and make your changes.

**Step 2 — Stage the file**

```bash
git add docs/path/to/your-file.md
```

**Step 3 — Commit with a message**

```bash
git commit -m "Update SOP: brief description of what changed"
```

**Step 4 — Push to GitHub**

```bash
git push origin master
```

The site will rebuild automatically within ~2 minutes.

---

## Workflow B — Adding a New SOP

**Step 1 — Create the file**

Create a new `.md` file in the appropriate `docs/` subfolder. Use lowercase filenames with hyphens, e.g. `my-new-sop.md`.

Start the file with a top-level heading:

```markdown
# Title of Your SOP

**SOP ID:** CATEGORY-XXX
**Version:** 1.0
**Date:** YYYY-MM-DD
**Author:** AGRP

---

## Overview
...
```

**Step 2 — Add it to the navigation**

Open `mkdocs.yml` and add an entry under the correct section in the `nav:` block:

```yaml
nav:
  - Home: index.md
  - HPC:
    - Getting Started: hpc/getting-started.md
    - Your New SOP: hpc/my-new-sop.md   # <-- add this line
```

!!! warning
    If you skip this step the file will exist on the server but **will not appear in the site navigation**.

**Step 3 — Stage both files**

```bash
git add docs/hpc/my-new-sop.md mkdocs.yml
```

**Step 4 — Commit and push**

```bash
git commit -m "Add SOP: title of new SOP"
git push origin master
```

---

## Checking the Deployment

After pushing, GitHub Actions runs the build. You can monitor it at:

**[github.com/evilliers/agrp-sops/actions](https://github.com/evilliers/agrp-sops/actions)**

A green tick means the site deployed successfully. A red cross means something went wrong — click the run to see the error log.

The live site is at: **[evilliers.github.io/agrp-sops](https://evilliers.github.io/agrp-sops)**

---

## Quick Reference

```bash
# Check what files have changed
git status

# Stage specific files
git add docs/hpc/my-sop.md mkdocs.yml

# Commit
git commit -m "Add/Update SOP: description"

# Push
git push origin master
```

---

## Tips

- **One commit per SOP** — keeps the history clean and easy to read.
- **Meaningful commit messages** — e.g. `Add SOP: ONT barcoding pipeline` rather than `update stuff`.
- **Preview locally before pushing** — if you have MkDocs installed, run `mkdocs serve` in the repo root and open `http://127.0.0.1:8000` to check formatting before pushing.
- **Images** — place any images in a `docs/assets/` folder and reference them with `![description](../assets/image.png)`.
