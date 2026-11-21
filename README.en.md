[简体中文](README.md) | English

# Silica Documentation

Bilingual documentation site for **Silica**, an embedded declarative UI framework for LeviLamina Minecraft Bedrock Edition client mods. The framework source lives in the [Silica](https://github.com/wuqinlanjian/Silica) repository.

Live site: <https://silica.feather.qzz.io/> (Chinese) · <https://silica.feather.qzz.io/en/> (English).

The site is built with MkDocs and the Material theme. Chinese is served at `/` and English under `/en/`; the language selector in the header switches the current page directly.

## Contents

| Chapter | What it covers |
| --- | --- |
| Overview | Architecture, render flow, and the core concepts |
| Getting started | Environment, build, deployment, and a minimal integration |
| DSL tutorial | Syntax, layout, widgets, bindings, scripts, components, styles, animation, lists, images |
| Widget reference | Built-in widgets and their commonly used properties |
| C++ integration | Loading documents, callbacks, `DataStore`, diagnostics |
| Troubleshooting | Reading diagnostics and fixing common symptoms |
| Examples | Settings panels, dialogs, dynamic lists, toggle buttons |

## Local preview

```bash
python -m pip install -r requirements.txt
python -m mkdocs serve
```

The preview server usually prints `http://127.0.0.1:8000/`. Changes to `.md` files and `mkdocs.yml` rebuild the site automatically.

## Build

```bash
python -m mkdocs build
```

The output goes to `site/`: the Chinese site at the root and the English site under `site/en/`. The repository has no automatic deployment workflow; the result is plain static files that can be hosted anywhere.

## Deployment

The site is published at <https://silica.feather.qzz.io/> (Chinese) and <https://silica.feather.qzz.io/en/> (English) with Cloudflare Pages. Connect this repository in the dashboard with:

| Setting | Value |
| --- | --- |
| Build command | `python -m pip install -r requirements.txt && python -m mkdocs build` |
| Build output directory | `site` |
| Root directory | repository root (leave empty) |

If the build image ships an older Python, add the environment variable `PYTHON_VERSION=3.12`.

## Adding a page

1. Add `docs/<name>.md` (Chinese) and `docs/<name>.en.md` (English) with matching headings and anchors.
2. Register the Chinese file in `nav` in `mkdocs.yml`; the i18n plugin resolves the `.en.md` counterpart automatically.
3. Add the title to `nav_translations` under the `en` locale so the English navigation stays translated.

## License

MIT. See [LICENSE](LICENSE).
