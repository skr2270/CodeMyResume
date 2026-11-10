# Exporters & Multi-Format Rendering Specification (`EXPORTERS_SPEC.md`)

All exporters in `src/cmr/exporters/` **must be pure, stateless renderers**. They accept a validated `ResumeAST` (`resume_ast.json`) and formatting configuration—never making network requests or LLM calls at export time.

---

## 1. Shared Purity & Formatting Rules

1. **Single Source of Truth:** `markdown_exporter.py`, `html_exporter.py`, `pdf_exporter.py`, `github_exporter.py`, and `linkedin_exporter.py` (inside `src/cmr/exporters/`) read exclusively from `ResumeAST`.
2. **Pre-Sanitized Input Guarantee:** `validator.py` runs before any exporter executes, ensuring zero leaked IPs, API tokens, excluded directories, or `[REDACTED_*]` placeholders exist inside `ResumeAST`.
3. **Neutral Academic Rendering (No Education "From" Dates):** All exporters omit education `start_date` ("from" dates) by default per ATS best practices—rendering only `Expected` completion dates for ongoing degrees (`Expected: 2026 / 2027`) and concise graduation dates for completed degrees (`Graduated: April 2022`).

---

## 2. ATS Markdown Exporter (`markdown_exporter.py`)

* **Canonical Section Headings:** Emits clean uppercase GitHub Flavored Markdown headings recognized by Workday, Greenhouse, and Lever parsers:
  * `# <FULL_NAME>` + subtitle + single-line pipe-separated contact links
  * `## PROFESSIONAL SUMMARY`
  * `## TECHNICAL COMPETENCIES & ECOSYSTEM`
  * `## PROFESSIONAL EXPERIENCE`
  * `## EDUCATION`
* **Single-Column Discipline:** Never wraps experience descriptions inside multi-column Markdown tables so automated ATS tokenizers preserve chronological reading order.
* **Competency Matrix Format:** Emits bold category rows (`- **Category:** Skill1, Skill2, ...`) derived directly from `ResumeAST.competency_matrix`.

---

## 3. Dynamic Two-Pass ReportLab PDF Exporter (`pdf_exporter.py`)

### 3.1 Page Geometry & Supported Paper Sizes
| Setting | `A4` *(Default)* | `LETTER` *(US Roles)* |
| :--- | :--- | :--- |
| **Page Dimensions (`width x height`)** | `595.27 x 841.89 pt` | `612.00 x 792.00 pt` |
| **Default Margins (`L / R / T / B`)** | `36 / 36 / 30 / 34 pt` | `36 / 36 / 30 / 34 pt` |
| **Printable Width** | `523.27 pt` | `540.00 pt` |

### 3.2 Pagination, Orphan & Glyph Rules
1. **Zero Manual `PageBreak()` Calls:** Content flows naturally across `target_pages` (`1` or `2`). Page boundaries never leave artificial dead bottom whitespace.
2. **Orphan & Detachment Prevention:**
   * Set `keepWithNext=True` on section headings and italicized role taglines.
   * Wrap `[job_header, job_tag, first_bullet(s)]` inside `reportlab.platypus.KeepTogether(...)` so a role title never sits stranded at the bottom of Page 1.
3. **Dynamic Job-Header Column Sizing:** Calculate right-column width per `ExperienceBlock` using `pdfmetrics.stringWidth(right_text, font_name, font_size) + 8.0` so long date/location headers (e.g., `"June 2019 – September 2020 | Hyderabad & Bangalore"`) never wrap their final city word onto a second orphan line.
4. **Unicode TTF & Glyph Safety:**
   * Bundle an OFL/Apache TrueType font family (e.g., `Inter` or `DejaVuSans`) in `src/cmr/assets/fonts/` (shipped via `[tool.setuptools.package-data]`) with OS font lookup fallback and WinAnsi sanitization for Type1 fallback mode—preventing black box (`■`) glyph failures on `₹`, em-dashes, or symbols.
5. **Active Clickable Annotations:** Email (`mailto:`), LinkedIn, and GitHub header links render as ReportLab `<a href="...">` annotations verified via `pdfplumber`.
6. **Footer Callback Isolation:** `onFirstPage` and `onLaterPages` render subtle footer text at `y <= 18 pt` (ReportLab bottom-up origin), cleanly separated below `bottomMargin = 34 pt`.

