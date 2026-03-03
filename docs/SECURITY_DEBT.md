# Security-Related Technical Debt

Analysis limited to **technical debt present in the repository**: hardcoded secrets, unsafe execution patterns, and missing validation or trust boundaries. Not hypothetical attack scenarios.

---

## 1. Hardcoded Secrets

### 1.1 API keys in backend source

**Location:** `backend/backend.py` (lines 16–17)

Gemini and Claude API keys are assigned as string literals. The comment states "NEVER hardcode API keys" but the code does not use environment variables or a config layer. Keys are committed with the application and cannot be rotated without a code change.

### 1.2 Real credentials in .env.example

**Location:** `backend/.env.example` (lines 4, 7)

The example environment file contains the same real API key values as in `backend.py`. Anyone copying the file (e.g. for local setup) is instructed by README to "add your API keys" but the template already contains production keys. This spreads secrets and makes it easy to commit them into new repos or share them by mistake.

### 1.3 API keys in standalone scripts

**Location:** `backend/gamemaker.py` (line 15), `backend/gameCreator.py` (lines 4, 27)

Gemini API keys are hardcoded inside these modules. They are separate from the Flask app but still in the same repo and share the same secret-management gap (no env-based config).

---

## 2. Unsafe Execution Patterns

### 2.1 Rendering API output as HTML without sanitization

**Location:** `frontend/src/Create.jsx` (line 531)

`enhancedPrompt.description` (from the Gemini enhance_prompt API) is passed through `parseMarkdown()` and then rendered with `dangerouslySetInnerHTML`. `parseMarkdown` only applies regex-based markdown substitution (bold, italic, headings, lists, line breaks) and does not strip or escape HTML (e.g. `<script>`, `<img onerror=...>`, event handlers). Any HTML or script in the model response is rendered as-is in the main document. The trust boundary (Gemini output is treated as safe for HTML context) is not enforced by sanitization.

### 2.2 Game iframe with allow-scripts and allow-same-origin

**Locations:** `frontend/src/Create.jsx` (line 665), `frontend/src/Game.jsx` (line 297), `frontend/src/Remix.jsx` (line 244)

AI-generated game HTML (from Claude) is executed in an iframe with `sandbox="allow-scripts allow-same-origin"` (Game.jsx also adds `allow-modals`). The combination of `allow-scripts` and `allow-same-origin` allows the iframe content to be treated as same-origin with the parent in some cases, which can weaken sandbox isolation. There is no documentation or code comment describing this as an intentional trust boundary or accepted risk.

### 2.3 Game preview iframe with no sandbox

**Location:** `frontend/src/Create_new.jsx` (lines 369–374)

The iframe that displays `gameHtml` (AI-generated HTML/JS) has no `sandbox` attribute. Unsandboxed iframe content runs with full default permissions (script execution, same-origin, form submission, etc.). This is a stricter unsafe-execution case than the sandboxed iframes elsewhere.

---

## 3. Missing Validation or Trust Boundaries

### 3.1 No size or type limits on request payloads

**Location:** `backend/backend.py` (enhance_prompt, generate_game, update_game)

- **enhance_prompt:** Accepts `data['prompt']` with no maximum length or type check. Very large prompts are forwarded to Gemini and can affect cost, latency, and failure modes.
- **generate_game:** Accepts `data['enhanced_prompt']` as string or object with no size limit. The full object (including potentially large `description` text) is sent to Claude and can contribute to truncation or abuse.
- **update_game:** Accepts `feedback` and `current_html` with no length or size limits. Large `current_html` is sent to Claude and written to disk when `game_id` is present; there is no guard against oversized payloads.

The backend does not define trust boundaries (e.g. max body size, max field length) for these inputs.

### 3.2 File path derived from untrusted config in gameCreator

**Location:** `backend/gameCreator.py` (line 268)

The output filename is built as `f"{config['title'].replace(' ', '_').lower()}.html"`. `config` comes from `create_json(prompt)` (Gemini), so `title` is ultimately user- and model-controlled. Characters such as `../`, `/`, or `\` are not stripped or forbidden. A title like `../../../tmp/pwned` would cause the file to be written outside the current directory. There is no validation that the resulting path stays within an allowed directory.

### 3.3 CORS allows any origin

**Location:** `backend/backend.py` (line 13)

`CORS(app)` is called with no arguments, so Flask-CORS uses defaults that allow any origin. The repo contains a `.env.example` with `CORS_ORIGINS=...`, but the application never reads or applies it. If the backend is exposed beyond localhost, any site can make authenticated (same-credential) requests to the API. There is no enforced trust boundary for which origins may call the backend.

### 3.4 update_game accepts arbitrary game_id from body without audit

**Location:** `backend/backend.py` (update_game, lines 286, 318–328)

`game_id` is taken from the request body (`data.get("game_id")`). If present and valid per `sanitize_game_id`, the backend overwrites that game’s `index.html` with the new content. There is no check that the client is allowed to modify that game (e.g. no authentication or ownership). Any client that knows a valid UUID can overwrite any saved game. The trust boundary (who may update which game) is not implemented.

### 3.5 No rate limiting or abuse controls

**Location:** `backend/backend.py` (all POST routes)

The API does not implement rate limiting, per-IP caps, or request throttling. `.env.example` mentions `RATE_LIMIT_PER_MINUTE=10` but the application does not use it. High-volume or automated calls to enhance_prompt, generate_game, or update_game can be made without restriction, affecting cost and availability. There is no documented or enforced boundary for acceptable use.

---

## Summary Table

| # | Category | Issue |
|---|----------|--------|
| 1 | Hardcoded secrets | API keys in `backend.py`; no use of environment or config. |
| 2 | Hardcoded secrets | Real API keys in `.env.example` instead of placeholders. |
| 3 | Hardcoded secrets | API keys in `gamemaker.py` and `gameCreator.py`. |
| 4 | Unsafe execution | `dangerouslySetInnerHTML` with Gemini output; no HTML sanitization. |
| 5 | Unsafe execution | Game iframes use `allow-scripts` + `allow-same-origin`; no documented trust boundary. |
| 6 | Unsafe execution | `Create_new.jsx` game iframe has no `sandbox` attribute. |
| 7 | Missing validation | No size/type limits on prompt, enhanced_prompt, feedback, or current_html. |
| 8 | Missing validation | `gameCreator.py` builds file path from `config['title']` with no path traversal checks. |
| 9 | Trust boundary | CORS allows any origin; CORS_ORIGINS in .env is unused. |
| 10 | Trust boundary | update_game does not enforce who may update which game. |
| 11 | Trust boundary | No rate limiting; RATE_LIMIT_PER_MINUTE in .env is unused. |

All items above are current state of the codebase and represent security-related technical debt, not hypothetical attacks.
