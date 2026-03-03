# Architectural, Test, and Documentation Debt

This document catalogs technical debt in the AI Game Generator repository as of the analysis date.

---

## 1. Architectural Debt

### 1.1 Security — Critical

| Issue | Location | Description |
|-------|----------|-------------|
| **Hardcoded API keys** | `backend/backend.py` (lines 16–17) | Gemini and Claude API keys are hardcoded. The comment says "NEVER hardcode API keys" but keys are in source. |
| **Real keys in .env.example** | `backend/.env.example` | Example file contains real API key values instead of placeholders. Anyone copying it leaks credentials. |
| **Duplicate hardcoded keys** | `backend/gamemaker.py`, `backend/gameCreator.py` | Additional Gemini (and possibly other) API keys hardcoded in standalone scripts. |
| **No env loading** | `backend/backend.py` | No `python-dotenv` or `os.getenv()`; keys are literal. README instructs users to use `.env`, but the app never loads it. |

**Recommendation:** Use `os.getenv("GEMINI_API_KEY")` / `os.getenv("CLAUDE_API_KEY")` (with `python-dotenv` in `requirements.txt` and `load_dotenv()` in `backend.py`). Replace real keys in `.env.example` with placeholders like `your_gemini_key_here`.

---

### 1.2 Backend structure

| Issue | Location | Description |
|-------|----------|-------------|
| **Monolithic route handlers** | `backend/backend.py` | All routes and business logic (prompt building, AI calls, file I/O) live in one ~400-line file. Hard to test and extend. |
| **No service layer** | `backend/backend.py` | AI client setup, prompt construction, and response parsing are inlined in route handlers. No separation of HTTP vs. domain logic. |
| **Duplicate HTML extraction logic** | `backend/backend.py` vs `backend/gamemaker.py` | `extract_html_from_response()` in backend; `gamemaker.py` uses a different regex-based extraction. Same concern implemented twice. |
| **Unused/alternative pipelines** | `backend/gamemaker.py`, `backend/gameCreator.py` | Two other game-generation flows exist but are not used by the Flask app. Unclear whether they are deprecated or alternate entry points; no shared module for “generate game from prompt.” |
| **File-based storage only** | `backend/backend.py` | Games stored under `games/{uuid}/` with no abstraction. Switching to DB or object storage would require touching many places. |
| **Relative GAMES_DIR** | `backend/backend.py` | `GAMES_DIR = "games"` is relative to CWD. Running the server from a different directory can create games in the wrong place or break paths. |

**Recommendation:** Extract AI and game-storage logic into modules (e.g. `services/prompt_enhancer.py`, `services/game_generator.py`, `storage/game_store.py`). Use a single shared HTML-extraction utility. Consider a `GAMES_DIR` from env with an absolute path default.

---

### 1.3 Frontend structure

| Issue | Location | Description |
|-------|----------|-------------|
| **Hardcoded API base URL** | `Create.jsx`, `Create_new.jsx`, `Game.jsx`, `Remix.jsx` | `http://127.0.0.1:5000` is repeated in multiple components. No shared config or env (e.g. `VITE_API_URL`). |
| **Typo in API URL** | `Remix.jsx` (e.g. line 49) | Uses `http://127.0.1.5000` (missing `0` in 127.0.0.1). Would cause failed requests if Remix were ever used. |
| **Duplicate Create flows** | `Create.jsx` vs `Create_new.jsx` | Two similar game-creation UIs; only `Create.jsx` is used in `App.jsx`. `Create_new.jsx` is dead code and causes maintenance/confusion. |
| **Orphan components** | `MintGame.jsx`, `Remix.jsx` | Not referenced in `App.jsx`. `MintGame.jsx` and `Remix.jsx` import `UserContext` / `useLogin`, but **`UserContext` does not exist** in the repo → these files would fail if ever mounted. |
| **Remix early return** | `Remix.jsx` line 14 | `return (<></>)` before any hooks; the rest of the component is dead code and violates Rules of Hooks if uncommented. |
| **Broken entry point** | `frontend/src/main.jsx` | Uses `ReactDOM.createRoot(root)` but `root` is never defined (should be `document.getElementById('root')`). Duplicate imports (`createRoot` and `ReactDOM`). App may only work if a global `root` exists elsewhere. |

**Recommendation:** Introduce a single API client or env (e.g. `import.meta.env.VITE_API_URL`) and use it everywhere. Remove or clearly deprecate `Create_new.jsx`. Either implement `UserContext` and wire MintGame/Remix into the app or remove them. Fix `main.jsx`: define `root` and remove redundant imports.

---

### 1.4 Configuration and deployment

| Issue | Location | Description |
|-------|----------|-------------|
| **No environment-based config** | Backend / Frontend | Backend port and host are hardcoded; frontend has no build-time env for API URL. No distinction for dev/staging/prod. |
| **CORS wide open** | `backend/backend.py` | `CORS(app)` with no origins list. Acceptable for local dev but risky if backend is ever exposed. |
| **Debug mode in code** | `backend/backend.py` | `app.run(debug=True, ...)` is hardcoded. Should be driven by env (e.g. `FLASK_DEBUG`). |
| **No CI/CD** | Repository root | No `.github/workflows` or other CI. No automated tests, lint, or build on push/PR. |

**Recommendation:** Use env vars for Flask host, port, debug, and CORS origins. Use Vite env for `VITE_API_URL`. Add a minimal CI workflow (e.g. install, lint, run tests if added).

---

## 2. Test Debt

### 2.1 Missing test infrastructure

