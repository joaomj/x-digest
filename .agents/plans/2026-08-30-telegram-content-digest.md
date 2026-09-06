# Plan: Weekly Telegram content digest via OpenRouter (v0)

## Goal
Each weekly `sync` (Sunday 06:00 `launchd`) sends you one private Telegram message that summarizes what the new bookmarks say. The digest groups the week's posts by theme, gives key points per theme, and links each point to its source post `https://x.com/.../status/<id>`. You read the week in Telegram without opening the vault.

## Deliverable
Approved repository plan at `x-digest/.agents/plans/2026-08-30-telegram-content-digest.md` that authorizes `software-delivery` to add the digest with OpenRouter as LLM for v0. No code is changed by this plan file.

## Work stages
1. Inspect `sync` pipeline, `Settings`, `Database.posts`, and logging to find the seam.
2. Define OpenRouter plus Telegram settings and secret handling.
3. Define digest content, prompt, token budget, and Telegram chunking.
4. Build one ordered chain of numbered steps with mandatory gates so a junior engineer can deliver without new design.

## Reason for chosen path
Reuse `Pipeline.sync` in `x-digest/src/x_digest/pipeline.py:244`, `Settings` in `x-digest/src/x_digest/config.py:31`, `Database` in `x-digest/src/x_digest/db.py:144`, and `JsonlLogger` in `x-digest/src/x_digest/logging_setup.py:25`. Send the digest as the last stage after `MarkdownWriter` in `x-digest/src/x_digest/pipeline.py:105` and before `write_run_manifest`. Call OpenRouter `POST https://openrouter.ai/api/v1/chat/completions` and Telegram `POST https://api.telegram.org/bot<token>/sendMessage` with `requests` already used in `x-digest/src/x_digest/x_api.py:1`. This adds no new Python dependency, keeps the `scripts/install-scheduler.sh:35` agent unchanged, and lets a future local server reuse the same OpenAI-compatible client by switching `llm_base_url`.

## Decisions made for v0
- LLM provider is OpenRouter. Local inference server (Mac mini or Contabo VPS) is v1 and will reuse the same client by changing `XDIGEST_LLM_BASE_URL`.
- Default model is `openai/gpt-oss-120b` on OpenRouter, routed to the cheapest ZDR-compliant provider. Request sets `provider: {sort: "price", zdr: true, data_collection: "deny"}` and top-level `zdr: true` so OpenRouter only routes to Zero Data Retention endpoints and the cheapest qualifying provider wins (equivalent to `:floor` suffix, verified via `GET https://openrouter.ai/api/v1/endpoints/zdr` where `openai/gpt-oss-120b` via AkashML and CoreWeave at $0.03/M input and $0.17/M output is the cheapest ZDR endpoint as of 2026-08-30).
- Digest style is themed summary with key points per theme, each point cites its post URL. Overall 2-sentence takeaway at top.
- Telegram overflow is chunked into 2 to 3 messages at 4000 characters with HTML parse mode. No truncation without link in v0.
- Failure policy is retry then best-effort. LLM and Telegram each retry 3 times on `429`, `500`, `502`, `503`, `504` with exponential backoff 1s, 2s, 4s. After retries, log `llm_failed` or `telegram_failed` in `run_events`, keep `runs.status` as `success` so the archive is safe. Next weekly run retries.
- Empty week still sends one short message "No new bookmarks since <last_run_date>" so you know the job ran.
- Credential storage is `.env` `XDIGEST_` variables primary, with optional Keychain fallback (`x-digest` service, accounts `telegram-bot-token`, `telegram-chat-id`, `openrouter-api-key`) same pattern as `x-digest/src/x_digest/auth.py:19` and `x-digest/scripts/backup-to-drive.sh:33`.

