# Contributing

Code contributions go against the main repo:
[umersanii/Mimic](https://github.com/umersanii/Mimic) — see its
[CONTRIBUTING.md](https://github.com/umersanii/Mimic/blob/main/CONTRIBUTING.md) for the full
guide (workflow, code style, issue reporting).

## Contributing to *these docs*

This docs site is its own repo
([umersanii/Mimic-docs](https://github.com/umersanii/Mimic-docs)), built with
[MkDocs](https://www.mkdocs.org/) and the [Material theme](https://squidfunk.github.io/mkdocs-material/).

To preview changes locally:

```fish
python3 -m venv .venv
source .venv/bin/activate.fish
pip install -r requirements.txt
mkdocs serve
```

This opens a local server (default `http://127.0.0.1:8000`) that live-reloads as you edit files
under `docs/`.

Each page has an "Edit this page" link (top right) that jumps straight to the source file on
GitHub if you spot something wrong while reading.

### Adding a new page

1. Create the `.md` file under `docs/`.
2. Add it to the `nav:` list in `mkdocs.yml`, or it won't show up in the sidebar.
3. Keep the same tone as the rest of the site: a plain-language explanation first, technical
   detail after — see the existing pages for the pattern.
