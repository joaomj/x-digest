# X Digest Data Pipeline Plan

## Scope

Build a local, private, append-only X bookmark archive.

- Use the official X API and XDK.
- Read bookmarks only. Do not make write operations in X.
- Do not generate summaries or use an LLM in this phase.
- Store raw API content and downloaded media locally.
- Keep the Bronze archive append-only. Do not delete or modify archived data.
- Provide a command-line interface first. Do not build a web application in this phase.

Use these OAuth scopes:

- `bookmark.read`
- `tweet.read`
- `users.read`
- `offline.access`

Do not request `tweet.write`.

## Architecture

Use a simplified medallion pipeline.

```text
X API bookmark sync
  -> Bronze: immutable raw API captures and media files
  -> Silver: normalized post records
  -> Gold: searchable local catalog and exports
```

The Bronze archive is the source of truth. The system must be able to rebuild
the Silver and Gold layers from Bronze data.

## Bronze Layer

The Bronze layer stores every successful X API response exactly as received.

- Write each raw response as compressed JSON.
- Write each object and manifest atomically.
- Never update or delete a Bronze object.
- Create a new observation when a later sync returns a known post.
- Store downloaded original media files as immutable archive objects.
- Record a failed media download independently. A failed download must not
  invalidate the post capture.

Each object must include or have a linked manifest with:

- `run_id`
- Fetch time
- API endpoint
- Pagination cursor
- Observed source IDs
- Payload SHA-256 hash
- Schema version
- Download status for media
- Media SHA-256 hash when available

Use this directory shape:

```text
vault/
  bronze/
    2026/07/22/<run-id>/
      folders.json.gz
      bookmarks-page-0001.json.gz
      manifest.json
      media/
        <media-key>.<extension>
```

## Silver Layer

The Silver layer parses Bronze objects into normalized SQLite records.

Use these main records:

- `posts`: Canonical X post ID and current normalized projection.
- `post_versions`: One normalized version for each distinct content hash.
- `post_observations`: Each time the sync sees a post. Link it to the exact
  Bronze object and run.
- `authors`: Author identifiers and source metadata.
- `folders`: Bookmark folder identifiers and names.
- `bookmark_memberships`: Observed post-to-folder membership.
- `media`: Media metadata, archive path, hash, and download status.
- `references`: Referenced post relationships.
- `runs`: One record for each pipeline execution.
- `run_events`: Structured events for each run stage.
- `checkpoints`: Durable extractor progress.

Normalize and retain:

- Ordinary post text
- Note Tweet text
- X Article body and metadata
- Canonical URL
- Author identity
- Creation time
- Language
- Public metrics
- Attachments
- Referenced posts
- Folder membership
- Retrieval times

Do not assume that the existing fetch skill stores complete X Article content.
Run a live data-shape spike with representative bookmarked Articles. Confirm
which X API fields contain the full Article body before the normalizer maps it.

Normalization must be repeatable. Rebuilding Silver from the same Bronze
objects must produce the same result.

## Gold Layer

The Gold layer provides local access to normalized records.

- Use SQLite FTS5 for full-text search.
- Index post text, Article body, author, folder, URL, and date.
- Generate Markdown and JSON exports from Silver records.
- Preserve canonical X URLs and source attribution in every export.
- Treat exports as derived artifacts. Regenerate them when needed.

Provide these CLI commands:

- `sync`
- `status`
- `search <query>`
- `show <post-id>`
- `export --format markdown|json`
- `verify`
- `rebuild-silver`

## Operational Design

- Use a typed Python project managed with `uv`.
- Keep application source in this repository.
- Keep vault data in a configurable local path. The default path is
  `~/Library/Application Support/x-digest/`.
- Store OAuth refresh tokens in macOS Keychain.
- Do not store tokens in `.env`, SQLite, or logs.
- Use the official XDK for OAuth, pagination, and API requests.
- Do not use browser automation or scraping.

Make all pipeline mutations safe to retry:

- Use a per-run process lock.
- Use SQLite transactions.
- Use atomic file writes.
- Checkpoint after each processed API page.
- Use bounded exponential retries for transient X API and network failures.
- Use idempotent upserts for normalized records.
- Store content hashes for raw payloads and normalized versions.
- Store checksums for downloaded media.

Persist observability data:

- Write structured JSONL application logs.
- Write `runs` and `run_events` records with the same `run_id`.
- Record stage, counts, duration, error class, retry status, and final status.
- Do not write credentials or OAuth tokens to logs.
- Provide `sync --dry-run` and `verify --full` commands.

## Login Automation

Install a per-user macOS `launchd` LaunchAgent.

- Use `RunAtLoad` to start the job when the user logs in.
- Store the date of the last successful sync in durable state.
- Exit with an `already_current` run event when the current local date already
  has a successful sync.
- Record network and authentication failures as retriable failures.
- Retry at a bounded interval while the user remains logged in.
- Configure `launchd` to write stdout and stderr to persistent log files.
- Include the application `run_id` in all logs and database events.

## Implementation Order

1. Define vault paths, configuration, log format, and the SQLite schema.
2. Create the immutable Bronze writer with manifests, hashes, atomic writes,
   and verification.
3. Implement OAuth PKCE setup with macOS Keychain token storage.
4. Implement X bookmark and folder extraction with pagination, retries, and
   durable page checkpoints.
5. Run an X Article data-shape spike against real bookmarked Articles.
6. Implement Silver normalization for posts, Note Tweets, Articles, folder
   membership, references, and media metadata.
7. Add original-media downloads, checksums, and independent failure tracking.
8. Add SQLite FTS5 search and CLI commands for access and exports.
9. Add rebuild-from-Bronze and integrity-verification commands.
10. Add the daily-at-login `launchd` scheduler, process lock, and retry state.
11. Add end-to-end CLI tests with an isolated real SQLite database and vault
    filesystem.

## Verification

The implementation must verify these behaviors through end-to-end CLI tests:

- A sync stores raw API pages and their manifests.
- A repeated sync does not duplicate normalized records.
- An interrupted sync resumes from a durable checkpoint.
- A known post can have multiple immutable Bronze observations.
- A malformed or incomplete Article payload is visible as an error state.
- A failed media download does not discard its associated post capture.
- `rebuild-silver` produces the same normalized records from Bronze data.
- `verify --full` detects a changed or missing Bronze object or media file.
- A second scheduler invocation on the same local date exits without a second
  successful sync.
