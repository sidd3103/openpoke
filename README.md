# OpenPoke 🌴

OpenPoke is a simplified, open-source take on [Interaction Company’s](https://interaction.co/about) [Poke](https://poke.com/) assistant—built to show how a multi-agent orchestration stack can feel genuinely useful. It keeps the handful of things Poke is great at (email triage, reminders, and persistent agents) while staying easy to spin up locally.

- Multi-agent FastAPI backend that mirrors Poke's interaction/execution split, powered by [OpenRouter](https://openrouter.ai/).
- Gmail tooling via [Composio](https://composio.dev/) for drafting/replying/forwarding without leaving chat.
- Trigger scheduler and background watchers for reminders and "important email" alerts.
- Next.js web UI that proxies everything through the shared `.env`, so plugging in API keys is the only setup.

## Requirements
- Python 3.10+
- Node.js 18+
- npm 9+

## Quickstart
1. **Clone and enter the repo.**
   ```bash
   git clone https://github.com/shlokkhemani/OpenPoke
   cd OpenPoke
   ```
2. **Create a shared env file.** Copy the template and open it in your editor:
   ```bash
   cp .env.example .env
   ```
3. **Get your API keys and add them to `.env`:**
   
   **OpenRouter (Required)**
   - Create an account at [openrouter.ai](https://openrouter.ai/)
   - Generate an API key
   - Replace `your_openrouter_api_key_here` with your actual key in `.env`
   
   **Composio (Required for Gmail)**
   - Sign in at [composio.dev](https://composio.dev/)
   - Create an API key
   - Set up Gmail integration and get your auth config ID
   - Replace `your_composio_api_key_here` and `your_gmail_auth_config_id_here` in `.env`
4. **(Required) Create and activate a Python 3.10+ virtualenv:**
   ```bash
   # Ensure you're using Python 3.10+
   python3.10 -m venv .venv
   source .venv/bin/activate
   
   # Verify Python version (should show 3.10+)
   python --version
   ```
   On Windows (PowerShell):
   ```powershell
   # Use Python 3.10+ (adjust path as needed)
   python3.10 -m venv .venv
   .\.venv\Scripts\Activate.ps1
   
   # Verify Python version
   python --version
   ```

5. **Install backend dependencies:**
   ```bash
   pip install -r server/requirements.txt
   ```
6. **Install frontend dependencies:**
   ```bash
   npm install --prefix web
   ```
7. **Start the FastAPI server:**
   ```bash
   python -m server.server --reload
   ```
8. **Start the Next.js app (new terminal):**
   ```bash
   npm run dev --prefix web
   ```
9. **Connect Gmail for email workflows.** With both services running, open [http://localhost:3000](http://localhost:3000), head to *Settings → Gmail*, and complete the Composio OAuth flow. This step is required for email drafting, replies, and the important-email monitor.

The web app proxies API calls to the Python server using the values in `.env`, so keeping both processes running is required for end-to-end flows.

## Project Layout
- `server/` – FastAPI application and agents
- `web/` – Next.js app
- `server/data/` – runtime data (ignored by git)
- `evals/` – agent-routing evals: synthetic world, frozen cases, harness and grader
- `tests/` – unit and eval tests

## Running the tests
Install the dev dependencies on top of the backend ones, then run pytest from the repo root:
```bash
pip install -r server/requirements.txt -r requirements-dev.txt
python -m pytest -q
```
No API keys or network are needed. Tests that read a past eval run folder are skipped, because run folders are kept locally and are not in the repo (expect about 19 skips on a fresh clone).

## Running the evals
All commands run from the repo root with the virtualenv active. Everything the evals read is committed: the synthetic world (`evals/world/`), the frozen case sets (`evals/cases/v1`, `evals/cases/v2`) and the regenerated agent descriptions. Each run writes a folder under `evals/results/` (ignored by git) and adds one line to `evals/results/EXPERIMENTS.md`.

**Retrieval eval (free, offline).** Does retrieval put the agents a request needs into the top 20, at N = 10, 30, 100 and 300 agents?
```bash
# Case set queries
python -m evals.run retrieval --split test --embedder fastembed --purpose "my retrieval run"
# Look-alike stress roster: two people who each own an agent on the same 48 topics
python -m evals.run retrieval --split test --world dense --embedder fastembed --purpose "my dense run"
```
Use `--embedder fastembed` to match production (the default, `hashing`, is a test stand-in). The first fastembed run downloads the embedding model once, so it needs network.

**Routing eval (paid).** Runs the cases through the real interaction-agent loop with stubbed tools, then grades its tool calls (reuse, create, or ask) by exact match. It needs `OPENROUTER_API_KEY` in `.env`.
```bash
# Prints the cost estimate and stops
python -m evals.run routing --split test --modes legacy_rules,full,hybrid --sizes 10,30,100,300 --runs 3 \
  --purpose "my routing run"
# Add --yes to run it; --max-cost caps actual spend (default $35)
python -m evals.run routing --split test --modes hybrid --sizes 10,300 --runs 1 --max-cost 2 --yes \
  --purpose "small hybrid check"
```
The full test-split command above is estimated at about $4 with the default model (Claude Haiku 5.5). Modes: `hybrid` (production: top 20 retrieved agent cards plus a search tool), `full` (every agent card), `legacy_rules` (agent names only).

**Looking at a run** (no LLM calls), for any run folder on your machine:
```bash
python -m evals.run grid evals/results/<run-folder>                       # case × N pass grid
python -m evals.run grid evals/results/<new> --diff evals/results/<old>   # what changed between two runs
python -m evals.run regrade evals/results/<run-folder>                    # re-grade with the current grader
python -m evals.run inspect evals/results/<run-folder>                    # export to Inspect AI logs
inspect view --log-dir evals/inspect_logs                                 # browse them
```

Past results are summarised one line per run in `evals/results/EXPERIMENTS.md`, and spend in `evals/results/SPEND.md`.

## License
MIT — see [LICENSE](LICENSE).
