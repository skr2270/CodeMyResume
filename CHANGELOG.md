# Changelog

All notable changes to **Code My Resume (CMR)** will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Pre-Implementation Specification, Hygiene & Export Contract Completion:**
  - [`docs/EXPORTERS_SPEC.md`](./docs/EXPORTERS_SPEC.md): Dedicated specification for ATS Markdown, Two-Pass ReportLab PDF (`A4`/`LETTER`, dynamic column widths, bundled TTF handling, proportional `pdfplumber` coordinates), continuously flowing HTML, GitHub `README.md` + `cmr diff`, and LinkedIn Pack caps.
  - Starter configuration & repository standards: [`config.yaml`](./config.yaml), [`career_context.example.yaml`](./career_context.example.yaml), [`.env.example`](./.env.example), [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md), [`.editorconfig`](./.editorconfig), [`.gitattributes`](./.gitattributes), [`.github/workflows/ci.yml`](./.github/workflows/ci.yml), PR template, and issue templates.
- **Complete Specification Suite (`a51ce41`):**
  - [`docs/PRD.md`](./docs/PRD.md), [`docs/SECURITY.md`](./docs/SECURITY.md), [`docs/TESTING_AND_QA.md`](./docs/TESTING_AND_QA.md), [`CONTRIBUTING.md`](./CONTRIBUTING.md), and [`CHANGELOG.md`](./CHANGELOG.md).
- **Architecture & Packaging Baseline (`5363e56`):**
  - Upgraded `pyproject.toml` and `requirements.txt` to unified `google-genai`, `pdfplumber`, `httpx`, Python 3.10 `tomli`, `setuptools` `src`-layout discovery, and `[dev]` extras.

### Changed
- **Real-Workspace Compatibility & Layering Alignment:**
  - Decoupled offline deterministic parsing (`cmr.parsers`, `cmr scan`) from optional LLM Map-Reduce summarization (`cmr.engine.summarizer`), and aligned `v0.1.0` vs. `v0.2.0` milestones across `README.md` and `docs/ROADMAP.md`.
  - Extended `detector.py`, `manifest_parser.py`, `topology_detector.py`, `git_parser.py`, and `TAXONOMY_RULES` for npm/pnpm `workspaces`, `devDependencies`, modern Nx layouts (excluding `*-e2e`), unprefixed `backend/`/`mobile/` pairs, umbrella multi-platform suites, nested `.git` child aggregation, and Gradle/container runtimes.
  - Added `LinkedInSection`, bullet-level `source_projects` provenance, `denied_claims`, `ai_workflow_note`, and `schema_version` to Pydantic contracts; scrubbed synthetic placeholder examples in `docs/PRD.md` and `docs/PROMPT_ENGINEERING.md`.
  - Configured `.gitignore` exclusions for `.cmr/`, `.pytest_cache/`, `.ruff_cache/`, personal `career_context.yaml`, and stray `--db` handles.

## [0.1.0-alpha] - 2026-11-09

### Added
- Initial repository creation, MIT `LICENSE`, `README.md`, and foundational design documents (`ARCHITECTURE.md`, `PARSERS_SPEC.md`, `PROMPT_ENGINEERING.md`, `ROADMAP.md`).
