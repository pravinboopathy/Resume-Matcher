# Plan: Job Alert Resume Tailoring Pipeline (inside Resume Matcher)

## Context

Build a CLI tool at `tools/job-tailor/` inside the Resume Matcher repo that monitors Gmail for LinkedIn job alerts, fetches JDs, and tailors a LaTeX resume. Instead of treating Claude as a monolithic black box, the pipeline leverages Resume Matcher's existing keyword extraction, gap analysis, truthfulness enforcement, AI phrase cleanup, and LiteLLM multi-provider wrapper.

**Key decision:** Lives inside this repo → direct imports from `apps/backend/app/` (no copying/vendoring).

---

## Architecture

### Pipeline flow:
```
JD text
  → sanitize (_sanitize_user_input from improver.py)
  → extract_keywords (LLM via llm.py's complete_json)
  → pre-tailor keyword match % (deterministic, refiner.py)
  → [skip if below min_match_threshold]
  → Claude: tailor .tex (LLM via llm.py's complete, with structured keywords + truthfulness rules)
  → AI phrase removal (deterministic, refiner.py + refinement.py blacklist)
  → post-tailor keyword gap analysis (deterministic, refiner.py)
  → .tex → pdflatex → PDF + keyword report
```

---

## Project Structure

```
tools/job-tailor/
├── config/
│   ├── config.yaml              # All configuration
│   └── credentials.json         # Gmail OAuth (gitignored)
├── resume/
│   └── base_resume.tex          # Master LaTeX resume
├── src/
│   ├── __init__.py
│   ├── cli.py                   # Click CLI entrypoint
│   ├── gmail_client.py          # Gmail API: fetch LinkedIn alert emails
│   ├── email_parser.py          # Extract job IDs from email HTML
│   ├── linkedin_client.py       # Fetch JDs from LinkedIn guest API
│   ├── adapters.py              # Thin wrappers adapting Resume Matcher for plain text
│   ├── resume_tailor.py         # Claude call: tailor .tex with keywords + truthfulness rules
│   ├── pdf_compiler.py          # pdflatex compilation
│   ├── state.py                 # Dedup via processed_jobs.json
│   └── pipeline.py              # Orchestrator: wires all steps
├── data/
│   ├── processed_jobs.json
│   └── output/                  # PDFs, .tex files, change summaries
└── requirements.txt             # Only NEW deps (gmail, click, beautifulsoup, requests)
```

---

## What to Import from Resume Matcher (not copy)

### From `apps/backend/app/llm.py`
| Function | Purpose | Used In |
|----------|---------|---------|
| `complete()` | Text completion with retries, provider abstraction | `resume_tailor.py` (tailor call) |
| `complete_json()` | JSON completion with extraction + truncation detection | `resume_tailor.py` (keyword extraction) |
| `get_llm_config()` | Config resolution (env > file > defaults) | `pipeline.py` |

### From `apps/backend/app/services/improver.py`
| Function | Lines | Purpose | Used In |
|----------|-------|---------|---------|
| `extract_job_keywords()` | 517-533 | LLM-based structured keyword extraction | `pipeline.py` (pre-tailor) |
| `_sanitize_user_input()` | 47-55 | Strip prompt injection from JD text | `resume_tailor.py` |

### From `apps/backend/app/services/refiner.py`
| Function | Lines | Purpose | Used In |
|----------|-------|---------|---------|
| `_keyword_in_text()` | 38-48 | Word-boundary regex matching | Used by gap analysis |
| `analyze_keyword_gaps()` | 149-198 | Deterministic: missing/injectable/non-injectable keywords | `pipeline.py` (pre + post tailor) |
| `calculate_keyword_match()` | 525-552 | Deterministic: match % | `pipeline.py` |
| `remove_ai_phrases()` | 201-255 | Deterministic: 60+ term blacklist with JD-aware protection | `pipeline.py` (post-tailor) |

### From `apps/backend/app/prompts/`
| Item | File | Purpose |
|------|------|---------|
| `EXTRACT_KEYWORDS_PROMPT` | `templates.py:179-196` | Prompt for keyword extraction LLM call |
| Truthfulness rules | `templates.py:198-227` | 9-rule set embedded in tailor system prompt |
| `AI_PHRASE_BLACKLIST` | `refinement.py:3-68` | Blacklist for phrase removal |
| `AI_PHRASE_REPLACEMENTS` | `refinement.py:71-134` | Replacement mappings |

---

## Adaptation Layer

The Resume Matcher functions expect JSON resume dicts. The CLI uses LaTeX strings. A thin adapter bridges this:

