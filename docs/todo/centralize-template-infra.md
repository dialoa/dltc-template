# Plan: Centralize Dialectica Template Infrastructure

## Context

The Dialectica compilation pipeline is spread across Dropbox folders, DockerHub, and two GitHub orgs (Philosophie-ch and dialoa). Julien Dutant built this incrementally; it works but is fragile:

- Templates live in Dropbox with manual version folders (1.0, 1.1, 1.2, 1.3)
- Template assembly Makefile hardcodes Julien's Mac paths (`/Users/julien/GitHub/`)
- Docker images built manually, pushed to DockerHub (`philosophiech/dltc-env`)
- No CI, no container registry automation
- `dltc-make` inside the container hardcodes `template/1.2` path
- Template is ~300MB because of pandoc-crossref binaries (3 platforms)
- VenusSBOP font is **proprietary** (Elsner+Flake) → Docker image must remain private

This plan centralizes everything under the `dialoa` GitHub org, adds CI, and removes the Dropbox dependency for templates.

---

## Part 1: Transfer repos Philosophie-ch → dialoa

### Steps
1. GitHub Settings > Transfer repository for each:
   - `Philosophie-ch/dltc-env` → `dialoa/dltc-env`
   - `Philosophie-ch/dltc-env-dockerfiles` → `dialoa/dltc-env-dockerfiles`
2. GitHub auto-creates redirects from old URLs
3. Update local remotes:
   ```
   cd /home/alebg/philosophie-ch/dialectica/dltc-env
   git remote set-url origin https://github.com/dialoa/dltc-env.git
   cd /home/alebg/philosophie-ch/dialectica/dltc-env-dockerfiles
   git remote set-url origin https://github.com/dialoa/dltc-env-dockerfiles.git
   ```
4. Make `dltc-env-dockerfiles` **private** (it contains VenusSBOP proprietary font)
5. `dltc-env` can stay **public** (just docker-compose + start script)

### Breaking changes
- JD's local clones need remote update (redirects work temporarily)
- DockerHub image reference `philosophiech/dltc-env` → `ghcr.io/dialoa/dltc-env`

---

## Part 2: CI for dltc-env-dockerfiles

### Current state
- Two Dockerfiles: `Dockerfile.amd64`, `Dockerfile.arm64`
- Manual `build.sh` script, no CI
- Installs: Ubuntu Jammy, TeXLive (full), Pandoc 3.5, Quarto 1.4.544, Inkscape, librsvg2
- Fonts: Libertinus (OFL), STIX Two (OFL), VenusSBOP-BoldExtended (**proprietary**)
- Images currently pushed to DockerHub (`philosophiech/dltc-env:latest-{arch}`)
- `build-assets/template/` contains an **older, incomplete** copy of template 1.2 (missing 6 filters vs Dropbox version) — should be removed once template is baked from git

### New CI workflow
File: `.github/workflows/build-and-push.yml`

- **Trigger**: push to `main`, manual dispatch (workflow_dispatch)
- **Strategy**: single job using `docker buildx` with QEMU for cross-platform
  - Build is rare (only when updating Pandoc/TexLive), so emulation slowness is acceptable
  - dialoa is on free GitHub plan (2000 min/month)
- **Registry**: `ghcr.io/dialoa/dltc-env` (private package)
- **Tags**: `latest-amd64`, `latest-arm64`, `{date}-amd64`, `{date}-arm64`
- **Auth**: `GITHUB_TOKEN` (automatic in Actions)

### Consumer-side changes

Update `dltc-env/.env.template`:
```
ARCH="amd64"
DLTC_WORKHOUSE_DIRECTORY="/path/to/dltc-workhouse"
GHCR_TOKEN="github_pat_with_read_packages_scope"
```

Update `dltc-env/dltc-env-start.sh`:
- Login to `ghcr.io` instead of DockerHub
- Pull `ghcr.io/dialoa/dltc-env:latest-${ARCH}`

Update `dltc-env/docker-compose.yml`:
- `image: ghcr.io/dialoa/dltc-env:latest-${ARCH}`

### Dockerfile changes
- Remove `build-assets/template/` (stale copy) — template comes from git clone at build time
- Add git clone of public template repo:
  ```dockerfile
  ARG TEMPLATE_VERSION=v1.2
  RUN git clone --branch ${TEMPLATE_VERSION} --depth 1 \
      https://github.com/dialoa/dltc-template.git /opt/dltc-template
  ```
- Update `dltc-make` script:
  ```bash
  maker="/opt/dltc-template/scripts/make.lua"
  ```