## Context and evidence
- Weekly sync is `Pipeline.sync` in `x-digest/src/x_digest/pipeline.py:244` called by `launchd` Sunday 06:00 via `x-digest/scripts/install-scheduler.sh:43`.
- `Pipeline.sync` writes Bronze via `x-digest/src/x_digest/bronze.py:59`, Silver via `x-digest/src/x_digest/silver.py:81`, media via `x-digest/src/x_digest/media.py`, Markdown via `x-digest/src/x_digest/markdown.py:15`, then manifest, usage, and `runs` status `x-digest/src/x_digest/pipeline.py:327`. No notification exists.
- Normalized content lives in `x-digest/src/x_digest/db.py:62` `posts` columns `text`, `note_text`, `article_body`, `article_json`, `username`, `url`, `created_at`, `first_seen_at`, `last_seen_at`. Search uses `x-digest/src/x_digest/gold.py:39` FTS5.
- Settings load via `pydantic-settings` with `XDIGEST_` prefix in `x-digest/src/x_digest/config.py:31`. Current `.env.example` has 7 keys and no Telegram or LLM keys.
- Logging is `JsonlLogger` in `x-digest/src/x_digest/logging_setup.py:25` with per-run files `data/logs/runs/<run_id>.jsonl`.
- `x-digest/pyproject.toml:8` dependencies are `keyring`, `pydantic-settings`, `xdk`. `requests` is transitive via `xdk`. No LLM SDK exists.
- Telegram Bot API is `POST https://api.telegram.org/bot<token>/sendMessage` with `chat_id`, `text`, `parse_mode=HTML`. Limit is 4096 characters per message.
- OpenRouter API is OpenAI-compatible `POST https://openrouter.ai/api/v1/chat/completions` with `Authorization: Bearer <key>`, `model`, `messages`, `max_tokens`, plus `provider: {sort: "price"}` to route to cheapest provider and `zdr: true` plus `provider: {zdr: true}` to enforce Zero Data Retention. Model `openai/gpt-oss-120b` has ZDR endpoints at `$0.03` input and `$0.17` output per million tokens via AkashML and CoreWeave as cheapest verified via `GET https://openrouter.ai/api/v1/endpoints/zdr` on 2026-08-30, with `:floor` suffix as equivalent alias for cheapest sort.

## Acceptance criteria
- When `XDIGEST_TELEGRAM_BOT_TOKEN` or `XDIGEST_TELEGRAM_CHAT_ID` or `XDIGEST_LLM_API_KEY` is missing, `sync` completes without error, logs `digest_skipped` with reason `missing_config`, and `runs.status` is `success`.
- When configured, weekly `sync` sends a themed Telegram digest that contains an overall takeaway, 2 to 5 themes, key points per theme, and source post URLs. Digest covers posts where `first_seen_at` is after the prior successful `runs.completed_at` (or all posts on first run), capped at `digest_max_posts` 20 and 800 characters per post body in the prompt, total prompt under 12000 characters.
- Telegram message uses HTML with escaped text, `disable_web_page_preview=false`, and is chunked at 4000 characters. No token appears in `run_events.details_json` or `data/logs/runs/<run_id>.jsonl`.
- `uv run x-digest digest --dry-run` prints the rendered digest to stdout without calling OpenRouter or Telegram. `uv run x-digest digest --send` sends to Telegram.
- LLM and Telegram calls use timeout 30s and 10s respectively, retry 429/5xx with backoff, and never raise to fail the run when policy is best-effort.
- `uv run pytest -q` and `uv run ruff check src tests` pass. Existing `verify --full` and `rebuild_silver` are unchanged.

## Dependencies
- Telegram bot created via `@BotFather` with token. Chat ID obtained via `curl https://api.telegram.org/bot<token>/getUpdates` after sending the bot one message.
- OpenRouter account with API key, credits, and chosen model enabled. Key stored in `.env` `XDIGEST_LLM_API_KEY` or Keychain `openrouter-api-key`.

## Risks
- Token leaked in logs or Git. Mitigation is never log `token`, `api_key`, or `chat_id` values, `data/` and `.env` are ignored in `x-digest/.gitignore:1`, Keychain is preferred for local.
- Prompt exceeds model context on large weeks. Mitigation is cap posts and truncate bodies, log `digest_truncated` when capped.
- LLM hallucination invents themes or links. Mitigation is prompt requires citation of `post_id` and `url` for each point, and builder verifies each cited `post_id` exists in the queried set.
- Telegram 429 rate limit. Mitigation is retry with backoff and chunking.
- OpenRouter downtime at 06:00. Mitigation is best-effort retry, log, keep archive, next week retries.

