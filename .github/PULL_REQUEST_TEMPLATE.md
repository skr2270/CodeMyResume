# Pull Request Checklist

## Summary of Changes
<!-- Describe what this PR changes and which phase/milestone (e.g., Phase 1 Scanner, Phase 2 Exporter) it addresses. -->

## Linked Issues / Spec Sections
<!-- Reference affected docs (e.g., docs/PARSERS_SPEC.md, docs/EXPORTERS_SPEC.md, docs/PRD.md). -->

## Verification Checklist
- [ ] Read relevant architecture/parser/exporter specs before modifying contracts or schemas
- [ ] All unit tests in `tests/` run **100% offline** with mocked `LLMClient` and `tmp_path` fixtures
- [ ] All newly parsed strings pass through `sanitizer.py` before caching or cloud LLM calls
- [ ] Spec docs updated if `TAXONOMY_RULES`, `ResumeAST`, CLI flags, or exporter rules changed
- [ ] `ruff check src tests` and `pytest -v` pass cleanly
