# Documentation-Related Technical Debt

Review of the AI Game Generator repository with focus on: AI-generated or hard-to-understand code, missing explanations of intent or system behavior, and lack of documentation for AI prompts, assumptions, or boundaries.

---

## 1. AI Prompts Undocumented (Intent, Assumptions, Boundaries)

### 1.1 Main Flask game-generation prompt (`backend/backend.py`, ~lines 141–225)

A single ~85-line string is sent to Claude to generate PhaserJS game HTML. There is no separate doc or comment that explains:

- **Intent:** Why this prompt is structured this way; what problem the “PHASER.JS FUNCTION VALIDATION” and “fillStar/fillHexagon” constraints are solving (e.g. past model hallucinations).
- **Assumptions:** That the model returns raw HTML or markdown-wrapped HTML; that 20k max tokens is sufficient; that the model can reliably produce a single, runnable document.
- **Boundaries:** What kinds of games or prompts are in/out of scope; that complex or very long games may be truncated with no documented handling; what “production-ready” means in practice.
- **Evolution:** No record of why certain lines exist (e.g. “this.add.graphics(...).fillPolygon is not a function”) or how the prompt has changed over time.

The same applies to the **update_game** prompt (~lines 288–304): no doc on intended use of “feedback” (e.g. natural language only, no code), size limits for `current_html`, or how truncation is handled.

### 1.2 Enhance-prompt flow (`backend/backend.py`, ~lines 77–90)

The Gemini “game design assistant” prompt is short but has no documented:

- **Intent:** Why the response is treated as a “description” and then merged with fixed fields (title, genre, game_mechanics, etc.) rather than structured JSON.
- **Assumptions:** That Gemini returns plain text suitable for that merge; that hardcoded fallbacks (“Action”, “Arrow keys or WASD”) are acceptable for all enhanced concepts.
- **Boundaries:** No stated limits on prompt length, language, or content that might produce poor enhancement.

### 1.3 Standalone scripts (`gamemaker.py`, `gameCreator.py`)

Multiple prompts exist (e.g. `create_json`, `generate_game_code`, `update_game`, `plan_generator`, `sprite_generator`, `game_logic_generator`) with:

- No module- or file-level explanation of each script’s role (alternative pipeline, legacy, or experimental).
- No doc for individual prompts: intent, expected input/output shape, or how they differ from the Flask app’s prompts (e.g. different models, different Phaser constraints).
- Typo in `gameCreator` (“phasor.Scene”, “sprited”) with no comment that this is known or intentional for the model.

---

## 2. Missing Explanations of System Behavior

### 2.1 Enhance response shape vs. usage

`enhance_prompt` returns an object with `title`, `description`, `genre`, `game_mechanics`, `visual_style`, `controls`, `objectives`. The backend **always** overwrites the model’s narrative with fixed defaults for everything except `description`. There is no doc explaining:

- That the API returns a structured object but only `description` is taken from the model; the rest are placeholders.
- Why the frontend can show “enhanced concept” and still have genre/controls etc. as generic values, or how that affects downstream generation.

### 2.2 Dual input contract for `generate_game`

The backend accepts `enhanced_prompt` as either a string or an object and branches (lines 124–140). There is no doc stating:

- That both forms are supported by design.
- What the frontend actually sends (object with optional fields) and what defaults are applied when fields are missing.
- How a string-only payload affects metadata (title, genre) when the game is saved.

### 2.3 HTML extraction and truncation

`extract_html_from_response()` strips markdown code fences and logs warnings for missing `</html>` or trailing `...`. There is no doc explaining:

- That Claude often wraps HTML in markdown and sometimes truncates.
- What the system does when output is truncated (e.g. the HTML is still saved and may be broken); no user-facing or operator-facing guidance.
- Why the regex sequence is ordered as it is or whether there are known failure modes (e.g. nested code blocks).

### 2.4 Game listing and “featured” vs. saved games

- **Backend:** `list_games` returns games from the `games/` directory only; there is no doc that this is the single source of persisted games.
- **Frontend:** HomePage uses a hardcoded `featuredGames` list (IDs 1–6) and never calls `list_games`. Game.jsx tries `get_game/:id` then falls back to the same hardcoded set. There is no doc explaining:
  - That the homepage “Featured AI Games” are not from the backend.
  - That numeric IDs (1–6) are for demo content and UUIDs are for real saved games, or how routing/links relate to the two sources.

### 2.5 Optional `game_id` in update_game

When `game_id` is missing or invalid, the route still returns updated HTML but does not persist it. There is no doc that this “preview-only” behavior is intentional or how the frontend should use it (e.g. always send `game_id` when updating an existing game).

---

## 3. AI-Generated or Hard-to-Understand Code Without Explanation

### 3.1 Large inline prompt strings

The generate_game and update_game prompts in `backend.py` are long, multi-section strings in the middle of route handlers. A human reading the file must infer structure (GAME CONCEPT, CRITICAL REQUIREMENTS, TECHNICAL SPECIFICATIONS, etc.) and cannot tell at a glance which parts have caused production issues or been tuned. There is no pointer to a separate prompt doc or versioned prompt file.

### 3.2 gameCreator’s embedded prompt template (`sprite_generator`)

The BootScene prompt (~lines 59–134) contains a long JavaScript snippet as an “example format” with escaped braces. The intent (e.g. “use this exact preload/generateTextures pattern”) and whether the model is expected to follow it literally or only as reference are not documented. The typo “phasor” and “sprited” appear in the prompt with no note.

### 3.3 Validation heuristics with no rationale