## Out of scope
- Local LLM server, Contabo VPS hosting, TLS proxy, or `OLLAMA` setup. The client is compatible but v0 ships only OpenRouter.
- Sending images, files, or Markdown files to Telegram.
- Interactive bot commands, editing old messages, or multiple chats.
- Changing Bronze/Silver schema, pipeline bookmark pagination, or `silver.sqlite` snapshot method.
- Migrating old Drive backup history or changing backup schedule.
- Committing `data/`, `silver.sqlite`, or any key to Git.

## Selected pattern and trade-off
One `LlmClient` that speaks OpenAI-compatible `chat/completions` plus one `DigestBuilder` plus one `TelegramSender`, all called from `Pipeline` as a final stage with `requests`. This reuses `requests`, keeps one HTTP pattern, adds no new dependency, and lets v1 switch to a local server by changing `llm_base_url`. Trade-off is no streaming, no vendor SDK helpers, and prompt truncation instead of map-reduce for very large weeks.

## Alternative considered
Add `openai` SDK plus `python-telegram-bot` library. Rejected because v0 needs only two `POST` calls, the SDKs add async and dependency weight, and OpenRouter already speaks the OpenAI wire format.

---

## Repository: x-digest

### Step 1: Extend Settings with Telegram and OpenRouter configuration
- Repository: `x-digest`
- Complete paths: `x-digest/src/x_digest/config.py:31` :: `class Settings`, `x-digest/.env.example:1` :: env template
- Change: In `x-digest/src/x_digest/config.py:31` add fields `telegram_bot_token: str | None = None`, `telegram_chat_id: str | None = None`, `telegram_timeout_seconds: float = Field(default=10.0, gt=0, le=60)`, `telegram_parse_mode: str = "HTML"`, `llm_provider: str = Field(default="openrouter")`, `llm_base_url: str = "https://openrouter.ai/api/v1"`, `llm_api_key: str | None = None`, `llm_model: str = "openai/gpt-oss-120b"`, `llm_max_tokens: int = Field(default=1200, ge=100, le=4000)`, `llm_timeout_seconds: float = Field(default=30.0, gt=0, le=120)`, `digest_max_posts: int = Field(default=20, ge=1, le=50)`, `digest_max_chars_per_post: int = Field(default=800, ge=100, le=4000)`. Routing and ZDR are enforced in `x-digest/src/x_digest/llm.py:10` via `provider` object, not by model suffix. Add validator that treats empty string as `None` for tokens and keys. Add helper `telegram_enabled() -> bool` that returns true only when bot token and chat id are non-empty. Add helper `llm_enabled() -> bool` that returns true only when `llm_api_key` is non-empty. Do not add new file. In `x-digest/.env.example:1` keep existing 7 keys and append documented block for `XDIGEST_TELEGRAM_BOT_TOKEN`, `XDIGEST_TELEGRAM_CHAT_ID`, `XDIGEST_LLM_API_KEY`, `XDIGEST_LLM_MODEL=openai/gpt-oss-120b`, `XDIGEST_LLM_BASE_URL=https://openrouter.ai/api/v1` with comments that Keychain alternative is `x-digest` service accounts `telegram-bot-token`, `telegram-chat-id`, `openrouter-api-key` and that routing to cheapest ZDR provider is handled in code.
- Acceptance criteria: `load_settings()` reads `XDIGEST_TELEGRAM_BOT_TOKEN=dummy` and `XDIGEST_LLM_MODEL=openai/gpt-oss-120b` from env, empty strings are `None`, `telegram_enabled()` and `llm_enabled()` behave as specified, and `.env.example` contains the new keys.
- Dependency: None. This is the first gate.
- Pattern: Extend `pydantic-settings` `Settings` with `XDIGEST_` prefix. Fits existing `config.py:31` pattern. Trade-off is env plus Keychain dual lookup adds one helper.
- Alternative: Create `telegram.ini` file. Rejected because project standard is `XDIGEST_` env plus Keychain via `src/x_digest/auth.py:19`.
- Out of scope: Creating `llm.py`, `digest.py`, `telegram.py`, or touching `pipeline.py`.
- Risk and rollback: Wrong field type breaks `load_settings()`. Rollback is `git checkout -- src/x_digest/config.py .env.example`.
- Gate command or check: `uv run ruff check src/x_digest/config.py && uv run python -c "from x_digest.config import load_settings; s=load_settings(); print(s.telegram_enabled(), s.llm_enabled(), s.llm_base_url)" && grep -F XDIGEST_TELEGRAM_BOT_TOKEN .env.example && grep -F XDIGEST_LLM_API_KEY .env.example`
- Gate pass condition: `ruff` exits 0, Python prints `False False https://openrouter.ai/api/v1`, and both greps return one match.
- Stop condition: `ruff` fails, Python raises `ValidationError`, or greps fail. Fix fields before next step.