### Files to modify
- `dltc-env-dockerfiles/.github/workflows/build-and-push.yml` (new)
- `dltc-env-dockerfiles/Dockerfile.amd64`
- `dltc-env-dockerfiles/Dockerfile.arm64`
- `dltc-env-dockerfiles/build-assets/scripts/dltc-make`
- `dltc-env-dockerfiles/build-assets/template/` (remove)
- `dltc-env/dltc-env-start.sh`
- `dltc-env/.env.template`
- `dltc-env/docker-compose.yml`
- `dltc-env/README.md`

---

## Part 3: New public template repo (dialoa/dltc-template)

### How make.lua resolves the template

1. **`dltc-make`** (baked in Docker) calls `pandoc lua <absolute-path>/scripts/make.lua`
2. **`make.lua`** has `findDialoa()` which searches for `template/{version}/dialoa.yaml` by walking up to 3 parent dirs from CWD — this is how it finds the template root when run from an article directory inside the workhouse
3. **YAML defaults files** use `${.}` (resolves to containing dir) — so `d_common.yaml`'s `template: ${.}/templates/dialectica` correctly resolves to `<template-root>/templates/dialectica`. The template is self-contained once found.
4. **`make.sh`** (shell wrapper) uses hardcoded relative paths `../../template/1.2/scripts/make.lua`

With the template baked into the image at `/opt/dltc-template/`, `dltc-make` calls make.lua directly by absolute path. `make.lua` still needs to resolve the template root for `${.}` references.

Approach: Modify `findDialoa()` to first check if make.lua's own parent directory contains `dialoa.yaml` (using `debug.getinfo` to self-locate). Falls back to current CWD-relative search behavior. This requires no argument changes and works in all contexts (Docker, local, mounted).

### Repo creation steps

1. Create empty `dialoa/dltc-template` (public)
2. Clone locally, add `.gitignore`:
   ```
   filters/pandoc-crossref
   filters/pandoc-crossref.exe
   filters/pandoc-crossref-nix
   csl/csl-bak/
   csl/bak/
   templates/package-lock.json
   EXTRAS/
   ```
3. Copy template v1.0 contents (flat, files at root), commit, tag `v1.0`
4. Replace with v1.1 contents, commit, tag `v1.1`
5. Replace with v1.2 contents, commit, tag `v1.2` ← this becomes `main` HEAD
6. Branch `dev` from `main`, replace with v1.3 contents, commit, push

```bash
git push origin main --tags
git push origin dev
```

### What to exclude
- `pandoc-crossref` binaries (~100MB each) — install in Docker image instead
- `csl/csl-bak/`, `csl/bak/` — manual backup junk
- `templates/package-lock.json` — Node artifact
- `EXTRAS/` — copyeditor helper scripts (move elsewhere if needed)

### Makefile update
Current `Makefile` hardcodes `/Users/julien/GitHub/`. Update to:
```makefile
GITHUB ?= $(HOME)/GitHub
```
This way it defaults to `~/GitHub` but is overridable: `make update GITHUB=/path/to/repos`.

### Relationship with filter repos
The repo stores assembled/built `.lua` files (not git submodules). The Makefile handles updating them from source repos.

### pandoc-crossref note
The binaries are platform-specific and ~100MB each. They are not currently installed in the Dockerfile — they come from the template folder. Move them into the Dockerfile (like Pandoc itself):
```dockerfile
ARG CROSSREF_VERSION=0.3.17.0
RUN wget https://github.com/lierdakil/pandoc-crossref/releases/download/v${CROSSREF_VERSION}/pandoc-crossref-Linux.tar.xz \
    && tar xf pandoc-crossref-Linux.tar.xz \
    && mv pandoc-crossref /usr/local/bin/ \
    && rm pandoc-crossref-Linux.tar.xz
```
Then update `paper-in-issue-nix.yaml` to reference `pandoc-crossref` (from PATH) instead of `${.}/filters/pandoc-crossref-nix`.

---

## Execution order

| Step | What | Depends on | Breaking? |
|------|------|------------|-----------|
| 1 | Create `dialoa/dltc-template`, import versions, push with tags | Nothing | No (additive) |
| 2 | Transfer repos to dialoa org | Coordinate with JD | URL changes |
| 3 | Make `dltc-env-dockerfiles` private | Step 2 | No |
| 4 | Add CI workflow to `dltc-env-dockerfiles` | Steps 2, 3 | No |
| 5 | Update Dockerfiles: clone template from git, install pandoc-crossref, remove build-assets/template | Steps 1, 4 | Rebuilds image |
| 6 | Update `dltc-env`: switch to GHCR, new auth | Steps 4, 5 | Consumers need new PAT |
| 7 | Update consumer docs | Step 6 | — |

