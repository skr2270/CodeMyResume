# Testing Strategy & Quality Assurance (`TESTING_AND_QA.md`)

**Code My Resume (CMR)** requires deterministic, fast, **100% offline-verifiable** automated tests across all core modules before any LLM API calls occur, executed in CI across both `ubuntu-latest` and `windows-latest` (`Python 3.10` & `3.12`).

---

## 1. Test Pyramid & Coverage Scope

Shared offline `tmp_path` builders and mocked `LLMClient` fixtures live in `tests/conftest.py`.

| Test Suite | Scope | Network / LLM Required? |
| :--- | :--- | :--- |
| **`tests/test_scanner.py`** | Top-down directory pruning (`walker.py`), skipping `node_modules`/`.venv`/`.next`/`vault/temp/`, and all 4 suite-grouping patterns (`detector.py`). | **No** (`tmp_path` fixtures) |
| **`tests/test_parsers.py`** | `manifest_parser.py` (Python 3.10 `tomli`, `workspaces`, `devDependencies`, `TAXONOMY_RULES`, Gradle), `topology_detector.py`, `infra_detector.py`, `doc_parser.py` (case-insensitive exclusions, full-file heading chunking), `git_parser.py` (multi-`.git` aggregation & `< 5%` fork filtering), and `sanitizer.py`. | **No** (mocked git/docs) |
| **`tests/test_engine.py`** | `summarizer.py` cache hits, `context_merger.py` (`project_groups`, excluded dirs), `llm_client.py` repair loop, and `validator.py` gates. | **No** (mocked `LLMClient`) |
| **`tests/test_exporters.py`** | Pure stateless rendering of golden `ResumeAST` fixtures into ATS Markdown, HTML, Two-Pass ReportLab PDF (`A4` & `LETTER`), GitHub README, and LinkedIn Pack. | **No** |

---

## 2. Mandatory Quality Gates (Golden Assertions)

### 2.1 Structural Topology & Workspace Grouping Gates
* **Fixture A (`modular_monolith`):** Single `src/main.ts` + `src/app.module.ts` importing 21 feature `*Module` classes $\rightarrow$ `topology == "modular_monolith"`, `service_or_module_count == 21`; `validator.py` rejects any bullet referencing that project if `"microservices"` appears.
* **Fixture B (`distributed_microservices`):** Modern Nx `nx.json` (`plugins`, no `"projects"` key) + 9 `apps/*/src/main.ts` + 6 `apps/*-e2e` dirs $\rightarrow$ `topology == "distributed_microservices"`, `service_or_module_count == 9` (`*-e2e` excluded).
* **Fixture C (`fork_leak`):** Forked Go repo with `< 5%` user commits $\rightarrow$ `is_fork_or_low_authorship == True`; `Go` never enters `ResumeAST.competency_matrix`.
* **Fixture D (`multi_git_polyglot_suite`):** Suite root with no `.git`, nested `.git` children (`foo-backend`, `foo-react-native`, `foo-web`) + unprefixed (`backend/`, `mobile/`) variant $\rightarrow$ unified `polyglot_fullstack_workspace`, git stats aggregated across all child `.git` roots.
* **Fixture E (`single_app_npm_workspace`):** Root `package.json` (`deps: []`, `workspaces: ["apps/*"]`) + `apps/web/package.json` (`three`, `vite`, `@mediapipe/tasks-vision`) + ignored `.next/package.json` $\rightarrow$ extracts all `apps/web` deps cleanly while ignoring `.next/`.

### 2.2 Secret & IP Redaction Gate (`sanitizer.py`)
* Given inputs containing `ssh root@103.160.106.186`, `AIzaSyDummyApiKey123`, `postgres://admin:secret@db.internal:5432/prod`, alongside `0.0.0.0`, `127.0.0.1`, `192.168.1.10`, version `16.0.10`, Windows build `10.0.22621.1`, and an `.env` sibling file:
  * `.env` is never opened (verified via spy/mock).
  * Public IP/key/URI strings become `[REDACTED_*]`, while RFC 1918, `0.0.0.0`, `127.0.0.1`, and version strings remain intact.

### 2.3 PDF Layout & Density Gate (`pdfplumber` Assertions)
Using **`pdfplumber`'s top-down coordinate system** (`char["top"]` / `char["bottom"]` from top edge `0` down to `page.height`), excluding the running footer band (`char["top"] >= page.height - bottom_margin + 4.0`):

$$\text{printable\_height} = \text{page.height} - \text{top\_margin} - \text{bottom\_margin}$$
$$\text{fill\_ratio}(p) = \frac{\max(\text{char["bottom"]} \text{ in body of page } p) - \text{top\_margin}}{\text{printable\_height}}$$

Every compiled PDF (`A4` and `LETTER`) in `tests/test_exporters.py` must pass:
1. `len(pdf.pages) == target_pages` (default: `2`).
2. `fill_ratio(page=0) >= 0.88`.
3. `fill_ratio(page=-1) >= 0.65`.
4. Active clickable hyperlinks (`len(pdf.pages[0].hyperlinks) >= 3`).
5. Dynamic job-header widths keep right-hand date/location labels (`"June 2019 – September 2020 | Hyderabad & Bangalore"`) on one line, with zero black-box glyph failures (`■`) on Unicode characters (`₹`, `—`).

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

* All filesystem paths in `walker.py`, `detector.py`, and `git_parser.py` must use `pathlib.Path` normalization (`/` vs `\`), support folder spaces (e.g., `PDF Consolidator`), and safely handle Windows `PermissionError` or cloud-hydration placeholders without crashing directory traversal.
