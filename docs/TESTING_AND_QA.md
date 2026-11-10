# Testing Strategy & Quality Assurance (`TESTING_AND_QA.md`)

**Code My Resume (CMR)** requires deterministic, fast, **100% offline-verifiable** automated tests across all core modules before any LLM API calls occur.

---

## 1. Test Pyramid & Coverage Scope

| Test Suite | Scope | Network / LLM Required? |
| :--- | :--- | :--- |
| **`tests/test_scanner.py`** | Top-down directory pruning (`walker.py`), `node_modules`/`.venv`/`vault/temp/` skipping, and polyglot multi-repo suite grouping (`detector.py`). | **No** (`tmp_path` fixtures) |
| **`tests/test_parsers.py`** | `manifest_parser.py` (Python 3.10 `tomli` + `package.json`), `topology_detector.py` (Nx microservices vs. NestJS modular monolith), `infra_detector.py`, `doc_parser.py` (Tier 1A/1B/2 + Map-Reduce chunking), `git_parser.py` (`< 5%` fork filtering), and `sanitizer.py` redaction. | **No** (mocked git/docs) |
| **`tests/test_engine.py`** | `context_merger.py` (`project_groups`, excluded dirs) + `validator.py` catching topology violations, legacy leaks (`Keycloak`/`Kubernetes`), ungrounded metrics, IP leaks, and LinkedIn char-cap overflows. | **No** (mocked `LLMClient`) |
| **`tests/test_exporters.py`** | Pure stateless rendering of golden `ResumeAST` fixtures into Markdown, HTML, ReportLab PDF, GitHub README, and LinkedIn Pack. | **No** |

---

## 2. Mandatory Quality Gates (Golden Assertions)

### 2.1 Structural Topology & Anti-Hallucination Gates
* **Fixture A (`modular_monolith`):** Single `src/main.ts` + `src/app.module.ts` importing 21 feature modules $\rightarrow$ `topology == "modular_monolith"`, `validator.py` rejects any bullet containing `"microservices"`.
* **Fixture B (`distributed_microservices`):** `nx.json` + 9 `apps/*/src/main.ts` $\rightarrow$ `topology == "distributed_microservices"`, `service_or_module_count == 9`.
* **Fixture C (`fork_leak`):** Forked Go repo with `< 5%` user commits $\rightarrow$ `is_fork_or_low_authorship == True`; `Go` must not enter `ResumeAST.competency_matrix`.

### 2.2 Secret & IP Redaction Gate (`sanitizer.py`)
* Given an input doc containing `ssh root@103.160.106.186`, `AIzaSyDummyApiKey123`, `postgres://admin:secret@db.internal:5432/prod`, and an `.env` sibling file:
  * `.env` must never be opened (verified via mock/spy).
  * Output digest must replace all sensitive strings with `[REDACTED_*]` placeholders.

### 2.3 PDF Layout & Density Gate (`pdfplumber` Assertions)
Every compiled PDF in `tests/test_exporters.py` is inspected programmatically with `pdfplumber`:
1. `len(pdf.pages) == target_pages` (default: `2`).
2. Page 1 body fill ratio $\ge 88\%$ (`(page.height - bottom_margin) - body_bottom_y <= 50.0` pt).
3. Final page body fill ratio $\ge 65\%$ (`body_bottom_y >= 540.0` pt on A4).
4. Active clickable hyperlinks exist (`len(pdf.pages[0].hyperlinks) >= 3` for Email, LinkedIn, GitHub).
5. Zero orphan wrapping on job headers and zero black-box glyph replacement characters (`■`).

### 2.4 LinkedIn Character-Limit Gate (`linkedin_exporter.py`)
Automated assertions verify exact character limits on rendered `LinkedIn_Profile_Pack.md`:
* `len(headline) <= 220`
* `len(about) <= 2600`
* `all(len(exp_desc) <= 2000 for exp_desc in experience_blocks)`
* `len(skills) <= 50`

---

## 3. Running Tests & Linting Locally

```bash
# Activate virtual environment
.venv\Scripts\activate   # macOS/Linux: source .venv/bin/activate

# Install editable package with dev tools
pip install -e ".[dev]"

# Run unit & exporter verification tests
pytest -v

# Run linter and formatting checks
ruff check src tests
```

---

## 4. Windows Path & Cross-Platform Edge Cases

* All filesystem paths in `walker.py` and `git_parser.py` must use `pathlib.Path` / `os.path` normalization (`/` vs `\`), handle spaces in folder names (e.g., `PDF Consolidator`), and safely handle Windows `PermissionError` or cloud-hydration placeholders without crashing directory traversal.