### Step 2: Add OpenRouter-compatible LlmClient with retries and redaction
- Repository: `x-digest`
- Complete paths: `x-digest/src/x_digest/llm.py:1` :: new file `class LlmClient`
- Change: Create `x-digest/src/x_digest/llm.py` with `class LlmClient` that takes `Settings`, `JsonlLogger | None`, `correlation_id`. Resolve `api_key` from `settings.llm_api_key` then Keychain `x-digest:openrouter-api-key` via `keyring.get_password` same as `src/x_digest/auth.py:19`. If no key, raise `ValueError` only when `complete()` is called, not at init. Method `complete(prompt: str, system: str) -> str` does `POST {base_url}/chat/completions` with headers `Authorization: Bearer <key>`, `Content-Type: application/json`, `HTTP-Referer: https://github.com/x-digest` and `X-Title: x-digest`, body `{"model": settings.llm_model, "messages": [{"role":"system","content":system},{"role":"user","content":prompt}], "max_tokens": settings.llm_max_tokens, "temperature": 0.3, "zdr": true, "provider": {"sort": "price", "zdr": true, "data_collection": "deny"}}`. The `provider.sort: "price"` routes to the cheapest provider for `openai/gpt-oss-120b` (equivalent to `:floor` suffix per OpenRouter docs), and `zdr: true` plus `provider.zdr: true` enforces Zero Data Retention per `https://openrouter.ai/docs/guides/features/zdr`. Timeout `settings.llm_timeout_seconds`, retry 3 times on `429`, `500`, `502`, `503`, `504` or `requests.RequestException` with backoff 1s, 2s, 4s. On success return `choices[0].message.content` stripped. On failure raise `RuntimeError` with status and body truncated to 500 chars, never include key. Log `llm_attempt` and `llm_failed` via `JsonlLogger` without secrets. Validate ZDR routing in tests by asserting request JSON contains `provider.sort == "price"` and `zdr == true`.
- Acceptance criteria: `LlmClient` calls OpenRouter with correct headers and body, retries on 429/5xx, times out at configured seconds, never logs the key, and returns text.
- Dependency: Step 1 gate passed.
- Pattern: Single `requests` client with explicit retry loop, same as `src/x_digest/x_api.py:176` `_retry`. Fits no new dependency. Trade-off is manual header management.
- Alternative: Use `openai` Python SDK. Rejected because it adds a dep for one endpoint and hides header control needed for `HTTP-Referer`.
- Out of scope: Digest prompt text, Telegram code, pipeline wiring, CLI.
- Risk and rollback: Key logged or wrong base_url path doubles `/v1`. Mitigation is unit test with `responses` mock. Rollback is `rm src/x_digest/llm.py`.
- Gate command or check: `uv run ruff check src/x_digest/llm.py && uv run pytest -q -k llm`
- Gate pass condition: `ruff` exits 0, tests show at least one `test_llm_rate_limit_retries_once` and one `test_llm_never_logs_key` passing.
- Stop condition: `ruff` fails or any `llm` test fails. Fix client before building the prompt.

