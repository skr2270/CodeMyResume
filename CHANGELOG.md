# Changelog

All notable changes to **Code My Resume (CMR)** will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Complete Specification Suite (`docs/` & Root):**
  - [`docs/PRD.md`](./docs/PRD.md): Formalized CLI contracts (`cmr scan`, `cmr init-context`, `cmr generate`, `cmr diff`), exit codes, `config.yaml`, and `career_context.yaml` schema examples.
  - [`docs/SECURITY.md`](./docs/SECURITY.md): Threat model, zero-read secret file blocklist, `sanitizer.py` regex redaction rules, and Ollama offline guarantee.
  - [`docs/TESTING_AND_QA.md`](./docs/TESTING_AND_QA.md): Offline test pyramid, topology/leak golden gates, `pdfplumber` fill-bounds/hyperlink assertions, and LinkedIn char-cap gates.
  - [`CONTRIBUTING.md`](./CONTRIBUTING.md) & [`CHANGELOG.md`](./CHANGELOG.md).
- **Architecture & Packaging Upgrades (`5363e56`):**
  - Upgraded `pyproject.toml` and `requirements.txt` to unified `google-genai`, `pdfplumber`, `httpx`, Python 3.10 `tomli`, `setuptools` `src`-layout discovery, and `[dev]` test dependencies (`pytest`, `ruff`).
  - Added scan-first workflow (`cmr scan` across 100% of `*.md` files $\rightarrow$ `cmr init-context --from-scan` $\rightarrow$ `cmr generate`), `project_groups`, Map-Reduce chunking + `.cmr/cache/` SHA-256 digest cache, `sanitizer.py`, Two-Pass ReportLab PDF layout calibration, and `linkedin_exporter.py`.

### Removed
- Deleted stray 0-byte `--db` root file and added `.cmr/`, `.pytest_cache/`, `.ruff_cache/`, and `--db` to `.gitignore`.

## [0.1.0-alpha] - 2026-11-09

### Added
- Initial repository creation, MIT `LICENSE`, `README.md`, and foundational design documents (`ARCHITECTURE.md`, `PARSERS_SPEC.md`, `PROMPT_ENGINEERING.md`, `ROADMAP.md`).