Steps 1 and 2 can happen in parallel. The rest is sequential.

---

## What breaks for JD

| What | Current | New | Migration |
|------|---------|-----|-----------|
| Template source | Dropbox folder | `dialoa/dltc-template` repo | `git clone`, tags for versions |
| `make update` | `/Users/julien/GitHub/` | `GITHUB=~/GitHub make update` | Set env var or edit Makefile |
| Docker image | DockerHub `philosophiech/dltc-env` | GHCR `ghcr.io/dialoa/dltc-env` | Update .env |
| Docker auth | DockerHub token | GitHub PAT (read:packages) | Generate PAT |
| `dltc-make` path | Reads from mounted workhouse | Reads from `/opt/dltc-template/` in image | Rebuild image |
| pandoc-crossref | In template/filters/ folder | Installed in Docker image (in PATH) | Rebuild image |
| Template updates | Copy files to Dropbox | `git commit && git push`, rebuild image | Learn git workflow |

---

## Repo consolidation

### Absorb into the template repo

| What | Current location | Action |
|------|-----------------|--------|
| `dialectica.csl` + dev scripts | `Philosophie-ch/dltc-csl` (separate repo) | CSL already in `csl/` as a copy. Move `csl-deploy.sh`, `csl-import.sh`, `b2c`, `test.csl.json` into `csl/dev/`. Archive the standalone repo. |
| Open-source fonts | Dropbox `dltc-workhouse/resources/fonts/` | Libertinus and STIX Two into `fonts/`. VenusSBOP (proprietary) stays in the private Docker image only. |

Not absorbed:
- Test issues (Dropbox `dltc-workhouse/tests/`) — used for manual checking, stays separate.
- `conversion-defaults.yaml` — not referenced by any template or script. Drop.

### Archive (superseded)

| Repo | Why |
|------|-----|
| `Philosophie-ch/dialectica-template` | Earlier attempt (submodules under `pandoc/`, Quarto docs site, last pushed Dec 2024). Superseded by this repo. |
| `Philosophie-ch/dltc-csl` | Absorbed into template repo. |

### Leave as-is

| Repo | Why |
|------|-----|
| `dialoa/dialectica-filters` | Upstream source for filters. Template consumes built copies via `make update`. |
| Individual filter repos (`dialoa/statement`, etc.) | Upstream sources. |
| `dialoa/pandokoma` | Upstream for `dialectica.latex` template. JD proposes replacing it with Pandoc's default template (see prerequisites in `decouple-from-dialectica.md`). If that happens, this dependency is removed. |

---

## Naming

The `dltc` prefix is Dialectica-specific. If the template is decoupled from Dialectica (see `decouple-from-dialectica.md`), naming should be generic.

| Current | Proposed | Notes |
|---------|----------|-------|
| `dialoa/dltc-template` | `dialoa/dialoa-template` or `dialoa/pandoc-journal-template` | Depends on whether this stays Dialectica-branded or becomes generic |
| `dltc-env` | `dialoa-env` | |
| `dltc-env-dockerfiles` | `dialoa-env-dockerfiles` | |
| `dltc-make` (script in Docker) | `dialoa-make` or `journal-make` | Hardcoded in Dockerfiles and user muscle memory |
| `findDialoa()` in make.lua | Keep — internal function name, low impact | |
| `dialoa.yaml` marker file | Keep — used by `findDialoa()` | |

Renaming is cosmetic and independent of the functional changes. Decide on final naming before the first tagged release.

---

## Open questions

1. **pandoc-crossref version**: Which version is currently used? Need to match it in the Dockerfile. Check the binary in Dropbox.
2. **dialoa org Actions minutes**: TeXLive full install takes ~30-60 min. With QEMU cross-compile for arm64 it could be longer. May need aggressive Docker layer caching or only build on release tags.
3. **Quarto**: Is it still needed? It's installed in the Dockerfile but not obviously used by the template.
4. **`make.sh` wrapper**: The shell wrapper in `scripts/make.sh` hardcodes `../../template/1.2/`. Once the template is baked into the image, this script is redundant (dltc-make calls make.lua directly). Can be removed or updated.

---

## Verification

1. Build Docker image locally with modified Dockerfile (both arch)
2. Run `dltc-make offhtml` inside container on a test article — verify HTML compiles
3. Run `dltc-make offpdf` — verify PDF compiles
4. Verify `git clone dialoa/dltc-template && git checkout v1.2` matches current Dropbox template/1.2 (minus excluded files)
5. Verify CI workflow builds and pushes to GHCR
6. Verify a fresh consumer can pull the private image with a PAT
7. Test `make update GITHUB=~/GitHub` pulls latest filters correctly
