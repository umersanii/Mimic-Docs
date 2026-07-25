# Mimic-Docs

Documentation site for [umersanii/Mimic](https://github.com/umersanii/Mimic), a
webcam-controlled, tendon-driven robotic hand.

Built with [MkDocs](https://www.mkdocs.org/) + the [Material theme](https://squidfunk.github.io/mkdocs-material/),
deployed to GitHub Pages.

## Local preview

```fish
python3 -m venv .venv
source .venv/bin/activate.fish
pip install -r requirements.txt
mkdocs serve
```

Then open `http://127.0.0.1:8000`.

## Deploying

Pushes to `main` build and publish automatically via `.github/workflows/deploy.yml`. To publish
by hand instead:

```fish
mkdocs gh-deploy
```

## License

Docs content is [MIT](LICENSE)-licensed, same as the main project.
