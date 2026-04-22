# /test — Run or Generate Tests

## Usage

```
/test unit                 # Run all unit tests
/test integration          # Run integration tests (requires Docker)
/test [feature]            # Run tests for specific feature
/test generate [feature]   # Generate missing tests for a feature
/test coverage             # Show coverage report
/test lint                 # Run ruff + mypy
```

## Run commands

```bash
# Unit tests (fast, no external deps)
pytest tests/unit/ -v -x --tb=short

# Integration tests
pytest tests/integration/ -v --tb=short

# All tests with coverage
pytest tests/ --cov=. --cov-report=term-missing --cov-fail-under=80

# Specific feature
pytest tests/ -k "registration" -v

# Type checking
mypy bot/ api/ worker/ shared/ --strict --ignore-missing-imports

# Linting
ruff check . && ruff format --check .
```

## Generate missing tests

When called with `generate [feature]`:

1. Read `docs/test-scenarios.md` for the BDD scenarios
2. Read existing code for the feature
3. Generate test file with:
   - Happy path tests
   - Error case tests
   - Security tests (HMAC, auth)
   - Edge case tests

### Template for test generation

Follow `.claude/skills/testing-patterns/SKILL.md` patterns.

Coverage requirements:
- Unit tests: all service functions, 80%+ coverage
- Integration tests: webhook handler, subscription lifecycle
- Type: always use `mypy --strict`

## CI check (pre-push)

```bash
# Run this before every push
ruff check . && mypy bot/ api/ worker/ shared/ --strict --ignore-missing-imports && pytest tests/unit/ -x -q
```
