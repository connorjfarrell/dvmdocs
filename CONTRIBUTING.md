# Contributing

Full contributor guide (build instructions, layout, style): see
[`dvmdocs/contributing.md`](dvmdocs/contributing.md), which is also published as the
**Contributing** page of the book.

## Quick start

```bash
python3 -m venv .venv
.venv/bin/pip install -r dvmdocs/requirements.txt
.venv/bin/jupyter-book build dvmdocs/
```

> This book is **Jupyter Book v1**. `requirements.txt` pins `jupyter-book<2` — do not upgrade
> past v1 without migrating `_config.yml` / `_toc.yml`.

A page skeleton to copy is in [`PAGE_TEMPLATE.md`](PAGE_TEMPLATE.md).
