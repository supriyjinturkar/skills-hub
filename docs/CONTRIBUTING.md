# Contribution Guidelines

Follow these steps when contributing a new skill or updating an existing one.

## Before You Start

1. Check the centralized skills repository to make sure a similar skill does not already exist.
2. If a related skill exists, consider extending it with a new version instead of creating a duplicate.
3. Read the [Skill Versioning And Branch Model](#skill-versioning-and-branch-model) to understand the branch model.

## Skill Versioning And Branch Model

This repository follows a branch-based versioning model for skills. It keeps each skill isolated while still maintaining a single shared repo.

### Branch Roles

- `develop`
  - Main working branch with the latest approved state of all skills.
- `empty`
  - An empty baseline branch used only to start a brand-new skill from scratch.
- `<skill-name>`
  - The long-lived branch for one skill (example: `pdf-processing`).
- `<skill-name>-<version>`
  - Temporary release branch for a new version (example: `pdf-processing-1.1.0`).

### Naming Rules

- Skill branches must use lowercase kebab-case.
- Version branches must include the skill name (avoid version-only branches like `1.1.0`).

### Source Of Truth

- The skill version lives in `metadata.version` in `SKILL.md`.
- Branch names, `metadata.version`, and release tags must stay aligned.


### Required Rules

- Never create a new skill directly from `develop` if a clean start is required.
- Always create new skills from `empty`.
- Always create version branches from the skill branch.
- Always bump `metadata.version` for meaningful changes.

## Creating A New Skill

1. Create a branch from `empty` named after your skill (e.g. `pdf-processing`).
2. Add your skill directory under `skills/<skill-name>/`.
3. Include at minimum:
   - `SKILL.md` with valid frontmatter (`name`, `description`, `metadata.author`, `metadata.version`).
   - A corresponding `tests/<skill-name>/` directory with tests that pass and meet the 75% coverage threshold.
4. If the skill has runnable code, place it under `scripts/`.
5. If the skill has supporting documents, place them under `references/`.
6. Run `pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-fail-under=75` locally and confirm everything passes.
7. Open a PR from your skill branch into `develop`.
8. Address code-review feedback.
9. After approval and merge, the CI workflow automatically creates the release tag (e.g. `pdf-processing-1.0.0`) and updates `registry.json`.

## Updating An Existing Skill

1. Create a version branch from the existing skill branch (e.g. `pdf-processing-1.1.0`).
2. Make your changes.
3. Bump `metadata.version` in `SKILL.md`.
4. Add or update tests under `tests/<skill-name>/` to cover any new or changed functionality - coverage must stay at or above 75%.
5. Run `pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-fail-under=75` locally.
6. Open a PR from the version branch into the skill branch.
7. After approval and merge into the skill branch, the CI workflow automatically creates the release tag (e.g. `pdf-processing-1.1.0`) and updates `registry.json`.
8. A separate CI workflow then syncs the released changes into `develop`.

## PR Review Checklist

Reviewers will verify:
- [ ] `SKILL.md` frontmatter is valid and version is bumped correctly
- [ ] Skill directory name matches `name` in frontmatter
- [ ] `tests/<skill-name>/` directory exists with meaningful tests
- [ ] All tests pass
- [ ] Coverage is >= 75%
- [ ] Scripts and references are placed in correct subdirectories
- [ ] No large unrelated files are included
- [ ] The skill body clearly explains purpose, inputs, steps, and expected output
- [ ] The correct branch model was followed (see [Skill Versioning And Branch Model](#skill-versioning-and-branch-model))

## Author Checklist

Before adding or updating a skill, verify:
- directory name is correct and in kebab-case
- `SKILL.md` exists with valid YAML frontmatter
- `name` in frontmatter matches directory name
- `metadata.version` is set and bumped correctly
- `description` clearly says when to use the skill
- `tests/<skill-name>/` directory exists with passing tests and >= 75% coverage
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