- **Frontend `validateGameHtml()` (Create.jsx):** Checks for `</html>`, “phaser”/“Phaser”, and rejects content ending with `...`. There is no doc on why these heuristics were chosen or their false positive/negative behavior (e.g. minified code, different casing).
- **Backend `gameCreator.validate_generated_code()`:** Rejects code containing the substring `"//"` or `"/* */"` as “potential incomplete code.” That flags any comment in generated JavaScript. No doc explains the intent or that this may be overly strict.

### 3.4 Regex-based HTML extraction in two places

`backend.py` uses `extract_html_from_response()` with a specific regex sequence. `gamemaker.py` uses a different approach (multiple `re.search` patterns for `<!DOCTYPE html>...`, `<html>...`, etc.). Neither file documents:

- Why that extraction strategy was chosen.
- What response formats (e.g. markdown with multiple code blocks) are supported or unsupported.
- That the two implementations can behave differently for the same model output.

---

## 4. Missing or Misleading Public Documentation

### 4.1 README API section does not match implementation

- Paths: README documents `POST /enhance-prompt`, `POST /generate-game`, `POST /update-game`, `GET /games`, `GET /games/<game_id>`. Implemented routes use underscores and different names: `/enhance_prompt`, `/generate_game`, `/update_game`, `/list_games`, `/get_game/<id>`.
- Request/response shapes: README shows `enhanced_prompt` as the enhance response; the API actually returns an object with `title`, `description`, `genre`, etc. README shows `prompt` and `title` for generate; the API expects `enhanced_prompt` (string or object).
- No mention of optional `game_id` on update, or of the actual fields returned by list_games and get_game (e.g. `metadata`, `html`).

So the only “documentation” of API behavior is wrong; intent and boundaries are not described accurately.

### 4.2 No documentation of AI pipeline as a whole

There is no single place that describes:

- The main pipeline: user prompt → enhance (Gemini) → generate (Claude) → persist (filesystem).
- Model choices (e.g. gemini-2.0-flash, claude-sonnet-4-20250514) and why.
- Token limits, temperature, or retry/error handling.
- That gamemaker.py and gameCreator.py are separate pipelines with different models and prompts, and how (or whether) they relate to the Flask app.

### 4.3 No documentation of failure modes or limits

Missing entirely:

- That generation can be truncated (e.g. long games) and how to recognize or handle it.
- Rate limits or recommended usage for Gemini/Claude.
- Maximum practical size for `current_html` or feedback text in update_game.
- What “incomplete game” or “truncation” errors mean in the UI and what the user can do.

### 4.4 Unused or alternative modules undocumented

`screenshot.py`, `gamemaker.py`, `gameCreator.py`, `temp.py` have no README or top-of-file doc explaining their role (e.g. screenshot for thumbnails, gamemaker/gameCreator as alternate or legacy pipelines). A maintainer cannot tell whether they are part of the supported product or internal tools.

---

## 5. In-Code Documentation Gaps

### 5.1 Backend

- **backend.py:** No module-level docstring. Route handlers have one-line docstrings only; no description of request/response contracts, error codes, or side effects. Helper functions (`sanitize_game_id`, `extract_html_from_response`) have minimal docstrings; the latter does not describe input/output format or truncation behavior.
- **gamemaker.py / gameCreator.py:** No module docstrings. Functions like `create_json`, `plan_generator`, `sprite_generator`, `game_logic_generator` lack docstrings describing purpose, arguments, return shape, or how they plug into the larger flow.

### 5.2 Frontend

- **Create.jsx:** No file or component-level doc. Step semantics (e.g. what “step 2” means, when to show edit vs. generate) are only inferable from code. `validateGameHtml` and `parseMarkdown` have no JSDoc or comment on intended use or limits.
- **Game.jsx:** Dual source of truth (backend vs. hardcoded sample data) and fallback logic are not commented. No doc on when the user sees a “real” game vs. a demo.
- **getRandomName.js:** Single export with no description of purpose (e.g. for Navbar display name) or that it depends on `@afuggini/namegenerator`.

### 5.3 Screenshot utility

`screenshot_game()` takes `html_content` and uses a fixed `blob_id = 'abcd'` and fixed path. No doc on whether the function is used by the app, from where it is called, or what the signature is intended to be (e.g. should `blob_id` be a parameter).

---

## Summary: Most Significant Documentation Debt Items

| # | Category | Issue |
|---|----------|--------|
| 1 | **AI prompts** | No documented intent, assumptions, or boundaries for the main generate_game and update_game prompts; no explanation of Phaser constraints or truncation handling. |
| 2 | **AI prompts** | Enhance-prompt flow: no doc that only `description` is from the model and the rest are hardcoded; no documented boundaries for the enhance step. |
| 3 | **System behavior** | Dual contract for `enhanced_prompt` (string vs object) and optional `game_id` on update are undocumented; response shapes and defaults are not specified. |
| 4 | **System behavior** | No explanation that HomePage “featured games” are hardcoded and separate from backend `list_games`; no single documented source of truth for “what games exist.” |
| 5 | **AI/output handling** | HTML extraction and truncation behavior (backend and frontend validation) are undocumented; no guidance for operators or users when output is incomplete. |
| 6 | **Public docs** | README API section wrong (paths and request/response shapes); no doc of full AI pipeline, model choices, or failure modes/limits. |
| 7 | **Code clarity** | Large inline prompts and validation heuristics (e.g. `"//"` in validate_generated_code, validateGameHtml rules) have no rationale or intent documented. |
| 8 | **Module boundaries** | gamemaker.py, gameCreator.py, screenshot.py, temp.py have no documented role or relationship to the main Flask app. |

These items make it difficult for humans to understand why the system behaves as it does, what the AI prompts are trying to achieve, and what the boundaries and failure modes are—and they increase the risk of incorrect changes when evolving prompts or behavior.
