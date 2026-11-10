# Contributing to Code My Resume (CMR)

Thank you for contributing to **Code My Resume (CMR)**! Whether you are adding a language manifest parser, expanding `TAXONOMY_RULES`, adding a PDF template, or strengthening ground-truth validation rules, please follow these engineering conventions.

---

## 1. Local Development Setup

```bash
# 1. Clone your fork
git clone https://github.com/<your-username>/CodeMyResume.git
cd CodeMyResume

# 2. Create and activate a virtual environment (Python 3.10+)
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# 3. Install in editable mode with dev dependencies
pip install -e ".[dev]"
```

---

## 2. Before Submitting a Pull Request

1. **Read the Design Specs First:** Check [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md), [`docs/PARSERS_SPEC.md`](./docs/PARSERS_SPEC.md), and [`docs/PROMPT_ENGINEERING.md`](./docs/PROMPT_ENGINEERING.md) so new modules align with `ProjectProfile`, `CareerContext`, and `ResumeAST`.
2. **Keep Tests 100% Offline:** Unit tests in `tests/` must never require live internet calls or real LLM API keys—mock `LLMClient` responses and use `tmp_path` directory fixtures.
3. **Never Bypass Secret Redaction:** Any new parser that feeds text to `engine/` must route extracted strings through `sanitizer.py` first.
4. **Run Checks:**
   ```bash
   ruff check src tests
   pytest -v
   ```

---

## 3. Common Contribution Paths

### Adding a New Package Manifest or `TAXONOMY_RULES` Entry
* Update `TAXONOMY_RULES` and parser specs in [`docs/PARSERS_SPEC.md`](./docs/PARSERS_SPEC.md) alongside `src/cmr/parsers/manifest_parser.py`.
* Add unit test cases with sample manifest payloads in `tests/test_parsers.py`.

### Adding or Tuning Exporters
* Every exporter (`src/cmr/exporters/`) must be a **pure, stateless renderer** accepting `ResumeAST` (`resume_ast.json`).
* PDF exporter changes must preserve zero manual `PageBreak()` pagination and pass `pdfplumber` bounds/hyperlink assertions in `tests/test_exporters.py`.

---

## 4. Commit Convention

CMR follows [Conventional Commits](https://www.conventionalcommits.org/):
* `feat:` new parser, exporter, or CLI flag
* `fix:` bug fix or edge-case handling
* `docs:` specification, PRD, or README updates
* `test:` offline unit tests or golden fixtures
* `refactor:` code cleanup without behavior changes
