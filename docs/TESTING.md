# Testing Requirements

Every skill must have a corresponding test directory under the top-level `tests/` folder.
A skill that does not meet the testing requirements will not be approved for merge.

## Rules

1. A `tests/<skill-name>/` directory must exist at the repository root, matching the skill's directory name.
2. All tests must pass before a PR can be approved.
3. Test coverage for the skill must be at least 75% (line coverage).
4. Coverage is measured only over the skill's own code (`scripts/`, helper modules, etc.) - not third-party libraries.

## Test Directory Layout

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

## What To Test

Skill tests should cover the following areas:

### 1. Unit Tests (required)

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

### 2. Script Tests (required when scripts are present)

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

### 3. Integration Tests (recommended)

If the skill orchestrates multiple scripts or calls external services, add integration tests that run the full flow end-to-end.
Mark these clearly so they can be run or skipped independently:

```python
import pytest

@pytest.mark.integration
def test_full_pdf_pipeline():
    # run extract -> validate -> output
    ...
```

## Running Tests

From the repository root, run tests for a specific skill:

```bash
# Run all tests for a skill
pytest tests/<skill-name>/

# Run with coverage report (measure coverage over the skill's scripts)
pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-report=term-missing

# Fail if coverage is below 75%
pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-fail-under=75
```

If the skill contains additional source directories beyond `scripts/`, include them in the `--cov` flag:

```bash
pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov=skills/<skill-name>/helpers --cov-fail-under=75
```

## CI Enforcement

The repository CI pipeline (via the GitHub Actions workflows described in `README.md#ci--automation-workflows`) will:
1. Detect which skills were changed in the PR.
2. Run `pytest tests/<skill-name>/ --cov=skills/<skill-name>/scripts --cov-fail-under=75` for each changed skill.
3. Block the merge if any test fails or coverage drops below 75%.
4. On successful merge, automatically create the release tag and update `registry.json`.

Contributors should run the same command locally before pushing.