| Issue | Description |
|-------|-------------|
| **No backend tests** | No `tests/` directory, no `pytest`, `unittest`, or `tox`. `backend/` has zero test files. |
| **No frontend tests** | No Jest, Vitest, or React Testing Library. `package.json` has no `test` script. No `*.test.jsx` or `*.spec.jsx`. |
| **No E2E tests** | No Playwright/Cypress (or similar) for critical flows (create game, play game). `screenshot.py` uses Playwright for screenshots only. |
| **No API contract tests** | Request/response shapes for `/enhance_prompt`, `/generate_game`, `/update_game`, `/get_game`, `/list_games` are not validated by tests. |

### 2.2 Untestable design

| Issue | Description |
|-------|-------------|
| **Tightly coupled routes** | Route handlers in `backend.py` directly instantiate AI clients and do I/O. Unit-testing “enhance prompt” or “generate game” without hitting real APIs or the filesystem requires refactoring. |
| **No dependency injection** | Flask app, genai, and anthropic clients are module-level globals. Hard to swap with mocks in tests. |
| **Inline prompts** | Large prompt strings live inside route handlers. Hard to review and test prompt changes in isolation. |

**Recommendation:** Add `pytest` and a `backend/tests/` with at least: (1) tests for `sanitize_game_id` and `extract_html_from_response` (pure functions), (2) one integration test that mocks Gemini/Claude and asserts request/response shape for one POST. Add Vitest (or Jest) for frontend; test one or two key components (e.g. Create step flow with mocked fetch). Extract AI and storage behind interfaces so routes can be tested with fakes.

---

## 3. Documentation Debt

### 3.1 Incorrect or inconsistent docs

| Issue | Location | Description |
|-------|----------|-------------|
| **Wrong API path names** | `README.md` (e.g. “API Endpoints”, “Example Usage”) | README documents `POST /enhance-prompt`, `POST /generate-game`, `GET /games`, `GET /games/<id>`. Actual routes use **underscores** and different paths: `/enhance_prompt`, `/generate_game`, `/list_games`, `/get_game/<id>`. Copy-pasting README curl commands fails. |
| **Wrong run command** | `README.md` (Backend Setup step 5) | Says “Run `python backend.py`”. From repo root this is wrong (file is `backend/backend.py`). Should be `python backend.py` from `backend/` or `python backend/backend.py` from root. |
| **Tech stack inaccuracy** | `README.md` (Tech Stack) | Lists “Python-dotenv - Environment variable management” but `requirements.txt` does not include `python-dotenv`, and the backend does not load `.env`. |
| **Backend Readme path names** | `backend/Readme.md` | Lists `POST /enhance-prompt`, `GET /games/<id>`. Same mismatch with actual routes (`/enhance_prompt`, `/get_game/<id>`). |

### 3.2 Missing documentation

| Issue | Description |
|-------|-------------|
| **No API spec** | No OpenAPI/Swagger or machine-readable API definition. Only prose in README with wrong paths. |
| **No architecture overview in README** | High-level architecture is in `docs/SYSTEM_ARCHITECTURE.md` but not summarized or linked from main README. |
| **Unused modules undocumented** | No note on what `gamemaker.py`, `gameCreator.py`, `screenshot.py`, or `temp.py` are for, or that they are optional/legacy. |
| **Frontend README is generic** | `frontend/README.md` is the default Vite template text, not specific to this project (no env vars, no how to point at backend). |
| **Missing LICENSE file** | README says “see the LICENSE file”; no `LICENSE` file exists in the repo. |
| **No contribution/testing guide** | Contributing section does not mention running tests or lint; there are no tests to run. |
| **No error/limitation docs** | No doc on rate limits, typical failure modes (e.g. truncated HTML from Claude), or how to debug “incomplete game” errors. |

### 3.3 In-code documentation

| Issue | Description |
|-------|-------------|
| **Minimal docstrings** | Backend route handlers have one-line docstrings; no args/returns. No module-level docstrings for `backend.py`, `gamemaker.py`, or `gameCreator.py`. |
| **No JSDoc in frontend** | React components and helpers (e.g. `getRandomName.js`) lack JSDoc or clear comments for props and behavior. |

**Recommendation:** Align README and backend Readme with actual routes and commands. Add `python-dotenv` to requirements and document it, or remove it from the stack list. Add a short “Architecture” section in README linking to `docs/SYSTEM_ARCHITECTURE.md`. Document or remove unused backend modules. Add a `LICENSE` file. Optionally add an OpenAPI snippet for the main endpoints. Add a short “Development” section: how to run backend/frontend, set env vars, and (when added) run tests.

---

## 4. Summary Table

| Category | Severity | Count | Top priority |
|----------|----------|-------|--------------|
| **Architectural** | Critical | 4 | Remove hardcoded API keys; load from env. |
| **Architectural** | High | 10+ | Fix `main.jsx`; centralize API base URL; remove or fix orphan components. |
| **Test** | High | 6 | No tests at all; add pytest + a few backend tests and a frontend test script. |
| **Documentation** | Medium | 8+ | Fix README API paths and run command; add LICENSE; document or remove unused code. |

---

## 5. Suggested remediation order

1. **Security:** Move API keys to env; use `python-dotenv`; sanitize `.env.example`.
2. **Stability:** Fix `main.jsx` (`root` and imports).
3. **Documentation:** Correct README API paths and run instructions; add LICENSE.
4. **Maintainability:** Introduce a single API base URL (env) in frontend; extract backend services.
5. **Quality:** Add backend tests for sanitize and HTML extraction; add a minimal frontend test script.
6. **Cleanup:** Remove or document `Create_new.jsx`, `MintGame.jsx`, `Remix.jsx`; fix or remove `UserContext` dependency.

This list can be used as a backlog for incremental improvement.