### Step 3: Add DigestBuilder that queries new posts and builds the themed prompt
- Repository: `x-digest`
- Complete paths: `x-digest/src/x_digest/digest.py:1` :: new file `class DigestBuilder`, `x-digest/src/x_digest/db.py:62` :: `posts` table, `x-digest/src/x_digest/db.py:19` :: `runs` table
- Change: Create `x-digest/src/x_digest/digest.py` with `class DigestBuilder` that takes `Database`, `Settings`. Method `collect_new_posts(run_id: str) -> list[dict]` finds prior successful `runs.completed_at` via `SELECT completed_at FROM runs WHERE status='success' AND completed_at IS NOT NULL ORDER BY completed_at DESC LIMIT 1`. If none, use all posts. Else query `SELECT post_id, username, url, text, note_text, article_body, created_at, first_seen_at FROM posts WHERE first_seen_at > ? ORDER BY created_at DESC LIMIT ?` with param `prior_completed_at` and `settings.digest_max_posts`. For each row build `display_text = (article_body or note_text or text).strip()[:settings.digest_max_chars_per_post]`. Return rows in ascending `created_at`. Method `build_prompt(posts: list[dict]) -> tuple[str, str]` returns system prompt that requires HTML-safe output, no markdown, grouped by 2 to 5 themes, each theme has 2 to 4 key points, each point ends with `(<username> <url>)`, overall 2-sentence takeaway first, then `Themes:` section, then `Sources:` line, and must cite only given `post_id`s. User prompt lists `Last run: <prior>` then `Posts (N):` with `1. [@username] post_id url created_at — body`. If `posts` empty, return special system "empty week" and user "No new posts". Method `summarize(posts, llm: LlmClient, run_id: str) -> dict` calls `llm.complete` when posts non-empty, otherwise returns canned "No new bookmarks since <prior>" text. Validates every cited `post_id` in LLM output exists in input set, logs `digest_truncated` if capped, returns `{"text": str, "post_ids": list[str], "prompt_chars": int}`.
- Acceptance criteria: `collect_new_posts` returns only posts since last success, capped at 20, prompt fits under 12000 characters, LLM output is validated for known post_ids, empty week returns canned text without calling LLM.
- Dependency: Step 2 gate passed.
- Pattern: Query `posts.first_seen_at` against `runs.completed_at` same pattern as incremental sync `src/x_digest/pipeline.py:282`. Fits existing checkpoint logic. Trade-off is `first_seen_at` not `created_at`, so edited reposts are not re-summarized.
- Alternative: Query by `post_observations.observed_at`. Rejected because `posts.first_seen_at` already records first archive time and is simpler.
- Out of scope: Telegram sending, pipeline event wiring, CLI.
- Risk and rollback: Prompt too long exceeds model context. Mitigation is cap plus truncation and test asserts `len(prompt) < 12000`. Rollback is `rm src/x_digest/digest.py`.
- Gate command or check: `uv run ruff check src/x_digest/digest.py && uv run pytest -q -k digest`
- Gate pass condition: `ruff` exits 0, tests show `test_collect_new_posts_since_last_success` and `test_build_prompt_cites_only_input_ids` passing.
- Stop condition: Any `digest` test fails or `ruff` fails. Fix builder before wiring Telegram.

### Step 4: Add TelegramSender with HTML escape, chunking, and retries
- Repository: `x-digest`
- Complete paths: `x-digest/src/x_digest/telegram.py:1` :: new file `class TelegramSender`
- Change: Create `x-digest/src/x_digest/telegram.py` with `class TelegramSender` that takes `Settings`, `JsonlLogger | None`, `correlation_id`. Resolve `bot_token` from `settings.telegram_bot_token` then Keychain `x-digest:telegram-bot-token`, and `chat_id` from `settings.telegram_chat_id` then Keychain `x-digest:telegram-chat-id`. Method `send(text: str) -> dict` escapes `&`, `<`, `>` for HTML, keeps `<a href="url">` links for post URLs by escaping then reinserting anchors. Splits `text` into chunks of 4000 characters without breaking a line if possible, preserving `parse_mode=HTML`. For each chunk `POST https://api.telegram.org/bot{token}/sendMessage` with `{"chat_id": chat_id, "text": chunk, "parse_mode": "HTML", "disable_web_page_preview": False}`. Timeout `settings.telegram_timeout_seconds`, retry 3 times on `429`, `500`, `502`, `503`, `504` or `RequestException` with backoff 1s, 2s, 4s. Log `telegram_chunk_sent` per chunk and `telegram_failed` on final failure, never log token or full text beyond 200 chars preview. Return `{"chunks": int, "chat_id": str}`.
- Acceptance criteria: Long digest is chunked at 4000, HTML is valid, 429 retries, token never appears in logs, and send returns chunk count.
- Dependency: Step 3 gate passed.
- Pattern: Direct `requests` POST with chunk loop, same retry shape as `src/x_digest/x_api.py:176`. Fits no new dep. Trade-off is no `python-telegram-bot` helpers.
- Alternative: Use `python-telegram-bot` `Bot.send_message`. Rejected for extra async dependency for one endpoint.
- Out of scope: LLM, digest prompt, pipeline status.
- Risk and rollback: HTML parse error from unescaped `<`. Mitigation is escape and test with `text` containing `<` and `&`. Rollback is `rm src/x_digest/telegram.py`.
- Gate command or check: `uv run ruff check src/x_digest/telegram.py && uv run pytest -q -k telegram`
- Gate pass condition: `ruff` exits 0, tests show `test_telegram_chunks_long_digest` and `test_telegram_retries_on_429` passing.
- Stop condition: `ruff` or any `telegram` test fails. Fix sender before pipeline hook.

