# discord docs

The documentation site for [discord](https://github.com/loopy5418/discord), a Discord bot framework for [Sonata](https://sonata.5418.org).

Built with [MkDocs](https://www.mkdocs.org) and [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/). No JavaScript toolchain is involved: the pages are Markdown files in `docs/`, and the site is configured in `mkdocs.yml`.

## Working on the docs

You need Python 3.9 or newer.

```sh
python3 -m venv .venv
. .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements.txt

mkdocs serve                    # live preview on http://127.0.0.1:8000
mkdocs build --strict           # what CI runs; fails on broken links
```

## Layout

| Path | What it holds |
| --- | --- |
| `mkdocs.yml` | Site settings and the navigation |
| `docs/index.md` | The home page |
| `docs/getting-started/` | Installing, a first bot, project layout, building |
| `docs/guide/` | Topic guides: commands, components, events, ... |
| `docs/reference/` | Every module, function, field and option |
| `docs/examples/` | The bundled example bots and recipes |
| `docs/help/` | Troubleshooting, internals, glossary |
| `docs/assets/` | CSS and the favicon |
| `.github/workflows/deploy.yml` | Builds the site and publishes it to GitHub Pages |

New pages must be added to `nav` in `mkdocs.yml`.

## Publishing

Pushing to `main` builds the site and deploys it to GitHub Pages. Pull requests only build it, so broken links fail the check before merging.

One-time setup on GitHub: **Settings > Pages > Build and deployment > Source: GitHub Actions**. Then set `site_url` in `mkdocs.yml` to the Pages address (for example `https://<user>.github.io/<repo>/`).
