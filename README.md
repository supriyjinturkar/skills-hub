# Skills Guide

This guide defines how to create, place, version, test, and maintain skills.

## Overview

Skills are hosted in a **centralized shared repository**.
Team members across the company can browse the repository, find a skill that fits their needs, and consume it directly as a `.zip` dependency in their project's `pyproject.toml`.

Because skills are consumed independently, every skill must be:
- **self-contained** — all code and references live inside the skill directory
- **versioned** — each skill carries its own semantic version (see [Versioning Standards Guide](versioning.md))
- **tested** — every skill has a corresponding test suite under the top-level `tests/` directory (see [Testing Requirements](#testing-requirements) below)

## Skill Location

Put each new skill in:

```text
skills/<skill-name>/
```

Minimum required file:

```text
skills/<skill-name>/SKILL.md
```

Corresponding test directory (mandatory — see [Testing Requirements](#testing-requirements)):

```text
tests/<skill-name>/
```

Optional directories:

```text
skills/<skill-name>/scripts/
skills/<skill-name>/references/
```

Example:

```text
skills/pdf-processing/
|-- SKILL.md
|-- scripts/
`-- references/

tests/pdf-processing/
`-- test.py
```

## Naming Rules

- Use a stable kebab-case skill name.
- The directory name and frontmatter `name` should match.
- Choose a name that describes the capability, not the implementation detail.

Good examples:
- `pdf-processing`
- `kubernetes-triage`
- `postgres-diagnostics`

Avoid:
- `pdfskill`
- `my-new-skill`
- `helper-v2`

## Required Frontmatter

Every `SKILL.md` must start with YAML frontmatter.

Example:

```yaml
---
name: pdf-processing
description: Extract PDF text, fill forms, merge files. Use when handling PDFs.
metadata:
  author: example-org
  version: "1.0.0"
---
```

## Frontmatter Rules

### `name`
- Required.
- Must be the canonical skill identifier.
- Must match the skill directory name.
- Use lowercase kebab-case only.

### `description`
- Required.
- Keep it short and task-oriented.
- It should explain when the skill should be used.

### `metadata.author`
- Required.
- Use the owning team, org, or package author name.

### `metadata.version`
- Required.
- Use semantic versioning.
- This version is important because packaged skill releases depend on stable versioned skill content.
- Bump the version whenever the skill changes in a meaningful way, including:
  - prompt or behavior changes
  - script changes
  - reference changes that affect output or guidance
  - compatibility-impacting changes

Recommended versioning:
- `MAJOR`: breaking skill behavior or incompatible contract change
- `MINOR`: backward-compatible capability improvement
- `PATCH`: fix or clarification with no intended contract break

## Skill Body Structure

After frontmatter, keep the body practical and structured.

Recommended sections:

```markdown
# Purpose

# When To Use

# Inputs

# Steps

# Scripts

# References

# Output Expectations
```

The exact headings can vary, but the skill should make these things clear:
- what the skill does
- when it should be selected
- what inputs it expects
- what steps the agent should follow
- what scripts/references are available
- what output quality is expected

## Scripts And References

### `scripts/`
Use `scripts/` for runnable helpers that belong to the skill.

Examples:
- parsing helpers
- API wrappers
- validation scripts
- repeatable operational commands

### `references/`
Use `references/` for supporting material.

Examples:
- protocol notes
- API docs snapshots
- field mapping notes
- troubleshooting references

Do not put large unrelated material into a skill directory.

## Placement Rules

- Add new skills only under the main `skills/` directory.
- Do not create one repo per skill by default.
- Multiple skills live in the same centralized skills repository and are discovered from that shared location.
- Keep each skill self-contained inside its own folder.
- Consumers install individual skills via the skill registry in their `pyproject.toml` (see [Consuming A Skill](#consuming-a-skill) below).

## Consuming A Skill

The centralized skills repository maintains a **`registry.json`** on the `develop` branch.
This registry lists every available skill, its released versions, and the download URL for each version.

### Registry Structure

`registry.json` lives at the repository root and follows this format:

```json
{
  "skills": {
    "news-summary": {
      "versions": {
        "1.0.0": {
          "url": "https://github.com/<org>/skills-repo/archive/refs/tags/news-summary-1.0.0.zip"
        }
      },
      "latest": "1.0.0"
    },
    "weather": {
      "versions": {
        "1.0.0": {
          "url": "https://github.com/<org>/skills-repo/archive/refs/tags/weather-1.0.0.zip"
        }
      },
      "latest": "1.0.0"
    }
  }
}
```

Each skill entry contains:
- `versions` — a map of version strings to their download URLs (`.zip` archives built from release tags)
- `latest` — the version string that `"latest"` resolves to

The registry is updated by the maintainer whenever a new skill version is released and merged into `develop`.

### Installing Skills In Your Project

To consume skills from the registry, add the following to your project's `pyproject.toml`:

```toml
[tool.skills]
registry = "https://raw.githubusercontent.com/<org>/skills-repo/refs/heads/develop/registry.json"

[tool.skills.install]
pdf-processing = "1.0.1"    # pin to an exact version
email-parser = "latest"      # always resolve to the latest released version
```

- **`[tool.skills].registry`** — points to the raw URL of the `registry.json` on the `develop` branch.
- **`[tool.skills.install]`** — lists each skill you want to use and the desired version.
  - Use an exact version string (e.g. `"1.0.1"`) for reproducible, pinned builds.
  - Use `"latest"` to always pull the most recently released version.

Replace `<org>` and `skills-repo` with your company's organization and repository name.

> **Tip:** Pinning to an exact version is recommended for production projects to keep builds reproducible. Use `"latest"` only for development or when you always want the newest release.

### Skill Archive Structure

When a `.zip` archive is downloaded from a GitHub release tag, GitHub wraps the repository contents in a root folder named after the tag. The archive layout looks like this:

```text
skills-hub-slack-1.0.1.zip
└── skills-hub-slack-1.0.1/       # GitHub-generated root folder
    ├── skills/
    │   └── slack/
    │       ├── SKILL.md
    │       └── scripts/
    └── tests/
        └── slack/
            └── test.py
```

The actual skill content lives at:

```text
<archive-root>/skills/<skill-name>/
```

For example, for the `slack` skill at version `1.0.1`:

```text
skills-hub-slack-1.0.1/skills/slack/
```

### Skill Resolver Script

The skill dependency resolver script (which processes the `[tool.skills.install]` entries) must handle this nested structure:

1. Download the `.zip` archive from the URL in `registry.json`.
2. Extract the archive — the contents will be inside a root folder (e.g. `skills-hub-slack-1.0.1/`).
3. Navigate into `<root-folder>/skills/<skill-name>/` to locate the actual skill.
4. Copy or link the skill into the consuming project's skill directory.
5. Optionally exclude the `tests/` folder from the extraction.

The `tests/` directory is useful for contributors working on the skill, but consumers typically do not need it. To support this, the resolver should accept an optional configuration:

```toml
[tool.skills]
registry = "https://raw.githubusercontent.com/<org>/skills-repo/refs/heads/develop/registry.json"
exclude_tests = true          # optional — skip the tests/ folder when installing (default: true)

[tool.skills.install]
slack = "1.0.1"
```

When `exclude_tests` is `true` (the default), the resolver extracts only the `skills/<skill-name>/` directory from the archive and ignores `tests/<skill-name>/`. When set to `false`, both `skills/` and `tests/` are extracted — useful for local development or running the skill's tests in the consuming project.

## Packaging And Release Notes

Skills live in one shared centralized repository.
For an approved skill release, the skill `metadata.version` and the release tag must stay aligned.

Release flow:
- when a skill is approved and merged, a CI workflow automatically creates the corresponding release tag and updates `registry.json`
- the shared skills package release is separate and is cut from `develop` when the repo maintainer decides to publish a new shared package version

If a skill changes materially, update `metadata.version` even if the shared package release happens later.

## CI / Automation Workflows

The repository will include GitHub Actions workflows to automate release management and quality enforcement.
Two primary workflows are planned:

### Workflow 1: New Skill Merged Into `develop`

Triggered when a **new skill branch** is merged into `develop` for the first time.

This workflow will:
1. Detect that a new skill directory was added under `skills/`.
2. Run the skill's tests (`pytest tests/<skill-name>/`) and enforce the 75 % coverage threshold.
3. Read `metadata.version` from the skill's `SKILL.md`.
4. Create the release tag (e.g. `pdf-processing-1.0.0`).
5. Update `registry.json` on the `develop` branch with the new skill entry, version, and `.zip` download URL.

### Workflow 2: Version Sub-Branch Merged Into Skill Branch

Triggered when a **version sub-branch** (e.g. `pdf-processing-1.1.0`) is merged into its parent **skill branch** (e.g. `pdf-processing`).

This workflow will:
1. Detect that the skill directory was modified on the skill branch.
2. Run the skill's tests and enforce the 75 % coverage threshold.
3. Read the updated `metadata.version` from `SKILL.md`.
4. Create the release tag for the new version (e.g. `pdf-processing-1.1.0`).
5. Update `registry.json` — add the new version entry and update the `latest` pointer.

### Workflow 3: Sync Release To `develop`

Triggered whenever a **release is created** (by Workflow 1 or Workflow 2).

This workflow will:
1. Identify the skill and version from the release tag.
2. Take the released skill changes from the skill branch.
3. Directly merge those changes into the `develop` branch so that `develop` always reflects the latest released state of every skill.

### Shared Workflow Responsibilities

Workflows 1 and 2 share these common steps:
- **Test gate** — the merge is blocked if any test fails or coverage is below 75 %.
- **Tag creation** — the release tag is created automatically from `metadata.version`.
- **Registry update** — `registry.json` is kept in sync so consumers always see the latest available versions.

Workflow 3 ensures `develop` stays up to date after every release.

> **Note:** These workflows are planned and will be implemented as GitHub Actions in the repository. Until then, the maintainer performs these steps manually.

## Testing Requirements

Every skill **must** have a corresponding test directory under the top-level `tests/` folder.
A skill that does not meet the testing requirements will not be approved for merge.

### Rules

1. A `tests/<skill-name>/` directory must exist at the repository root, matching the skill's directory name.
2. All tests must **pass** before a PR can be approved.
3. Test coverage for the skill must be at least **75 %** (line coverage).
4. Coverage is measured only over the skill's own code (`scripts/`, helper modules, etc.) — not third-party libraries.

### Test Directory Layout

```text
skills/<skill-name>/
|-- SKILL.md
|-- scripts/
|   |-- extract_text.py
|   `-- validate_output.py
`-- references/

tests/<skill-name>/
|-- __init__.py                # may be empty
|-- conftest.py                # shared fixtures (optional)
|-- test_extract_text.py       # unit tests for extract_text.py
|-- test_validate_output.py
`-- test_integration.py        # end-to-end / integration tests (optional)
```

### What To Test

Skill tests should cover the following areas:

#### 1. Unit Tests (required)

Test individual functions, classes, and helpers in isolation.

- Every file under `scripts/` should have a corresponding `test_<filename>.py`.
- Mock external dependencies (APIs, file systems, databases) so tests run offline and fast.
- Cover both the happy path and meaningful error / edge cases.

Example (`tests/<skill-name>/test_extract_text.py`):

```python
import pytest
from scripts.extract_text import extract_pages

def test_extract_pages_returns_list():
    result = extract_pages("tests/fixtures/sample.pdf")
    assert isinstance(result, list)
    assert len(result) > 0

def test_extract_pages_empty_pdf():
    result = extract_pages("tests/fixtures/empty.pdf")
    assert result == []

def test_extract_pages_invalid_path():
    with pytest.raises(FileNotFoundError):
        extract_pages("does_not_exist.pdf")
```

#### 2. Script Tests (required when scripts are present)

If a skill contains runnable scripts, test that each script:
- exits with code `0` on valid input
- exits with a non-zero code and a clear error message on invalid input
- produces the expected output or side effects

Example (`tests/<skill-name>/test_validate_output.py`):

```python
import subprocess, sys

def test_validate_output_script_success():
    result = subprocess.run(
        [sys.executable, "scripts/validate_output.py", "--input", "tests/fixtures/valid.json"],
        capture_output=True, text=True,
    )
    assert result.returncode == 0

def test_validate_output_script_failure():
    result = subprocess.run(
        [sys.executable, "scripts/validate_output.py", "--input", "tests/fixtures/bad.json"],
        capture_output=True, text=True,
    )
    assert result.returncode != 0
    assert "error" in result.stderr.lower()
```

#### 3. Integration Tests (recommended)

If the skill orchestrates multiple scripts or calls external services, add integration tests that run the full flow end-to-end.
Mark these clearly so they can be run or skipped independently:

```python
import pytest

@pytest.mark.integration
def test_full_pdf_pipeline():
    # run extract -> validate -> output
    ...
```

### Running Tests

From the repository root, run tests for a specific skill:

```bash
# Run all tests for a skill
pytest tests/<skill-name>/

# Run with coverage report (measure coverage over the skill's scripts)
pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-report=term-missing

# Fail if coverage is below 75 %
pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-fail-under=75
```

If the skill contains additional source directories beyond `scripts/`, include them in the `--cov` flag:

```bash
pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov=skills/<skill-name>/helpers --cov-fail-under=75
```

### CI Enforcement

The repository CI pipeline (via the GitHub Actions workflows described in [CI / Automation Workflows](#ci--automation-workflows)) will:
1. Detect which skills were changed in the PR.
2. Run `pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-fail-under=75` for each changed skill.
3. Block the merge if any test fails or coverage drops below 75 %.
4. On successful merge, automatically create the release tag and update `registry.json`.

Contributors should run the same command locally before pushing.

## Contribution Guidelines

Follow these steps when contributing a new skill or updating an existing one.

### Before You Start

1. Check the centralized skills repository to make sure a similar skill does not already exist.
2. If a related skill exists, consider extending it with a new version instead of creating a duplicate.
3. Read the [Versioning Standards Guide](versioning.md) to understand the branch model.

### Creating A New Skill

1. Create a branch from `empty` named after your skill (e.g. `pdf-processing`).
2. Add your skill directory under `skills/<skill-name>/`.
3. Include at minimum:
   - `SKILL.md` with valid frontmatter (`name`, `description`, `metadata.author`, `metadata.version`).
   - A corresponding `tests/<skill-name>/` directory with tests that pass and meet the 75 % coverage threshold.
4. If the skill has runnable code, place it under `scripts/`.
5. If the skill has supporting documents, place them under `references/`.
6. Run `pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-fail-under=75` locally and confirm everything passes.
7. Open a PR from your skill branch into `develop`.
8. Address code-review feedback.
9. After approval and merge, the CI workflow automatically creates the release tag (e.g. `pdf-processing-1.0.0`) and updates `registry.json`.

### Updating An Existing Skill

1. Create a version branch from the existing skill branch (e.g. `pdf-processing-1.1.0`).
2. Make your changes.
3. Bump `metadata.version` in `SKILL.md`.
4. Add or update tests under `tests/<skill-name>/` to cover any new or changed functionality — coverage must stay at or above 75 %.
5. Run `pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-fail-under=75` locally.
6. Open a PR from the version branch into the skill branch.
7. After approval and merge into the skill branch, the CI workflow automatically creates the release tag (e.g. `pdf-processing-1.1.0`) and updates `registry.json`.
8. A separate CI workflow then syncs the released changes into `develop`.

### PR Review Checklist

Reviewers will verify:
- [ ] `SKILL.md` frontmatter is valid and version is bumped correctly
- [ ] Skill directory name matches `name` in frontmatter
- [ ] `tests/<skill-name>/` directory exists with meaningful tests
- [ ] All tests pass
- [ ] Coverage is ≥ 75 %
- [ ] Scripts and references are placed in correct subdirectories
- [ ] No large unrelated files are included
- [ ] The skill body clearly explains purpose, inputs, steps, and expected output
- [ ] The correct branch model was followed (see [Versioning Standards Guide](versioning.md))

## Author Checklist

Before adding or updating a skill, verify:
- directory name is correct and in kebab-case
- `SKILL.md` exists with valid YAML frontmatter
- `name` in frontmatter matches directory name
- `metadata.version` is set and bumped correctly
- `description` clearly says when to use the skill
- `tests/<skill-name>/` directory exists with passing tests and ≥ 75 % coverage
- scripts and references are placed in the correct subdirectories
- the skill body explains expected inputs, steps, and outputs
- the correct branch model was followed

## Example Layout

```text
skills/pdf-processing/
|-- SKILL.md
|-- scripts/
|   |-- extract_text.py
|   `-- validate_output.py
`-- references/
    `-- form-field-notes.md

tests/pdf-processing/
|-- __init__.py
|-- conftest.py
|-- test_extract_text.py
`-- test_validate_output.py
```

## Related Documentation

- [Versioning Standards Guide](versioning.md)
