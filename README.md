<p align="center">
  <img src="https://raw.githubusercontent.com/tim-engine/tim/main/.github/tim_logo.png" alt="Tim Engine" width="120px" height="120px"><br>
  <strong>pkgs</strong>, the official package registry for Tim Engine<br>
  Every package installable with <code>tim install</code> is listed here.
</p>

<p align="center">
  <a href="https://github.com/tim-engine/tim">tim-engine/tim</a> &bullet;
  <a href="https://tim.openpeeps.dev/">Documentation</a> &bullet;
  <a href="https://github.com/tim-engine/pkgs/blob/main/packages.json">packages.json</a>
</p>

## What lives here

This repo contains one file that matters: [`packages.json`](packages.json).
It is the registry that Tim Engine reads when you run `tim install <name>`.

Tim is wired to this repo by default:

| Setting | Value |
|---|---|
| Source name | `tim-engine` |
| Registry URL | `https://raw.githubusercontent.com/tim-engine/pkgs/main/packages.json` |
| Local cache | `~/.tim/registries/tim-engine.json` |

Each entry in `packages.json` describes one installable package:

```json
{
  "name": "bootstrap",
  "url": "https://github.com/tim-engine/bootstrap.timl",
  "method": "git",
  "tags": ["tim", "timl", "bootstrap", "frontend", "components", "css", "ui"],
  "description": "Bootstrap v5.x components for Tim Engine",
  "license": "MIT",
  "web": "https://github.com/tim-engine/bootstrap.timl"
}
```

Field notes (all fields are required):

| Field | Meaning |
|---|---|
| `name` | Package name. Lowercase, valid identifier, matches the `name` in the package `tim.config.yml`. |
| `url` | Public git URL of the package repo. This is what gets cloned. |
| `method` | Always `"git"` for now. |
| `tags` | Search keywords. Include at least `tim` or `timl`. |
| `description` | One line summary shown to users browsing the registry. |
| `license` | SPDX identifier, e.g. `MIT`. |
| `web` | Homepage of the package (usually the repo URL). Entries without `web` are skipped by the client. |

## Package index