```python
# tools/job-tailor/src/adapters.py

from apps.backend.app.services.refiner import _keyword_in_text
from apps.backend.app.prompts.refinement import AI_PHRASE_BLACKLIST, AI_PHRASE_REPLACEMENTS

def keyword_in_text(keyword: str, text: str) -> bool:
    """Direct passthrough — already works on plain text."""
    return _keyword_in_text(keyword, text)

def analyze_keyword_gaps_text(
    jd_keywords: dict, tailored_text: str, master_text: str
) -> dict:
    """Adapted gap analysis for plain text (not JSON resume dicts).

    Reimplements the loop from refiner.py:149-198 but takes
    plain text instead of calling _extract_all_text() on dicts.
    """
    all_kw = set()
    all_kw.update(jd_keywords.get("required_skills", []))
    all_kw.update(jd_keywords.get("preferred_skills", []))
    all_kw.update(jd_keywords.get("keywords", []))

    missing, injectable, non_injectable = [], [], []
    for kw in all_kw:
        if not _keyword_in_text(kw, tailored_text):
            missing.append(kw)
            if _keyword_in_text(kw, master_text):
                injectable.append(kw)
            else:
                non_injectable.append(kw)

    total = len(all_kw) or 1
    return {
        "missing_keywords": missing,
        "injectable_keywords": injectable,
        "non_injectable_keywords": non_injectable,
        "current_match_percentage": (total - len(missing)) / total * 100,
        "potential_match_percentage": (total - len(non_injectable)) / total * 100,
    }

def remove_ai_phrases_text(tex: str, job_description: str = "") -> tuple[str, list[str]]:
    """AI phrase removal on raw LaTeX string.

    Reimplements the inner clean_text() logic from refiner.py:201-255
    without the recursive dict walker.
    """
    import re
    jd_lower = job_description.lower()
    jd_protected = {p.lower() for p in AI_PHRASE_BLACKLIST if p.lower() in jd_lower}

    removed = []
    for phrase in AI_PHRASE_BLACKLIST:
        if phrase.lower() in jd_protected:
            continue
        if phrase.lower() in tex.lower():
            removed.append(phrase)
            replacement = AI_PHRASE_REPLACEMENTS.get(phrase.lower(), "")
            tex = re.compile(re.escape(phrase), re.IGNORECASE).sub(replacement, tex)
    return tex, removed
```

---

## Module Implementations

### `src/resume_tailor.py` — The Claude Call

**Changes from original spec:**

1. **Uses LiteLLM wrapper** (`apps/backend/app/llm.py:complete()`) instead of direct `anthropic.Anthropic()`
2. **Feeds structured keywords** from `extract_job_keywords()` into the user message
3. **Adopts truthfulness rules** from `apps/backend/app/prompts/templates.py:198-227`
4. **Sanitizes JD** with `_sanitize_user_input()` before including in prompt
5. **Keeps delimiter-based output format** (`---ANALYSIS---`, `---LATEX---`, `---CHANGES---`) — this is better than JSON for LaTeX content since LaTeX has lots of special characters

System prompt additions from Resume Matcher:
```
CRITICAL TRUTHFULNESS RULES:
1. DO NOT add any skill, tool, technology not in the original resume
2. DO NOT invent numeric achievements
3. DO NOT add company/product names not in original
4. DO NOT upgrade experience level
5. DO NOT add languages/frameworks candidate hasn't used
6. DO NOT extend employment dates
7. [strategy-dependent: nudge/keywords/full]
8. Preserve factual accuracy
9. NEVER remove existing skills, certifications, languages, or awards
```

### `src/pipeline.py` — Updated Orchestrator

Key changes from spec:

```python
# Step 1: Extract keywords (LLM call via llm.py)
from apps.backend.app.services.improver import extract_job_keywords
jd_keywords = await extract_job_keywords(posting.description)

# Step 2: Pre-tailor match assessment (deterministic)
from .adapters import analyze_keyword_gaps_text
pre_analysis = analyze_keyword_gaps_text(jd_keywords, base_tex, base_tex)
pre_match = pre_analysis["current_match_percentage"]

if pre_match < config["min_match_threshold"]:
    print(f"  Skipping: {pre_match:.0f}% match (below {config['min_match_threshold']}%)")
    state.mark_processed(ref.job_id, {"skipped": True, "skip_reason": "low_match"})
    continue

# Step 3: Tailor (LLM call via llm.py, with keywords in prompt)
result = await tailor_resume(base_tex, posting, jd_keywords)

# Step 4: Post-process (deterministic)
from .adapters import remove_ai_phrases_text
tailored_tex, removed_phrases = remove_ai_phrases_text(result.tailored_tex, posting.description)

# Step 5: Post-tailor analysis (deterministic)
post_analysis = analyze_keyword_gaps_text(jd_keywords, tailored_tex, base_tex)
```

### `src/gmail_client.py`, `src/email_parser.py`, `src/linkedin_client.py`
Unchanged from original spec — no Resume Matcher overlap.

### `src/pdf_compiler.py`
Unchanged from original spec — uses pdflatex, not Playwright.

### `src/state.py`
Unchanged from original spec — flat JSON file, not TinyDB.

---

## Output Per Job

Each job produces three files plus an enhanced report:

```
data/output/
├── Files.com_Infrastructure_Engineer_2026-04-01.pdf
├── Files.com_Infrastructure_Engineer_2026-04-01.tex
└── Files.com_Infrastructure_Engineer_2026-04-01_changes.md
```