### Step 5: Wire digest as last stage of Pipeline.sync
- Repository: `x-digest`
- Complete paths: `x-digest/src/x_digest/pipeline.py:44` :: `class Pipeline`, `x-digest/src/x_digest/pipeline.py:105` :: `_archive_media_and_markdown`, `x-digest/src/x_digest/pipeline.py:327` :: manifest plus `runs` status block
- Change: In `x-digest/src/x_digest/pipeline.py:44` add imports `from .digest import DigestBuilder` and `from .telegram import TelegramSender` and `from .llm import LlmClient`. Add method `_send_digest(self, run_id: str, counts: dict[str, int]) -> None:` that checks `self.settings.telegram_enabled()` and `self.settings.llm_enabled()`, else logs `digest_skipped` with `reason=missing_config` at `info` level and returns. When enabled, calls `DigestBuilder(self.database, self.settings).collect_new_posts(run_id)`, builds prompt, calls `LlmClient(...).complete` when posts non-empty, calls `TelegramSender(...).send(text)`, updates `counts["digest_chunks"]`, `counts["digest_posts"]`, `counts["digest_chars"]`, and emits `digest_sent` with `chunks`, `posts`, `chars`, plus `post_ids` truncated to 20. On any `Exception` log `llm_failed` or `telegram_failed` at `warning` with `error` truncated to 500 chars, update `counts["digest_failed"]=1`, do not raise, do not change `runs.status`. Insert call `self._send_digest(run_id, counts)` inside `sync` after `self._archive_media_and_markdown(run_id, counts)` at `x-digest/src/x_digest/pipeline.py:327` and before `self.bronze.write_run_manifest(run_id)`. Also handle `dry_run` and `max_pages` bounded sync: call digest only when `not dry_run and max_pages is None`, otherwise emit `digest_skipped` `reason=bounded_sync` or `dry_run`.
- Acceptance criteria: Unconfigured `sync` still succeeds and logs `digest_skipped`. Configured `sync` with new posts sends themed digest, records `digest_sent`, and updates `runs.counts_json` with digest counts. LLM or Telegram failure does not flip `runs.status` to `failed`.
- Dependency: Step 4 gate passed.
- Pattern: Best-effort final stage before manifest, same as media and Markdown pattern. Fits existing `counts` and `run_events` flow. Trade-off is digest is not retried outside the run.
- Alternative: Send digest after `write_run_manifest` and update `runs` again. Rejected because it adds extra `runs` update and manifest no longer reflects digest.
- Out of scope: Changing bookmark pagination, media download, or backup script.
- Risk and rollback: Digest raises and fails the run. Mitigation is `try/except` that swallows and logs. Rollback is `git checkout -- src/x_digest/pipeline.py`.
- Gate command or check: `uv run ruff check src/x_digest/pipeline.py && uv run pytest -q -k "pipeline or sync" && uv run python -c "import ast, pathlib; print('digest' in pathlib.Path('src/x_digest/pipeline.py').read_text())"`
- Gate pass condition: `ruff` exits 0, pipeline tests pass including `test_sync_sends_themed_digest`, and the last Python prints `True`.
- Stop condition: `ruff` fails, pipeline test fails, or the file does not contain `digest`. Fix pipeline before CLI.

