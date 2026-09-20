# paceuniversity-cs371
Pace University CS371

## How to contribute
The official project site is [https://aseriy.github.io/paceuniversity-cs371/](https://aseriy.github.io/paceuniversity-cs371/). It is updated only by the project/repo coordinator.

1. Work on your own branch, named after your GitHub username.
2. Make your documentation updates on that branch and test them locally (see [Building the docs locally](#building-the-docs-locally)).
3. Submit a PR when ready.
4. Once the PR is merged, the repo coordinator publishes the site.

## Building the docs locally
Test your documentation changes locally before raising a PR. You need Python 3 with pip (the site is maintained with Python 3.13.7).

Create and activate a virtual environment in the repository root, then install the toolchain:

macOS/Linux:
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Windows:
```
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

Preview the site with live reload:

```bash
mkdocs serve
```

Open `http://127.0.0.1:8000/` in your browser — it reloads automatically as you edit files under `docs/`.

Notes:

- The built `site/` directory is git-ignored; commit only your `docs/` changes, never `site/`.
- On Windows, if a command isn't found, prefix it with `python -m` (e.g. `python -m mkdocs serve`).
