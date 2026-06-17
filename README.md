# ResXR Documentation

This repository contains the official documentation for the **ResXR Toolkit**, built with [Zensical](https://zensical.org) (a Material for MkDocs successor). The live site is at [docs.resxr.org](https://docs.resxr.org).

## Viewing locally

Clone the repo and start the live-reload server:

```bash
git clone https://github.com/ResXR/resxr-docs.git
cd resxr-docs
uv run zensical serve
```

Open [http://localhost:8000](http://localhost:8000). Any change to a file in `docs/` reloads automatically.

> **Prerequisite:** [uv](https://docs.astral.sh/uv/getting-started/installation/) — a fast Python package manager that installs all dependencies automatically.

## Contributing

Every page on the live site has a **pencil icon** (✏️) in the top-right corner that links directly to the source file on GitHub. To propose a change:

### All changes go through `dev`

`main` serves the live site and is protected. Open all PRs against the `dev` branch; maintainers promote `dev → main` on release.

### Option A — Fork (no write access needed)

```bash
# 1. Fork at github.com/ResXR/resxr-docs, then:
git clone https://github.com/<your-username>/resxr-docs.git
cd resxr-docs
git remote add upstream https://github.com/ResXR/resxr-docs.git

# 2. Work on dev
git checkout dev

# 3. Edit files under docs/, then:
git add docs/path/to/file.md
git commit -m "docs: describe the change"
git push origin dev
```

Then open a PR on GitHub — set **base repository** to `ResXR/resxr-docs` and **base branch** to `dev`.

### Option B — Direct push (maintainers with Write access)

```bash
git clone https://github.com/ResXR/resxr-docs.git
cd resxr-docs
git checkout dev
# ... edit, commit, push to origin dev ...
```

## Repository permissions

| Role | How to get it |
| ---- | ------------- |
| **Write** (maintainer) | An Admin goes to **Settings → Collaborators and teams → Add people**, sets role to **Write**, and sends the invite. Accept via email. |
| **Admin** | Granted by an existing Admin. Required to change branch protection rules or manage collaborators. |
| **Fork contributor** | No permission needed — fork the repo and open a PR from your fork. |

### Branch protection (`main`)

`main` has push protection enabled — direct pushes and force-pushes are blocked. To adjust rules, go to **Settings → Branches → `main`** (Admin access required).

### Deployment permissions

The site deploys automatically via GitHub Actions when `main` is updated. The workflow uses the built-in `GITHUB_TOKEN` — no extra secret is needed. If a deployment job fails with a permissions error, go to **Settings → Actions → General → Workflow permissions** and set it to **Read and write permissions**.