| Package | Description | Repo |
|---|---|---|
| `bootstrap` | Bootstrap v5.x components for Tim Engine | [tim-engine/bootstrap.timl](https://github.com/tim-engine/bootstrap.timl) |

## Using packages: `tim install`

Prerequisites: the Tim Engine CLI. See [tim-engine/tim](https://github.com/tim-engine/tim) for install instructions.

### 1. Install a package by name

```bash
tim install bootstrap
```

This resolves `bootstrap` in the `tim-engine` source, clones it into the local
cache (`~/.tim/packages/_cache/bootstrap`), picks the newest tagged version
(or `HEAD` when the repo has no tags yet), and copies a clean checkout to
`~/.tim/packages/bootstrap/<version>`.

### 2. Install a specific version, branch, or tag

```bash
tim install bootstrap@0.1.0   # exact version (needs a matching git tag, e.g. 0.1.0 or v0.1.0)
tim install bootstrap@main     # a branch or tag name
```

### 3. Install with a version constraint

```bash
tim install "bootstrap >= 0.1.0"
```

Supported operators: `=`, `==`, `>=`, `>`, `<=`, `<`, `^` (caret), `~` and `~>` (tilde).

### 4. Install straight from a git URL (no registry needed)

```bash
tim install https://github.com/tim-engine/bootstrap.timl
tim install https://github.com/tim-engine/bootstrap.timl#main
```

Handy for trying a package before it is published here, or for private repos.

### 5. Use the package in your templates

Installed packages are imported with the `pkg/` prefix:

```timl
@import "pkg/bootstrap/grid"
@import "pkg/bootstrap/button"
```

Or pull in everything through the aggregator (when the package ships one):

```timl
@import "pkg/bootstrap/bootstrap"
```

### 6. Declare dependencies in `tim.config.yml`

List what your project or package needs so others get the full tree on install:

```yaml
name: my-site
type: project
version: 0.1.0
license: MIT
description: "My Tim Engine site"

requires:
  - bootstrap >= 0.1.0
```

### 7. Manage installed packages

```bash
tim develop                  # link the current directory as an editable package
tim develop ./my-package     # link a specific directory
tim remove bootstrap         # remove all installed versions
tim remove bootstrap@0.1.0   # remove one version
```

### 8. Where things live on disk

| Path | Contents |
|---|---|
| `~/.tim/packages/_cache/<name>` | Bare git clone used for version discovery |
| `~/.tim/packages/<name>/<version>` | Clean installed copy your templates resolve against |
| `~/.tim/develop/<name>` | Symlink created by `tim develop` (editable, wins over registry copies) |
| `~/.tim/registries/tim-engine.json` | Downloaded copy of this repo `packages.json` |
| `~/.tim/sources.json` | Configured sources (defaults to the `tim-engine` source above) |

### Troubleshooting

| Symptom | Fix |
|---|---|
| `Could not download registry for tim-engine: curl failed ... 404` | Your cached registry predates this repo having a `packages.json`, or the file is unreachable. Check the Registry URL in the table above loads in a browser, then delete `~/.tim/registries/tim-engine.json` and retry so Tim re-downloads it. |
| `No registry data seeded, run source.fetch` | Same cause: no registry was ever downloaded successfully. Fix connectivity to the Registry URL, clear the cached file above, and retry. |
| `Package not found in registry: <name>` | The name is not in `packages.json` (check spelling and the package index), or your local registry copy is stale (delete the cached file and retry). As a workaround, install by git URL (step 4). |
| Wrong version installed | The resolver picks the newest git tag satisfying your constraint. Push the missing tag on the package repo, or pin with `tim install <name>@<ref>`. |

## Publishing a package

The easy path is `tim publish`, which validates your package, forks this
repo, adds the entry, and opens the pull request for you:

```bash
cd my-widgets
tim publish --tags "tim, widgets, ui" --dry-run   # preview the entry, changes nothing
tim publish --tags "tim, widgets, ui"             # validate, fork, PR
```

On first run it asks for a GitHub token (least-privilege `public_repo`
scope is enough) and a password, then stores the token encrypted in
`~/.tim/secret` (Argon2id + XChaCha20-Poly1305). Every later run just asks
for the password. Useful flags: `--path <dir>` to publish a different
directory, `--web <url>` to override the homepage, `--message <text>` to
append PR body text, `--direct` to push the branch to this repo instead of
a fork (maintainers), `--reset-token` to replace the stored token, and
`--yes` to skip the confirmation prompt.

Prefer the manual route? It is equivalent: add your package to
[`packages.json`](packages.json) via a pull request to this repo. The full
requirements below apply either way.

### Step 1. Put your package in a public git repo

Any public host that serves git over HTTPS works (GitHub recommended).

### Step 2. Add a `tim.config.yml` manifest at the repo root

The canonical name is `tim.config.yml` (`tim.config.yaml` is also accepted).
Minimal example for a package:

```yaml
name: my-widgets
type: package
version: 0.1.0
author: Your Name
license: MIT
description: "Handy widgets for Tim Engine"

requires:
  - bootstrap >= 0.1.0
```

Rules:

- `name` must match the name you request in the registry (lowercase identifier).
- `type` is `package` for reusable libraries, `project` for apps and sites.
- `version` is semver (`0.1.0`). It is used for `HEAD` installs until you push tags.
- `requires` is a list of `name [constraint]` or git URL entries. Omit it when you have no dependencies.
- Keep generated output, `node_modules`-style vendored code, and editor files out of the repo. Installs copy the tree minus `.git`, tests, examples, and docs.

### Step 3. Tag your releases

```bash
git tag 0.1.0 && git push origin 0.1.0
```

Both `0.1.0` and `v0.1.0` tag styles are recognized. The resolver installs the
highest tag satisfying the requested constraint. Untagged repos still install
(the manifest `version` is used and recorded as `HEAD`), but tags are strongly
recommended so users can pin versions.

### Step 4. Test a local install before submitting

```bash
# from inside your package directory
tim develop

# from any other project, install by URL to prove the repo is consumable
tim install https://github.com/<you>/<repo>.git
```

If both work, the package is ready for the registry.

### Step 5. Submit the entry (`tim publish`, or a manual PR)

With `tim publish` (recommended) this step is automatic: it re-validates the
manifest, checks the name is still free, forks this repo, inserts your entry
in sorted order on a branch named `add-<name>-<time>`, pushes, and opens the
PR with the checklist pre-filled. Review the checklist below anyway so the
PR sails through.

Manual alternative:

1. Fork this repo and add one JSON object to the array in `packages.json`, keeping entries sorted by `name`.
2. Double check every required field from the table above (`web` included, otherwise the client skips your entry).
3. In the PR description include: package repo URL, one line on what it does, and confirmation that `tim install <url>` works.
4. A maintainer reviews, merges, and your package becomes installable by name (`tim install <name>`) once the raw file updates (usually within minutes; users with a stale cache may need to delete `~/.tim/registries/tim-engine.json` once).

Example diff:

```diff
   [
     {
       "name": "bootstrap",
       ...
+    },
+    {
+      "name": "my-widgets",
+      "url": "https://github.com/<you>/my-widgets.timl",
+      "method": "git",
+      "tags": ["tim", "timl", "widgets"],
+      "description": "Handy widgets for Tim Engine",
+      "license": "MIT",
+      "web": "https://github.com/<you>/my-widgets.timl"
     }
   ]
```

Validation checklist for reviewers:

- [ ] Entry is valid JSON and keeps alphabetical order by `name`.
- [ ] `name` matches the `name` in the package manifest.
- [ ] `url` clones anonymously (`git ls-remote <url> HEAD` succeeds).
- [ ] Repo root contains `tim.config.yml` (or `.yaml`) with `name`, `version`, `description`, `license`.
- [ ] At least one semver tag exists (or the package is intentionally untagged and installs from the default branch).
- [ ] No name squatting or trademark conflicts.

### Updating or removing your package

- New versions need no registry change. Just push a new semver tag; Tim discovers tags on install.
- To change the description, tags, or repo URL, open a PR editing your entry.
- To unpublish, open a PR removing your entry and explain why. Installed copies keep working; only new installs stop resolving.

## Contributing

Issues and PRs are welcome. For registry entries, follow the publishing steps
above. For anything else (docs, tooling), open an issue first so maintainers
can point you in the right direction.

## License

MIT. See [LICENSE](LICENSE).
