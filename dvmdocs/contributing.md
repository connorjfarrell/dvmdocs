# Contributing to the Documentation

This site is a [Jupyter Book](https://jupyterbook.org/) built from Markdown (MyST) files in
the [`DVMProject/dvmdocs`](https://github.com/DVMProject/dvmdocs) repository. Corrections and
new content are welcome.

```{note}
Much of the current content was drafted with AI assistance from the upstream project
repositories. Treat it as a starting point: verify against the relevant repo before relying
on any detail, and fix anything that's wrong or stale — that's exactly the kind of
contribution this project needs.
```

## Build it locally

```{important}
This book uses **Jupyter Book v1** (Sphinx-based). Jupyter Book v2 (`mystmd`) does **not**
read this repo's `_config.yml` / `_toc.yml`. Always pin `jupyter-book<2`.
```

```bash
git clone https://github.com/DVMProject/dvmdocs.git
cd dvmdocs
python3 -m venv .venv
.venv/bin/pip install -r dvmdocs/requirements.txt
.venv/bin/jupyter-book build dvmdocs/
```

Open `dvmdocs/_build/html/index.html`, or serve it (needed for search):

```bash
python3 -m http.server 8000 --directory dvmdocs/_build/html
```

Rebuild after edits. Use `jupyter-book build dvmdocs/ --all` to force a full rebuild if
cross-references look stale.

## Repository layout

```text
dvmdocs/
├── _config.yml        # book settings, Sphinx/MyST config
├── _toc.yml           # table of contents — add new pages here
├── requirements.txt   # build dependencies
├── intro.md           # landing page
├── overview/          # what DVMProject is, hardware/software, glossary
├── software/          # per-application reference
├── hardware/          # per-board reference
├── guides/            # task-oriented how-tos
└── media/             # images
```

## Adding a page

1. Create the `.md` file in the appropriate folder.
2. Add it to `_toc.yml` under the right `part` / `sections`.
3. Rebuild and check it renders with no new warnings.

## Style conventions

- One `#` H1 per page, matching (roughly) the `_toc.yml` position. Sentence-case headings.
- Use admonitions for asides: `{note}`, `{warning}`, `{important}`, `{tip}` (colon-fence
  syntax works: ` :::{note} ... ::: `).
- Code and config in fenced blocks with a language tag (`bash`, `yaml`, `text`). Keep config
  snippets short (≈15 lines); link the full upstream file instead of pasting it.
- **Cite the source.** End any page derived from a project repo with a
  *Source / further reading* list linking the upstream README, example config, or technical
  note. Prefer linking canonical upstream files over duplicating them here (they drift).
- Internal links are relative: `[text](../software/dvmhost/dvmhost.md)` or
  `[text](dvmhost.md#calibration)`.
- Cross-link glossary terms with `` {term}`FNE` `` where it aids a new reader.

## Accuracy

DVM changes quickly. If you can't verify something against a current repo, mark it — e.g. a
`{note}` saying "confirm against your `dvmhost` version" — rather than stating it flatly.
Version-specific strings (release tags, tarball names) should be written as examples.

## Diagrams

Use [Mermaid](https://mermaid.js.org/) fenced blocks (` ```{mermaid} `) so diagrams stay in
version control as text. Only add binary images to `media/` when a photo or schematic is
genuinely needed.
