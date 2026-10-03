---
title: GitHub Profile AI Reviewer
emoji: 🧪
colorFrom: green
colorTo: blue
sdk: docker
app_port: 7860
pinned: false
---

# GitHub Profile AI Reviewer

FastAPI service that rates a public GitHub user profile out of 100.

The API accepts a GitHub username and returns a `rating_score`, developer level, language breakdown, contribution signals, streak data, and model metadata.

## Folder Structure

```text
app/
  main.py              FastAPI app entrypoint and static frontend serving
  api/                 HTTP routes for health checks and profile analysis
  schemas.py           Request and response models
  service.py           Application service layer
  ai/                  Embedding and scoring logic
  clients/             GitHub API clients and queries
  core/                Settings and logging
  graph/               Active analysis workflow
  static/index.html    Browser API console
backend/               Legacy LangGraph prototype pipeline
experiments/           Standalone prototype files
scripts/               Developer utilities and CLI helpers
src/                   React documentation website
```

## Token Behavior

`GITHUB_TOKEN` is optional.

When `GITHUB_TOKEN` is set, the service uses GitHub GraphQL for richer profile and contribution data. When it is empty, users can still rate any public GitHub username through GitHub's public REST API. Public REST mode is limited by GitHub's unauthenticated rate limits and cannot see private contribution data.

If you hit GitHub's public rate limit, add a valid token to `.env` and restart the server. Public mode keeps its request count low and caches successful GitHub responses for `GITHUB_CACHE_TTL_SECONDS` to avoid wasting quota during repeated tests.

## Environment

Copy `.env.example` to `.env` and adjust only what you need:

```env
GITHUB_TOKEN=
GITHUB_API_URL=https://api.github.com/graphql
GITHUB_REST_API_URL=https://api.github.com
GITHUB_PUBLIC_REPO_LIMIT=20
GITHUB_FETCH_COMMIT_COUNTS=false
GITHUB_CACHE_TTL_SECONDS=900
APP_HOST=0.0.0.0
APP_PORT=8000
SCORING_BACKEND=heuristic
```

Keep `SCORING_BACKEND=heuristic` unless you have trained model weights. The neural backend is experimental.

## Run Locally

Run the React documentation website from the repository root:

```powershell
npm install
npm run dev
```

Open:

```text
http://localhost:5173/
```

Run the FastAPI backend separately:

On Windows PowerShell, use a project virtual environment:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

If `py -3.12` is not installed, use your available Python launcher version:

```powershell
py -0p
```

The default requirements start the API with deterministic fallback embeddings. Optional ML packages are separate:

```powershell
python -m pip install -r requirements-ml.txt
```

On macOS/Linux:

```bash
python3 -m pip install -r requirements.txt
python3 -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Open the browser console UI:

```text
http://localhost:8000/
```

Request a rating:

```bash
curl -X POST http://localhost:8000/analyze \
  -H "Content-Type: application/json" \
  -d '{"username":"octocat"}'
```

## Deploy With Docker

```bash
docker compose up --build -d
```

The API and `index.html` console will be available on `http://localhost:${APP_PORT:-8000}`.

Useful deployment commands:

```bash
docker compose ps
docker compose logs -f api
docker compose down
```

For a cloud VM, install Docker, clone the repo, create `.env`, then run the same `docker compose up --build -d` command. If you deploy behind a domain or reverse proxy, forward public traffic to container port `8000`.

Build the React documentation site:

```bash
npm install
npm run build
```

The static output is generated in `dist/`. You can deploy `dist/` to Netlify, Vercel, GitHub Pages, Nginx, or any static hosting provider.

## Use The API In `index.html`

The frontend at `app/static/index.html` calls the API with `fetch`:

```html
<script>
  async function analyze(username) {
    const response = await fetch("/analyze", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ username }),
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.detail || "Analyze request failed");
    }

    return response.json();
  }
</script>
```

Use a full base URL only when the HTML is hosted somewhere else:

```js
fetch("https://your-domain.com/analyze", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ username: "octocat" }),
});
```

## Response Fields

- `rating_score`: main score from 0 to 100.
- `hiring_readiness_score`: compatibility alias for the same 0 to 100 score.
- `public_activity.public_commits`: public commits counted for the user.
- `public_activity.public_prs_created`: public pull requests created by the user.
- `model_info.data_source`: `graphql` when a server token is configured, otherwise `rest-public`.

In `rest-public` mode, GitHub exposes these counts from recent public events only. With authenticated GraphQL mode, the counts come from GitHub's contribution totals.


## GitHub Action: Auto Profile Review + Code Diff Review on PR Open

Two workflows work together to review every pull request, including ones from forks:

**`.github/workflows/pr-checks.yml`** — runs on every PR (`opened`, `reopened`, `synchronize`), including from forks:

1. Spins up the FastAPI service in the CI runner (plain `uvicorn`, no Docker/Redis needed).
2. Calls `POST /analyze` with the PR author's GitHub username and builds a profile review summary.
3. Reviews the actual code diff: lines changed, `ruff` lint issues on changed non-test `.py` files, and whether tests were touched.
4. Saves both results as a build artifact (does not post comments directly — a PR from a fork gets a read-only token, so it can't).
5. Fails the job if the code diff score is below `CODE_REVIEW_MIN_SCORE` (default `50`), which can be used as a required status check to block merges.

**`.github/workflows/post-pr-comments.yml`** — runs after `pr-checks.yml` finishes, in the repository's trusted context (always has write access, even for fork PRs):

1. Downloads the artifact from the triggering run.
2. Posts the profile review and code diff review as two PR comments.

**Requirements:** no secrets needed — `GITHUB_TOKEN` is provided automatically by GitHub Actions. `SCORING_BACKEND` is fixed to `heuristic` in CI.

**Local testing:** reproduce what the workflow does locally by starting the server (`python -m uvicorn app.main:app --host 0.0.0.0 --port 8000`) and calling `/analyze` with `curl` or the browser console at `http://localhost:8000/`.

## Batch Org-Level Hiring Pipeline

`POST /analyze-org` runs the existing `/analyze` pipeline over every member of a GitHub organization and returns a combined, ranked report. No new scoring logic — it reuses `AnalyzerService.analyze()` for each member.

Request:

```bash
curl -X POST http://localhost:8000/analyze-org \
  -H "Content-Type: application/json" \
  -d '{"org": "your-org-name", "max_members": 50}'
```

`max_members` is optional (defaults to no limit, capped at 500). Members are analyzed concurrently (5 at a time) — if one member's analysis fails, it's recorded in `failures` and the rest of the batch still completes.

Response shape: `{ org, total_members, analyzed_count, failed_count, ranked_results: [...], failures: [...] }`, where `ranked_results` is sorted by `rating_score` descending.

Without `GITHUB_TOKEN`, only members who've made their org membership public are visible (`/orgs/{org}/public_members`). With `GITHUB_TOKEN` set, all members are visible (`/orgs/{org}/members`).





run terminal (source .venv/Scripts/activate && uvicorn $(grep -rl "FastAPI(" --include=*.py app backend 2>/dev/null | head -1 | sed 's/\.py$//; s#/#.#g'):app --reload)