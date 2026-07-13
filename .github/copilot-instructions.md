# Copilot instructions for `calc-api-hands-on`

These instructions describe how to work efficiently in this repository. **Trust
them first** and only fall back to broad codebase searches when the information
here is missing, incomplete, or proven wrong by what you observe.

## What this repository is

A small sample **HTTP API for multiplication and division**, implemented as
**Azure Functions** in **Python**. It is a hands-on/learning project, so it is
intentionally tiny. The whole codebase is a handful of Python files.

- Endpoints: `/multiply` and `/divide`, each accepting **GET and POST**.
- Responses: JSON by default; a minimal HTML fragment is returned when the
  request's `Accept` header prefers `text/html` over `application/json`.
- Division by zero returns HTTP `200` with `result` set to the string
  `"Infinity"` (this is intentional per `docs/requirements.md`, not a bug).
- Requirements docs (in Japanese) live in `docs/requirements.md` (functional)
  and `docs/nonrequirements.md` (non-functional). Target runtime is Python 3.11
  on the Azure Functions Consumption Plan (Japan East).

## Repository layout

```
.
├── .github/workflows/ci.yml   # CI: installs deps, runs `pytest -q` (Python 3.11)
├── docs/
│   ├── requirements.md        # Functional requirements (Japanese)
│   └── nonrequirements.md     # Non-functional requirements (Japanese)
├── src/
│   ├── __init__.py            # makes `src` an importable package
│   ├── calc.py                # core math: to_decimal(), multiply(), divide()
│   ├── common.py              # json_response()/html_response() helpers
│   ├── multiply/
│   │   ├── __init__.py        # Azure Function entrypoint: main(req)
│   │   └── function.json      # httpTrigger binding (anonymous, get+post)
│   └── divide/
│       ├── __init__.py        # Azure Function entrypoint: main(req)
│       └── function.json      # httpTrigger binding (anonymous, get+post)
├── tests/
│   └── test_calc.py           # pytest unit tests for src/calc.py
├── host.json                  # Azure Functions host config (version 2.0)
├── local.settings.json.example# template; copy to local.settings.json (gitignored)
├── requirements.txt           # azure-functions, pytest, httpx, python-dotenv
└── README.md                  # Japanese quick-start
```

Notes on structure:

- Business logic lives in `src/calc.py` and is pure/decoupled from Azure. Prefer
  adding or changing calculation logic there and covering it in `tests/`.
- Each function folder (`src/multiply`, `src/divide`) has a `main(req)`
  entrypoint plus a `function.json` binding. The two `__init__.py` entrypoints
  share near-identical helpers `_get_param()` and `_want_html()` (copied, not
  shared). If you change request parsing/HTML behavior, update **both**.
- `src/common.py` helpers exist but are currently unused by the function
  entrypoints.

## Environment

- CI and the project target **Python 3.11**. The dev sandbox may have a newer
  Python (e.g. 3.12); the tests still pass, but keep code compatible with 3.11.
- There is **no** `pyproject.toml`, `setup.cfg`, `pytest.ini`, `conftest.py`,
  linter config, or formatter config. Do not assume any linter/formatter is
  configured—none is. Do not add one unless the task explicitly asks.

## Build, test, and run

### Install dependencies (always do this first in a fresh environment)

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### Run the tests — IMPORTANT gotcha

The tests import the code as a package: `from src.calc import multiply, divide`.
Because there is no `conftest.py` or pytest config, the repository root is **not**
automatically added to `sys.path`.

- ❌ `pytest -q` run from the repo root **fails** with
  `ModuleNotFoundError: No module named 'src'`.
  (Note: the CI workflow `.github/workflows/ci.yml` invokes this same bare
  `pytest -q` command, so CI is susceptible to the same failure. Do not treat
  bare `pytest` as the reliable way to run the tests.)
- ✅ Use one of these instead, from the repository root:

```bash
python -m pytest -q          # preferred: `python -m` puts the repo root on sys.path
# or
PYTHONPATH=. pytest -q
```

Expected result: `5 passed`.

When adding tests, run them the same way. Keep new unit tests for pure logic in
`tests/` and import from `src.…`.

### Run the API locally

Requires **Azure Functions Core Tools** (provides the `func` command), which is
not installed by `pip install -r requirements.txt`. If `func` is unavailable in
the environment, you cannot start the HTTP host—validate logic via `pytest`
instead.

```bash
cp local.settings.json.example local.settings.json
func start
```

`local.settings.json` is gitignored; never commit it.

## Conventions & expectations

- Monetary/precision math uses `decimal.Decimal` (see `src/calc.py`, precision
  set to 28). Preserve this—do not switch to float arithmetic in the core logic.
- Invalid/missing parameters return HTTP `400` with a JSON body shaped like
  `{"error": "BadRequest", "message": "..."}`. Match this shape for new errors.
- Division by zero must stay a `200` with `result: "Infinity"`.
- Requirement docs are authoritative for intended behavior; check
  `docs/requirements.md` before changing API semantics.

## Validating your changes before finishing

1. `pip install -r requirements.txt` (if not already installed).
2. `python -m pytest -q` from the repo root and confirm all tests pass.
3. If you changed request parsing or response formatting in a function
   entrypoint, mirror the change across both `src/multiply/__init__.py` and
   `src/divide/__init__.py`.

Following the commands above (especially `python -m pytest`) avoids the most
common failure in this repo and lets you validate quickly.