The `_changes.md` now includes a keyword analysis section:

```markdown
# Infrastructure Engineer (Remote) at Files.com

**Apply:** https://www.linkedin.com/jobs/view/4392944812
**Salary:** $110,000 - $250,000
**Pre-tailor match:** 45% → **Post-tailor match:** 78%
**Potential (with all injectable keywords):** 89%

## Analysis
[Claude's analysis of the role and alignment]

## Changes Made
[Claude's explanation of what changed and why]

## Keyword Report

### Matched (28 of 36 keywords):
Python, AWS, Terraform, CI/CD, Docker, ...

### Missing but injectable (in your resume, not yet reflected):
- Kubernetes (mentioned in your DevOps project)
- Grafana (in your monitoring experience)

### Gaps (not in your resume — cannot add truthfully):
- Chef, Puppet, Datadog

### AI phrases cleaned:
- "spearheaded" → "led"
- "leveraged" → "used"
- "cutting-edge" → "modern"
```

---

## Config

```yaml
gmail:
  credentials_path: "config/credentials.json"
  token_path: "config/token.json"
  query: "from:jobs-noreply@linkedin.com newer_than:1d"
  max_results: 20

linkedin:
  search_queries:
    - keywords: "infrastructure engineer"
      location: "United States"
      time_filter: "r86400"
  request_delay_seconds: 3

llm:
  # Uses Resume Matcher's config resolution:
  # env vars (ANTHROPIC_API_KEY etc.) > config > defaults
  provider: "anthropic"
  model: "claude-sonnet-4-20250514"
  max_tokens: 8192

resume:
  base_tex_path: "resume/base_resume.tex"

tailoring:
  strategy: "full"                # nudge | keywords | full
  enable_keyword_extraction: true
  enable_ai_phrase_removal: true
  min_match_threshold: 30         # skip jobs below this pre-tailor match %

output:
  directory: "data/output"
  filename_pattern: "{company}_{job_title}_{date}"

state:
  path: "data/processed_jobs.json"
```

---

## Dependencies

```
# tools/job-tailor/requirements.txt (NEW deps only — Resume Matcher deps assumed available)
google-api-python-client>=2.100.0
google-auth-oauthlib>=1.0.0
google-auth-httplib2>=0.1.0
beautifulsoup4>=4.12.0
click>=8.1.0
requests>=2.31.0
pyyaml>=6.0
```

System: `pdflatex` (from `brew install basictex` or `apt install texlive`)

---

## Python Path Setup

Since the CLI lives at `tools/job-tailor/` and imports from `apps/backend/app/`:

```python
# tools/job-tailor/src/__init__.py
import sys
from pathlib import Path

# Add repo root to path so we can import from apps/backend/app/
REPO_ROOT = Path(__file__).resolve().parent.parent.parent.parent
sys.path.insert(0, str(REPO_ROOT / "apps" / "backend"))
```

---

## Implementation Steps

1. **Create directory structure** — `tools/job-tailor/` with all subdirs
2. **Write `src/adapters.py`** — thin wrappers that adapt Resume Matcher's JSON-oriented functions for plain text
3. **Write `src/resume_tailor.py`** — Claude call using `llm.py`'s `complete()`, with structured keywords and truthfulness rules
4. **Write `src/pipeline.py`** — orchestrator with pre/post keyword analysis and AI phrase removal
5. **Write `src/gmail_client.py`** — Gmail API client (from original spec, unchanged)
6. **Write `src/email_parser.py`** — LinkedIn email parser (from original spec, unchanged)
7. **Write `src/linkedin_client.py`** — LinkedIn guest API client (from original spec, unchanged)
8. **Write `src/pdf_compiler.py`** — pdflatex wrapper (from original spec, unchanged)
9. **Write `src/state.py`** — JSON file dedup (from original spec, unchanged)
10. **Write `src/cli.py`** — Click CLI with `run`, `job`, `status`, `test-gmail`, `test-linkedin` commands
11. **Write `config/config.yaml`** — default configuration
12. **Add `tools/job-tailor/` to `.gitignore`** for `credentials.json`, `token.json`, `data/output/`

---

## Verification

1. **Import test:** `cd tools/job-tailor && python -c "from app.services.refiner import _keyword_in_text; print('OK')"` — verify path setup works
2. **Keyword extraction:** Run `extract_job_keywords()` on a sample JD, verify structured output
3. **Gap analysis:** Run `analyze_keyword_gaps_text()` with known base .tex and JD keywords, verify missing/injectable classification
4. **AI phrase removal:** Feed Claude-generated LaTeX through `remove_ai_phrases_text()`, verify replacements
5. **Single job e2e:** `python -m src.cli job <linkedin_job_id>` — verify PDF + .tex + changes.md with keyword report
6. **Threshold filter:** Set `min_match_threshold: 80`, run against a weak-match job, verify it's skipped
7. **Gmail flow:** `python -m src.cli test-gmail` then `python -m src.cli run --source email`
8. **Dedup:** Run twice — second run skips all previously processed jobs
