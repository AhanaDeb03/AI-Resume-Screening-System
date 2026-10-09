# AI Resume Screening and Ranking System

This CLI screens a folder of resumes for SDE interns who show both real Python evidence and practical AI, LLM, RAG, or agentic-system exposure. It processes the whole batch without stopping on one malformed file, applies the hard filter before ranking, enriches public GitHub profiles in a bounded and cached pass, and writes an explainable JSON report.

## What it produces

`results.json` contains:

- a highest-score-first shortlist with rank, score breakdown, supporting evidence, project summary, strengths, concerns, and GitHub summary;
- all rejected candidates with explicit hard-filter reasons;
- parsing failures without losing the rest of the batch;
- an all-candidates list, duplicate-file audit, and batch totals.

The supported inputs are PDF (required), plus DOCX and TXT (bonus). PDF text is extracted with `pypdf`.

## Setup

Use Python 3.11 or newer.

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env
```

Environment variables can be set in your shell or loaded by your preferred dotenv tool. The application never hard-codes credentials.

## Run

```bash
python main.py --input ./resumes --output ./output/results.json
```

Create the same JSON plus a portable, static insights page with no server or external frontend:

```bash
python main.py --input ./resumes --output ./output/results.json --html-report ./output/insights.html
```

Open `output/insights.html` locally to filter candidates by name/skill/status and inspect their evidence, score, GitHub summary, parser method, and detected sections.

For a completely offline, repeatable run, use `--skip-github`.

```bash
python main.py --input ./resumes --output ./output/results.json --skip-github
```

GitHub enrichment is enabled by default when a profile URL exists. It makes one public Events API request per unique username, limits concurrent requests to `MAX_NETWORK_WORKERS` (default 6), continues after rate limits or failures, and stores successful calls in `output/.cache/github.json` for 24 hours by default. Set `GITHUB_TOKEN` to increase public API limits; it is read only from the environment.

### Optional structured LLM enrichment

The deterministic pipeline is the default. To add a small, replaceable OpenAI-compatible JSON-mode adapter for eligible candidates, set `LLM_API_KEY` (and optionally `LLM_BASE_URL` and `LLM_MODEL`) and add `--enable-llm`:

```bash
python main.py --input ./resumes --output ./output/results.json --enable-llm
```

The adapter validates its JSON response with a Pydantic schema and may update only the project summary, strengths, and concerns. It never controls hard eligibility, and one model failure is recorded on that candidate without failing the batch.

## Design Decisions

### Filtering strategy

Hard eligibility is deterministic and runs before any ranking: a resume must contain Python evidence and at least one concrete AI/LLM/RAG/agentic implementation or framework signal. JavaScript, Java, React, and Next.js are not negative signals when both requirements are met. Each decision retains compact source-text evidence; candidates rejected by the filter cannot receive a rank.

### Scoring strategy

The baseline follows the assignment's 100-point model:

| Category | Maximum | Evidence rewarded |
| --- | ---: | --- |
| AI, agentic, or RAG project depth | 40 | Agents, retrieval, embeddings/vector search, tools, state/orchestration, evaluation, and data/model workflows |
| Python and backend engineering | 30 | Python, FastAPI, Django/Flask, async, PostgreSQL, and Redis |
| Cloud, deployment, and full stack | 15 | Cloud, Docker, Kubernetes, deployment/CI, React, and Next.js |
| GitHub activity | 10 | Recent public activity plus maintained/relevant repository names seen in recent public events |
| Engineering depth | 5 | Testing, caching, queues, architecture, concurrency, monitoring, and failure handling |

AI points remain low when the extractor finds only a framework/keyword and no project-like context. The report can apply a 5–15 point project-quality penalty for explicit thin-wrapper or tutorial-style cues. The final score is capped at the category maxima, includes GitHub at 10 or fewer points, and is paired with evidence rather than a black-box ranking.

### LLM usage

LLM use is optional so the core system is testable, fast, and usable without credentials. The `StructuredLLMAdapter` uses a Pydantic JSON schema and is isolated in `resume_screening/llm.py`; it is safe to replace with another provider. Hard filters remain outside the LLM, and any LLM error leaves the deterministic output usable.

### GitHub scoring

Without a token, the lightweight enrichment uses public GitHub Events, not scraping. With `GITHUB_TOKEN`, it first uses one authenticated GitHub GraphQL query per profile to inspect public repository languages, stars, and push dates; it falls back safely to the public Events API when GraphQL cannot be used. Recent activity gives 0–5 points and maintained/relevant repositories give 0–5 points. A missing, private, rate-limited, or failing profile produces a clear status and zero points rather than a batch failure. Requests are deduplicated by username, bounded, and cached per run directory.

## Optional extensions

### OCR and section detection

The normal PDF parser remains the fast default. For a textless/image-only PDF, install the optional dependencies plus the Tesseract executable, then enable the fallback:

```bash
pip install -r requirements-optional.txt
python main.py --input ./resumes --output ./output/results.json --enable-ocr
```

OCR runs only after normal text extraction fails. Every parsed candidate now records the parser used, whether OCR was needed, and detected resume sections (summary, skills, experience, projects, education, certifications), without embedding the full resume text in the result.

### Review API with persistent overrides

The optional FastAPI service persists screening runs and reviewer decisions in SQLite. It does not replace the CLI; it exposes the same pipeline for integration or reviewer workflows.

```bash
pip install -r requirements-optional.txt
uvicorn resume_screening.api:create_app --factory --reload
```

Available endpoints:

- `POST /screen` with `{"input_directory":"./resumes"}` to save a run.
- `GET /results` or `GET /results/{run_id}` to retrieve a run with current reviewer overrides applied.
- `POST /reviews` with `source_file`, `decision` (`approve`, `reject`, or `needs_review`), optional notes, and an optional 0–100 score override.

Reviewer changes are explicit, stored separately from the original run, and cause ranks to be rebuilt at read time. An approved profile without a manual score stays at the bottom until a reviewer assigns one, preserving the original automated evidence.

## Tests

```bash
pytest -q
```

The suite covers hard-filter and score behavior using synthetic text, section detection, a golden three-resume batch, static report generation, and persistent reviewer overrides.