### 3.3 Two-Pass Density & Balance Calibration
Because `ReportLab` uses bottom-up coordinates (`y = 0` at bottom) while `pdfplumber` uses **top-down coordinates (`top = 0` at top, `bottom = page.height` at bottom)**:

1. **Footer Band Exclusion:** Exclude characters rendered in the running footer region (`char["top"] >= page.height - bottom_margin + 4.0`).
2. **Proportional Fill Ratio Formula (Paper-Size Independent):**
   $$\text{printable\_height} = \text{page.height} - \text{top\_margin} - \text{bottom\_margin}$$
   $$\text{fill\_ratio}(p) = \frac{\max(\text{char["bottom"]} \text{ in body of page } p) - \text{top\_margin}}{\text{printable\_height}}$$
3. **Iterative Calibration Search:**
   * **Pass 1:** Render at baseline typography (`body_font=8.3pt`, `leading=11.6pt`, `job_spacer=6pt`). Measure `len(pdf.pages)`, `fill_ratio(1)`, and `fill_ratio(last)`.
   * **Pass 2+ Adjustment:** Scale typography monotonically across bounded ranges:
     * `body_font` in `[7.6pt, 8.8pt]` (`0.15pt` steps), `leading = body_font * 1.38`, and vertical spacer scale `[3pt, 8pt]`.
     * Scale **down** if `len(pages) > target_pages` or if the final page overflows thinly (`< min_last_page_fill_ratio`).
     * Scale **up** if `len(pages) == target_pages` and `fill_ratio(1) < min_page1_fill_ratio` (`0.88`).
   * **Budget Failure Exit:** If constraints (`len(pages) == target_pages`, `fill_ratio(1) >= 0.88`, `fill_ratio(last) >= 0.65`) cannot be met inside font bounds, exit with **Exit Code `3`**.

---

## 4. Print-Ready HTML Exporter (`html_exporter.py`)

* **Continuous Flow Layout:** Uses a single scrollable document container—never fixed-height `.page` `div` wrappers with `page-break-after: always`.
* **CSS `@page` & Print Controls:**
  ```css
  @page { size: A4; margin: 11mm 14mm; }
  .section-title, .job-header, .job-desc-tag { break-after: avoid; page-break-after: avoid; }
  .bullet-item, .edu-entry { break-inside: avoid; page-break-inside: avoid; }
  ```
* Honors `config.pdf.page_size` (`A4` vs `letter`) identically in `@page`.

---

## 5. GitHub Profile Exporter (`github_exporter.py`) & `cmr diff`

* **Output (`output/README.md`):**
  * Centered developer header, headline, and Shields.io social/contact badges (`Email`, `LinkedIn`, optional portfolio link).
  * Executive technical summary + Markdown competency matrix table (`Category | Core Stack`).
  * **Flagship Engineering Highlights:** Renders `ResumeAST.flagship_projects` with `tagline`, `architecture_summary`, badge/code stack tags, and `repo_url` hyperlinks.
  * **Recent Impact & Education:** Compact summary of verified roles and academic timeline.
* **`cmr diff --target <PATH_OR_OWNER/REPO>`:** Computes a colored unified diff (`rich`) between `./output/README.md` and either a local file path or public GitHub `raw` `README.md` fetched via `httpx` before overwriting.

---

## 6. LinkedIn Profile Pack Exporter (`linkedin_exporter.py`)

Consumes `ResumeAST.linkedin` (`LinkedInSection`, fallback-derived deterministically from `ResumeAST` if absent) and renders `LinkedIn_Profile_Pack.md` containing:
1. **Cleanup Checklist:** Automated reminder list populated from `CareerContext.denied_claims`, excluded employers, and `is_fork_or_low_authorship` filtered languages so stale profile items get removed.
2. **Profile Headline:** Enforces `len(headline) <= 220`.
3. **About Section:** Enforces `len(about) <= 2600` (3-paragraph narrative + categorized `WHAT I BUILD & LEAD` scannable summary + `OPEN TO` + contact).
4. **Experience Entries:** Each role description enforces `len(desc) <= 2000`.
5. **Projects & Education Blocks:** Mirrors `ResumeAST.flagship_projects` and `ResumeAST.education`.
6. **Skills List:** Ordered flat list of verified skills capped strictly at **`<= 50`** entries.

**Deterministic Overflow Guard:** If a synthesized field slightly exceeds a LinkedIn limit, `linkedin_exporter.py` attempts sentence/bullet-boundary compaction first; if still over limit in `--strict` mode, `validator.py` / exporter fails with **Exit Code `2`**.