### Step 6: Add CLI digest preview and send commands
- Repository: `x-digest`
- Complete paths: `x-digest/src/x_digest/cli.py:48` :: `build_parser`, `x-digest/src/x_digest/cli.py:79` :: `main` dispatch, `x-digest/src/x_digest/config.py:31` :: `load_settings`
- Change: In `x-digest/src/x_digest/cli.py:48` add subparser `digest` with args `--dry-run` (default true when no flag, prints without network), `--send` (requires config, calls OpenRouter and Telegram), `--limit INT` override for `digest_max_posts`, `--model STR` override for `llm_model` for one-off test. In `x-digest/src/x_digest/cli.py:79` add branch `elif args.command == "digest":` that loads settings, overrides if provided, creates `Database(settings.database_path).initialize()`, uses `DigestBuilder` and `LlmClient` and `TelegramSender` same as pipeline, prints JSON `{"text": str, "post_ids": [], "chunks": int}` to stdout. `--dry-run` must not require Telegram or LLM config, it prints the rendered prompt and the fake summary. `--send` requires both `telegram_enabled()` and `llm_enabled()` or exit 2 with error message.
- Acceptance criteria: `uv run x-digest digest --dry-run` prints JSON with `text` and does not call network. `uv run x-digest digest --send` with missing config exits 2. With dummy config and mocked network the command exits 0.
- Dependency: Step 5 gate passed.
- Pattern: CLI preview plus send, same dispatch shape as `verify`, `search`, `show` in `x-digest/src/x_digest/cli.py:48`. Fits existing `argparse` plus `Database` init.
- Alternative: Add `send-digest` as separate top-level binary. Rejected because it duplicates settings plumbing.
- Out of scope: Scheduler plist, backup script, docs.
- Risk and rollback: CLI import fails when `requests` not installed. Mitigation is reuse `requests` already present. Rollback is `git checkout -- src/x_digest/cli.py`.
- Gate command or check: `uv run ruff check src/x_digest/cli.py && uv run x-digest digest --dry-run 2>&1 | head -n 50 && uv run x-digest digest --help 2>&1 | grep -F -- --send`
- Gate pass condition: `ruff` exits 0, first command prints JSON with `"text"`, second grep finds `--send`.
- Stop condition: `ruff` fails or `digest --dry-run` does not print JSON. Fix CLI before docs.

### Step 7: Update .env.example, README, and tech-context for OpenRouter Telegram digest
- Repository: `x-digest`
- Complete paths: `x-digest/.env.example:1` :: env block, `x-digest/README.md:143` :: CLI reference, `x-digest/README.md:191` :: Backup section, `x-digest/tech-context.md:514` :: Pipeline section, `x-digest/.gitignore:1` :: secrets boundary
- Change: In `x-digest/.env.example:1` keep the 7 existing keys and add block `XDIGEST_TELEGRAM_BOT_TOKEN`, `XDIGEST_TELEGRAM_CHAT_ID`, `XDIGEST_LLM_API_KEY`, `XDIGEST_LLM_MODEL=openai/gpt-oss-120b`, `XDIGEST_LLM_BASE_URL=https://openrouter.ai/api/v1` with comments that Keychain `x-digest` accounts `telegram-bot-token`, `telegram-chat-id`, `openrouter-api-key` are read as fallback, that routing to cheapest ZDR provider is handled in `src/x_digest/llm.py:10` via `provider: {sort: "price", zdr: true, data_collection: "deny"}` so the model stays `openai/gpt-oss-120b` without `:floor` suffix, and that `data/.env` stays ignored. In `x-digest/README.md:143` add `## Weekly Telegram digest` subsection after `## CLI reference` that explains Sunday 06:00 `launchd` digest, OpenRouter `openai/gpt-oss-120b` via cheapest ZDR provider, setup via `@BotFather` and `curl getUpdates`, themed summary, 4096 chunking, and `uv run x-digest digest --dry-run` preview. In `x-digest/tech-context.md:514` add `14.3 Weekly Telegram content digest (v0)` that describes OpenRouter `openai/gpt-oss-120b` flow with `provider.sort: price` plus `zdr: true`, prompt budget, chunking, retry, and `run_events` `digest_sent`/`digest_skipped`/`llm_failed`/`telegram_failed`. Verify `.gitignore` already ignores `.env` and `data/` so no edit needed, but gate checks it.
- Acceptance criteria: Docs describe OpenRouter as v0, list the five env keys, explain `getUpdates` for chat ID, and note chunking and best-effort retry. Two docs plus `.env.example` changed.
- Dependency: Step 6 gate passed.
- Pattern: Minimal doc alignment with implementation, same as prior backup doc step `x-digest/.agents/plans/2026-08-30-gcp-free-tier-backup.md:150`. Trade-off is local LLM hosting doc is deferred to v1.
- Alternative: Keep Telegram setup only in `tech-context.md`. Rejected because `README.md` is the user entry point for `.env` keys.
- Out of scope: Changing scheduler timing, backup script, or Bronze/Silver schema.
- Risk and rollback: Docs and code drift on key names. Rollback is `git checkout -- .env.example README.md tech-context.md`.
- Gate command or check: `grep -F XDIGEST_TELEGRAM_BOT_TOKEN .env.example && grep -F XDIGEST_LLM_API_KEY .env.example && grep -F "Weekly Telegram digest" README.md && grep -F "openrouter.ai" tech-context.md`
- Gate pass condition: Each grep returns at least one match and no token value appears in docs.
- Stop condition: Any grep fails or docs contain a real token. Correct docs before final gate.

