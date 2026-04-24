# skill-version-generate - Documentation

> Back to [README](../README.md)

## Prerequisites

- Claude Code CLI (installed locally)
- `git` ≥ 2.23 (uses `git log` trailer syntax)
- Initialized git repository (`.git/` present)
- `git config user.name` and `user.email` must be set (hard requirement for releaser validation)
- `gh` CLI (optional) — enables automatic GitHub handle population; degrades gracefully when absent or unauthenticated

## Installation

### Clone into the Claude Code skills directory

```bash
git clone https://github.com/pardnchiu/skill-version-generate ~/.claude/skills/version-generate
```

Claude Code picks up `version-generate` as an available skill on its next session.

### Verify installation

```bash
ls ~/.claude/skills/version-generate/SKILL.md
ls ~/.claude/skills/version-generate/scripts/
```

Both paths must exist.

## Configuration

### Releaser identity

The skill reads from `git config`; there is no separate config file.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

If either field is missing, Step 0 aborts with remediation guidance.

### GitHub handle (optional)

When `gh` CLI is authenticated, `released_by.github` in the frontmatter is populated automatically:

```bash
gh auth login
```

If unauthenticated, the field is omitted — the workflow does not abort.

## Usage

### Basic

Invoke within a project directory:

```
/version-generate
```

Execution flow:

1. Validate `git config user.name` / `user.email`
2. Resolve the latest tag (`git tag -l 'v*.*.*' --sort=-v:refname | head -1`)
3. Parse commits in `$LATEST_TAG..HEAD` (Conventional Commits first)
4. Compute `NEW_VERSION` by tag priority
5. Write `.doc/NEW_VERSION.md` with frontmatter and traceability fields
6. Prepend one line to `.doc/CHANGELOG.md`

### Advanced

#### Projects without tags (first release)

When no tag exists, `LATEST_TAG` defaults to `v0.0.0` and `NEW_VERSION` is fixed to `v0.1.0`.

#### Breaking changes

A commit subject containing `!` or a body containing a `BREAKING CHANGE:` trailer triggers a MAJOR bump and **requires a Migration section**. Generation aborts without leaving a partial file if migration content is missing.

```
feat!: change AuthClient.Login signature
```

or

```
feat: change AuthClient.Login signature

BREAKING CHANGE: Login now requires context.Context and Credentials struct.
```

#### DOC / CHORE-only changes

When commits contain only `STYLE` / `DOC` / `TEST` / `CHORE` types, `type: none` is written to the frontmatter, **the master index is not updated**, and the current version is preserved.

## CLI Reference

### Core workflow (`scripts/*.md` implementation details)

| Step | File | Responsibility |
|---|---|---|
| 0 | `scripts/00-validate-releaser.md` | Validate `git config user.name` / `user.email`; optionally fetch `gh api user` |
| 1 | `scripts/01-detect-version.md` | Resolve `LATEST_TAG`; normalize `REMOTE_URL` |
| 2 | `scripts/02-collect-changes.md` | Conventional Commits parsing + diff fallback |
| 3 | `scripts/03-classify-and-bump.md` | Tag classification and SemVer computation |
| 4 | `scripts/04-output-template.md` | Emit full `.doc/NEW_VERSION.md` template |
| 5 | `scripts/05-update-index.md` | Prepend new release to `.doc/CHANGELOG.md` |
| ∞ | `scripts/06-rules-and-edge-cases.md` | Classification rules and edge cases |

### Conventional Commits type mapping

| Conventional | Internal tag | Version impact |
|---|---|---|
| `feat` | FEAT | MINOR |
| `fix` | FIX | PATCH |
| `perf` | PERF | PATCH |
| `refactor` | REFACTOR | PATCH |
| `security` | SECURITY | PATCH |
| `revert` | REMOVE | PATCH |
| `docs` | DOC | — |
| `test` | TEST | — |
| `style` | STYLE | — |
| `chore` / `build` / `ci` | CHORE | — |
| Any type + `!` | Adds BREAKING | MAJOR |

### SemVer bump rules

| Condition | `NEW_VERSION` |
|---|---|
| `LATEST_TAG = v0.0.0` (no tags) | `v0.1.0` |
| Any `BREAKING` detected | `v(X+1).0.0` |
| Any `FEAT` detected | `vX.(Y+1).0` |
| Any `PATCH_TAGS` detected | `vX.Y.(Z+1)` |
| Only `NO_BUMP` tags | `LATEST_TAG` (`type: none`) |

### Frontmatter fields

| Field | Source | Required |
|---|---|---|
| `version` | `NEW_VERSION` | ✅ |
| `previous` | `LATEST_TAG` | ✅ |
| `date` | Today (`YYYY-MM-DD`) | ✅ |
| `type` | `major` / `minor` / `patch` / `none` | ✅ |
| `breaking` | `true` / `false` | ✅ |
| `released_by.name` | `git config user.name` | ✅ |
| `released_by.email` | `git config user.email` | ✅ |
| `released_by.github` | `gh api user --jq .login` | Omit if missing |
| `contributors` | `git log $LATEST_TAG..HEAD --format='%an <%ae>' \| sort -u` | ✅ |
| `co_authors` | `Co-authored-by` trailer | Omit if empty |
| `compare` | `$REMOTE_URL/compare/$LATEST_TAG...$NEW_VERSION` | Omit for non-GitHub remotes |

### Entry format

Each change entry suffix is `(#PR, @author) [short_hash]`; missing fields drop their parentheses:

| Completeness | Example |
|---|---|
| Full | `Add OAuth2 login flow (#142, @alice) [a3f9c2d]` |
| No PR | `Add OAuth2 login flow (@alice) [a3f9c2d]` |
| No handle | `Add OAuth2 login flow (#142) [a3f9c2d]` |
| Minimal | `Add OAuth2 login flow [a3f9c2d]` |

### Output locations

| Path | Content |
|---|---|
| `.doc/vX.Y.Z.md` | Complete changelog for a single release |
| `.doc/CHANGELOG.md` | Master index (newest on top) |

### Abort conditions

| Condition | Behavior |
|---|---|
| `git config user.name` or `user.email` missing | Step 0 aborts |
| `BREAKING` change without Migration content | Step 4 aborts |
| No commits in `$LATEST_TAG..HEAD` | Emit "no changes" message and exit |