### Step 8: Manual end-to-end gate with dry-run and live Telegram check
- Repository: `x-digest`
- Complete paths: `x-digest/src/x_digest/pipeline.py:244` :: `sync`, `x-digest/src/x_digest/digest.py:1`, `x-digest/src/x_digest/telegram.py:1`, `x-digest/data/logs/runs/<run_id>.jsonl:1` :: per-run log
- Change: No file edit. Run gates in order: 1) Dry-run without secrets, 2) Mocked live test, 3) Real credential test on a temp vault. This is the delivery gate.
- Acceptance criteria: Dry-run prints themed text without network, mocked live test logs `digest_sent`, and real-credential test on a temp vault delivers a Telegram message that contains themes and source links and logs `digest_sent` in `run_events` and `data/logs/runs/<run_id>.jsonl`.
- Dependency: Step 7 gate passed.
- Pattern: Dry-run plus mocked plus live `verify --full` pattern, same as `x-digest/.agents/plans/2026-08-30-gcp-free-tier-backup.md:165` final gate. Fits best-effort digest without touching live `data/` until the last subcheck.
- Alternative: Trust mocked test alone. Rejected because it does not exercise OpenRouter plus Telegram auth.
- Out of scope: Pruning old backups, scheduling a separate digest timer.
- Risk and rollback: Live test spams the chat. Mitigation is use a test chat ID then switch to real. Rollback is `install-scheduler.sh --remove` has no digest side effect, digest can be disabled by unsetting env keys.
- Gate command or check: `uv run pytest -q && uv run ruff check src tests && XDIGEST_TELEGRAM_BOT_TOKEN=dummy XDIGEST_TELEGRAM_CHAT_ID=123 XDIGEST_LLM_API_KEY=dummy uv run x-digest digest --dry-run 2>&1 | grep -F '"text"' && uv run pytest -q -k "digest or telegram or llm" && echo "live gate: set real XDIGEST_TELEGRAM_BOT_TOKEN, XDIGEST_TELEGRAM_CHAT_ID, XDIGEST_LLM_API_KEY then run: uv run x-digest digest --send 2>&1 | head; rclone-style log check: tail -n 20 data/logs/runs/*.jsonl | grep -F digest_sent"`
- Gate pass condition: First `pytest` and `ruff` exit 0, `digest --dry-run` prints JSON with `"text"`, mocked `digest or telegram or llm` tests pass, and the live `digest --send` exits 0 and `data/logs/runs/<run_id>.jsonl` contains `digest_sent` with no token.
- Stop condition: Any subcommand in the chain exits non-zero, or live log contains a token, or Telegram message is not themed. Diagnose `data/logs/runs/<run_id>.jsonl` and `run_events` before declaring delivery complete.

## Verification summary
Each step has a mandatory gate command and pass condition listed above. No next step starts with a failing gate. Evidence to retain after delivery is: `ruff` output, `pytest -q` output, `digest --dry-run` JSON, `data/logs/runs/<run_id>.jsonl` digest lines, `SELECT event, details_json FROM run_events WHERE run_id='<run_id>'` rows, and one real Telegram message screenshot or `getUpdates` proof.

## Open decisions before delivery
- None for v0. Model is fixed to `openai/gpt-oss-120b` with `provider: {sort: "price", zdr: true, data_collection: "deny"}` and `zdr: true`. If OpenRouter reports no ZDR endpoint for this model in the future, delivery must stop and ask for a new ZDR-compliant model choice.

