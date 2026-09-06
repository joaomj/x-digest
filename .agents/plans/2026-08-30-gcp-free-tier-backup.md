# Plan: GCP Free Tier backup for x-digest with personal account

## Goal
Keep a reliable off-machine copy of `data/bronze/`, the consistent `silver.sqlite` snapshot, and `data/logs/` on Google Cloud Storage using your personal GCP account, within the Always Free limits, without Drive OAuth expiry. The user sees a weekly 06:15 backup that logs `backup end` and never deletes remote files.

## Deliverable
Approved repository plan at `.agents/plans/2026-08-30-gcp-free-tier-backup.md` that authorizes `software-delivery` to repoint the existing backup flow from Drive to a private GCS bucket. No code is changed by this plan file.

## Work stages
1. Inspect current backup contract and scheduler.
2. Define GCP console work that matches Always Free constraints.
3. Map rclone GCS authentication to a bucket-scoped service account.
4. Map the minimal script, scheduler, and doc edits that preserve the snapshot and `rclone check` safety.
5. Define objective gates and rollback for each step.

## Reason for chosen path
GCS keeps the same `sqlite3 .backup` staging and `rclone copy` / `copyto` / `check` pattern that already exists in `x-digest/scripts/backup-to-drive.sh:25-41`. A single Standard regional bucket in `us-central1`, `us-west1`, or `us-east1` stays inside Always Free for this archive size, has no minimum duration or retrieval fee, and a bucket-scoped service account avoids the Drive Testing-mode refresh token expiry. Current size is small (last success `786 matching files`), so 5 GB free covers it.

## Decisions required before delivery
- Confirm GCP project ID to use (create new or reuse existing personal project).
- Choose bucket region among `us-central1`, `us-west1`, `us-east1` for Always Free.
- Choose globally unique bucket name.
- Accept service account JSON key file at `~/.config/gcloud/x-digest-backup-key.json` with `600` permissions, or request Workload Identity alternative.

## Context and evidence
- Current behavior: `x-digest/scripts/backup-to-drive.sh:30-41` stages `silver.sqlite` with `.backup`, copies `data/` excluding `silver.sqlite*`, uploads snapshot with `copyto`, verifies with `check --one-way`, logs to `x-digest/data/logs/backup.log`.
- Scheduler: `x-digest/scripts/install-backup-scheduler.sh:30-63` installs `com.x-digest.backup` at Sunday 06:15, and `x-digest/scripts/install-scheduler.sh:35-71` installs sync at 06:00.
- Docs: `x-digest/README.md:191-224` and `x-digest/tech-context.md:1004-1018` describe Drive backup. `x-digest/data/` is ignored by Git, `rclone v1.75.0` is installed at `/Users/joao/homebrew/bin/rclone`, `gcloud` is not yet installed.
- Cloud docs verified in research: `https://rclone.org/googlecloudstorage/` supports `service_account_file`, `https://cloud.google.com/storage/pricing` defines Always Free 5 GB, `https://docs.cloud.google.com/storage/docs/creating-buckets` defines uniform bucket-level access and soft delete defaults, `https://docs.cloud.google.com/iam/docs/keys-create-delete` states keys do not expire by default.

## Acceptance criteria
- One private bucket exists with `STANDARD`, uniform bucket-level access enabled, public access prevention enforced, soft delete 7 days, location in an Always Free region.
- One service account has only bucket-scoped `roles/storage.objectUser` on that bucket, one active JSON key at `~/.config/gcloud/x-digest-backup-key.json` with `600`.
- `rclone` remote `gcs` of type `google cloud storage` uses `service_account_file` and can `lsd gcs:<bucket>`.
- `x-digest/scripts/backup-to-drive.sh` points at `gcs:<bucket>` and retains the snapshot and `check` flow.
- Launchd agent `com.x-digest.backup` still runs Sunday 06:15 and points at the updated script.
- `README.md` and `tech-context.md` describe the GCS path, no Drive OAuth reference remains as the primary path.
- One manual backup run logs `backup end`, `rclone check` reports `0 errors`, and `uv run x-digest verify --full` passes after a restore drill on a temp vault.

## Dependencies
- Google Cloud project with billing linked, even when monthly charge is zero inside Always Free.

## Risks
- Storing the JSON key with loose permissions or committing it exposes the bucket. Mitigation is `chmod 600`, `.gitignore` already ignores `data/` and `.env`, and key is outside the repo.
- Choosing a bucket location outside Always Free regions incurs a small charge. Mitigation is creation gate that checks location.
- Overwriting `silver.sqlite` loses prior versions unless soft delete is kept. Mitigation is keeping the default 7 day soft delete and never enabling delete of remote files.
- Missing `gcloud` on this Mac requires console-only steps. Mitigation is a console-described gate for each cloud step.

## Out of scope
- Changing Bronze/Silver schema, pipeline, search, or `silver.sqlite` snapshot method.
- Adding encryption beyond GCS default or client-side crypt.
- Migrating old Drive backup history to GCS.
- Changing the weekly 06:00 sync schedule.
- Committing `data/` or the service account key to Git.

## Selected pattern and trade-off
Keep the existing shell plus rclone copy pattern and repoint the remote. This reuses the proven staging and verification, limits churn to 3 repo files, and adds no Python dependency. Trade-off is continued reliance on a long-lived JSON key file that must be protected and rotated manually.

## Alternative considered
Local-only external disk copy plus free-tier R2/B2. Rejected because the user chose GCP consolidation and Always Free already covers this size at zero cost.

---

## Repository: x-digest

### Step 1: Confirm or create GCP project and link billing
- Repository: `x-digest`
- Complete paths: `x-digest/scripts/backup-to-drive.sh:7-8` :: `REMOTE`/`DEST` destination depends on project
- Change: In Google Cloud Console, confirm the personal project ID to use. If no suitable project exists, create one. Link a billing account to the project via Billing > Link account. No repo file is edited in this step.
- Acceptance criteria: Project exists and shows a linked billing account. Future bucket creation is unblocked.
- Dependency: None. This is the first gate.
- Pattern: Console project plus billing linkage. Fits because GCS requires billing even inside Always Free. Trade-off is one-time billing setup.
- Alternative: Skip billing and stay on Drive. Rejected because Drive expiry remains and GCS Always Free still requires billing linkage.
- Out of scope: Creating the bucket, service account, or any repo edit.
- Risk and rollback: Wrong project chosen. Rollback is to select a different project ID before any bucket exists; no data is created yet.
- Gate command or check: In console open Billing for the project and run `gcloud projects describe PROJECT_ID --format="value(billingAccountName)"` if `gcloud` is available.
- Gate pass condition: Console shows `Billing account: <account>` and the command returns a non-empty `billingAccounts/...` string.
- Stop condition: Billing shows `Not linked` or the command returns empty. Do not proceed to bucket creation.

### Step 2: Create private GCS bucket inside Always Free
- Repository: `x-digest`
- Complete paths: `x-digest/scripts/backup-to-drive.sh:8` :: `DEST` value will reference this bucket
- Change: Create one bucket via Console (or `gcloud storage buckets create gs://BUCKET_NAME --location=us-central1 --default-storage-class=STANDARD --uniform-bucket-level-access --public-access-prevention --soft-delete-duration=7d`). Location must be `us-central1`, `us-west1`, or `us-east1`. Name must be globally unique, for example `x-digest-backup-<random>`. Keep versioning off.
- Acceptance criteria: Bucket exists, `STANDARD`, location in Always Free region, uniform bucket-level access enabled, public access prevention enforced, soft delete 7 days.
- Dependency: Step 1 gate passed.
- Pattern: Single regional Standard bucket with private defaults. Fits Always Free and avoids minimum durations or retrieval fees. Trade-off is 5 GB free cap versus 10 GB on other providers.
- Alternative: Dual-region Standard for higher availability. Rejected because dual-region costs more and is not Always Free.
- Out of scope: Service account, rclone remote, script edits.
- Risk and rollback: Bucket name collision or wrong location. Rollback is `gcloud storage buckets delete gs://BUCKET_NAME` before any backup runs if the bucket is empty.
- Gate command or check: `gcloud storage buckets describe gs://BUCKET_NAME --format="json(name,location,storageClass,uniformBucketLevelAccess,publicAccessPrevention,softDeletePolicy)"`
- Gate pass condition: JSON shows `location` is `us-central1` or one of the two other Always Free regions, `storageClass` is `STANDARD`, `uniformBucketLevelAccess.enabled` is `true`, `publicAccessPrevention` is `enforced`, and soft delete retention is `604800s`.
- Stop condition: Location is not Always Free, class is not Standard, or uniform access is disabled. Fix the bucket before continuing.

### Step 3: Create bucket-scoped service account and key
- Repository: `x-digest`
- Complete paths: `x-digest/scripts/backup-to-drive.sh:13-19` :: `RCLONE_BIN` lookup and future `service_account_file` usage, plus `x-digest/.gitignore:1-10` :: ignored secrets boundary
- Change: In Console IAM, create service account `x-digest-backup@PROJECT_ID.iam.gserviceaccount.com`. Grant `roles/storage.objectUser` only on the bucket from Step 2 via bucket Permissions tab, not at project level. Create one JSON key, download to `~/.config/gcloud/x-digest-backup-key.json`, run `chmod 600 ~/.config/gcloud/x-digest-backup-key.json`. Do not place the file in the repo.
- Acceptance criteria: Service account exists, has exactly one active key, has bucket-scoped `objectUser` on only this bucket, key file exists outside the repo with `600`.
- Dependency: Step 2 gate passed.
- Pattern: Least-privilege bucket-scoped service account with long-lived JSON key for unattended launchd runs. Fits rclone GCS auth. Trade-off is manual key custody.
- Alternative: ADC via `gcloud auth application-default login`. Rejected because it requires `gcloud` on this Mac and periodic reauth for unattended jobs.
- Out of scope: rclone remote creation, script or doc edits.
- Risk and rollback: Key with broad project permissions or committed key. Rollback is to remove the project-level grant, delete the key with `gcloud iam service-accounts keys delete KEY_ID --iam-account=SA_EMAIL`, and recreate with bucket-only grant.
- Gate command or check: `ls -l ~/.config/gcloud/x-digest-backup-key.json && jq -r .client_email ~/.config/gcloud/x-digest-backup-key.json && gcloud iam service-accounts keys list --iam-account=SA_EMAIL --format="table(KEY_ID,VALID_AFTER_TIME,DISABLED)" && gcloud storage buckets get-iam-policy gs://BUCKET_NAME --format="json(bindings)"`
- Gate pass condition: `ls -l` shows `-rw-------`, `client_email` equals the new service account, keys list shows one `USER_MANAGED` key not disabled, and the bucket IAM policy shows `roles/storage.objectUser` for that service account and no project-wide `roles/storage.admin`.
- Stop condition: Key file is world-readable, exists inside `x-digest/`, or the service account has project-level storage admin. Fix before configuring rclone.

### Step 4: Configure rclone GCS remote on the Mac
- Repository: `x-digest`
- Complete paths: `x-digest/scripts/backup-to-drive.sh:7-19` :: `REMOTE`/`DEST` and `RCLONE_BIN` resolution
- Change: On this Mac run `rclone config` and create remote `gcs` of type `google cloud storage` with `service_account_file = /Users/joao/.config/gcloud/x-digest-backup-key.json` and `project_number` set to the project number if prompted for bucket creation access. Do not set `anonymous` or `env_auth`. Leave Drive remote `xdigest` untouched for now.
- Acceptance criteria: `rclone config show` contains `[gcs]` with `type = google cloud storage` and `service_account_file` pointing at the Step 3 path, and the remote can list the empty bucket.
- Dependency: Step 3 gate passed.
- Pattern: Native rclone GCS service account auth. Fits unattended launchd without browser OAuth. Trade-off is path coupling to the key file.
- Alternative: Use `service_account_credentials` inline JSON in `rclone.conf`. Rejected because it duplicates the secret in the config file.
- Out of scope: Editing the backup script, scheduler, or docs.
- Risk and rollback: Wrong `project_number` or key path. Rollback is `rclone config update gcs service_account_file /correct/path` or `rclone config delete gcs` and recreate.
- Gate command or check: `rclone config show --check 2>&1 | sed -n '/\[gcs\]/,/^\[/p' && rclone lsd gcs:BUCKET_NAME --fast-list`
- Gate pass condition: Config snippet shows `[gcs]` with `type = google cloud storage` and the correct `service_account_file`, and `lsd` exits `0` with either empty output or the bucket object list.
- Stop condition: Config shows wrong type or key path, or `lsd` returns `403` or `404`. Do not edit the backup script until the remote lists the bucket.

### Step 5: Repoint the backup script to GCS and keep snapshot safety
- Repository: `x-digest`
- Complete paths: `x-digest/scripts/backup-to-drive.sh:7-8` :: `REMOTE`/`DEST` and `x-digest/scripts/backup-to-drive.sh:30-41` :: `sqlite3 .backup` plus `rclone copy` / `copyto` / `check` block
- Change: Edit `x-digest/scripts/backup-to-drive.sh:7-8` to set `REMOTE="gcs"` and `DEST="${REMOTE}:BUCKET_NAME"` where `BUCKET_NAME` is the Step 2 name. Keep `STAGING="$(mktemp -d)"`, `sqlite3 ... ".backup '$STAGING/silver.sqlite'"`, `rclone copy "$PROJECT_DIR/data/" "$DEST/" --exclude "silver.sqlite*"`, `rclone copyto "$STAGING/silver.sqlite" "$DEST/silver.sqlite"`, and `rclone check "$PROJECT_DIR/data/" "$DEST/" --exclude "silver.sqlite*" --one-way` exactly. Do not add delete or sync semantics. Keep logging to `x-digest/data/logs/backup.log`.
- Acceptance criteria: Script still stages a consistent snapshot before any upload, never deletes remote files, and references only `gcs:BUCKET_NAME`.
- Dependency: Step 4 gate passed.
- Pattern: In-place remote repoint with preserved copy semantics. Fits the smallest diff and preserves verification. Trade-off is the old Drive remote stays configured but unused until removed.
- Alternative: Create a new `backup-to-gcs.sh` and keep `backup-to-drive.sh` unchanged. Rejected because two backup scripts diverge and double maintenance.
- Out of scope: Scheduler plist content, doc prose, or Bronze/Silver logic.
- Risk and rollback: Staging removed or `check` dropped, causing unchecked uploads. Rollback is `git checkout -- scripts/backup-to-drive.sh` to restore the last committed script.
- Gate command or check: `bash -n x-digest/scripts/backup-to-drive.sh && grep -F 'REMOTE="gcs"' x-digest/scripts/backup-to-drive.sh && grep -F 'sqlite3' x-digest/scripts/backup-to-drive.sh && grep -F 'rclone check' x-digest/scripts/backup-to-drive.sh`
- Gate pass condition: `bash -n` exits `0`, first grep finds `REMOTE="gcs"`, second finds the `.backup` line, third finds the `rclone check` line.
- Stop condition: Syntax check fails or `REMOTE` is not `gcs` or the snapshot line is missing. Fix the script before touching the scheduler.

### Step 6: Reinstall the launchd backup scheduler on the updated script
- Repository: `x-digest`
- Complete paths: `x-digest/scripts/install-backup-scheduler.sh:6-7` :: `LABEL`/`PLIST` and `x-digest/scripts/install-backup-scheduler.sh:30-63` :: plist `ProgramArguments` and `StartCalendarInterval`
- Change: Run `x-digest/scripts/install-backup-scheduler.sh` to regenerate `~/Library/LaunchAgents/com.x-digest.backup.plist` so it executes the Step 5 script. Do not change `Weekday 0 Hour 6 Minute 15`, `StandardOutPath`, or `StandardErrorPath`. If the script was renamed, update only the single `ProgramArguments` path and reinstall.
- Acceptance criteria: `com.x-digest.backup` is loaded, points at the GCS-aware script, and still runs Sunday 06:15 after the 06:00 sync.
- Dependency: Step 5 gate passed.
- Pattern: Reuse existing launchd installer. Fits precedent in `x-digest/scripts/install-scheduler.sh:35-71`. Trade-off is reliance on user session launchd.
- Alternative: Add a cron job. Rejected because the project already standardizes on launchd for both agents.
- Out of scope: Changing the sync agent or backup script logic.
- Risk and rollback: Scheduler points at the old Drive script or is unloaded. Rollback is `x-digest/scripts/install-backup-scheduler.sh --remove` then reinstall, or `launchctl bootstrap` again.
- Gate command or check: `cat ~/Library/LaunchAgents/com.x-digest.backup.plist && launchctl print "gui/$(id -u)/com.x-digest.backup" 2>&1 | grep -E "ProgramArguments|StartCalendarInterval|state"`
- Gate pass condition: Plist `ProgramArguments` contains the bucket-aware `scripts/backup-to-drive.sh` path, `StartCalendarInterval` shows `Weekday 0 Hour 6 Minute 15`, and `launchctl print` shows `state = waiting` or `running` without `error`.
- Stop condition: Plist is missing, points at the old remote script, or the service is not loaded. Reinstall before verification.

### Step 7: Update README and tech-context to the GCS backup path
- Repository: `x-digest`
- Complete paths: `x-digest/README.md:191-224` :: `## Backup to Google Drive` section and `x-digest/tech-context.md:1004-1018` :: `14.2 Backup and restore` plus `x-digest/tech-context.md:1179-1180` :: agent reference
- Change: In `x-digest/README.md:191-224` rename the section to `Backup to Google Cloud Storage (Free Tier)` and replace the Drive plus `GOOGLE_DRIVE_CLIENT_ID` text with: GCS bucket `gs://BUCKET_NAME`, rclone type `google cloud storage`, service account file `~/.config/gcloud/x-digest-backup-key.json`, `rclone copy` semantics with no deletes, and `scripts/backup-to-drive.sh` as the entry point. In `x-digest/tech-context.md:1004-1018` replace the Drive remote description with the same GCS bucket, Standard regional, uniform access, public access prevention, soft delete 7 days. Keep the four-step backup list verbatim except for the destination. Retain the launchd 06:15 reference.
- Acceptance criteria: Docs describe the GCS backup as the primary path and no longer instruct Drive OAuth as the main flow. Two files changed.
- Dependency: Step 6 gate passed.
- Pattern: Minimal doc alignment with implementation. Fits the single-path standard. Trade-off is the old Drive instructions are removed from the primary flow.
- Alternative: Keep both Drive and GCS documented as equal options. Rejected because it keeps the expired Drive path prominent.
- Out of scope: Changing sync, search, or API cost sections.
- Risk and rollback: Docs and script drift. Rollback is `git checkout -- README.md tech-context.md`.
- Gate command or check: `grep -F "Google Cloud Storage" x-digest/README.md && grep -F "gs://" x-digest/README.md && grep -F "service_account_file" x-digest/README.md && grep -F "gs://" x-digest/tech-context.md`
- Gate pass condition: Each grep returns at least one match and no `GOOGLE_DRIVE_CLIENT_ID` remains in the backup section.
- Stop condition: Greps fail or Drive client ID remains in the backup section. Correct the docs before the final run.

### Step 8: Run the backup, verify the bucket, and drill a restore
- Repository: `x-digest`
- Complete paths: `x-digest/scripts/backup-to-drive.sh:25-42` :: `log` plus staging plus `rclone` commands and `x-digest/data/logs/backup.log:1` :: log output
- Change: No file edit. Execute one manual backup via `x-digest/scripts/backup-to-drive.sh`, then list and check the bucket, then restore into a temp vault and run `verify --full`. This is the delivery gate.
- Acceptance criteria: Backup completes, remote holds `bronze/` objects and one `silver.sqlite` from the snapshot, and a restored temp vault verifies cleanly.
- Dependency: Step 7 gate passed.
- Pattern: Manual run plus `rclone check` plus app-level `verify --full` on a temp vault. Fits `x-digest/tech-context.md:14.2` restore guidance without touching the live `data/`.
- Alternative: Trust `rclone check` alone without a restore drill. Rejected because it does not exercise `verify --full` on restored files.
- Out of scope: Pruning old Drive backups or enabling bucket versioning.
- Risk and rollback: Bucket exceeds Always Free or upload fails mid-copy. Rollback is to keep the backup disabled (`install-backup-scheduler.sh --remove`) and retry after fixing billing or key scope; live `data/` is never mutated by this step.
- Gate command or check: `x-digest/scripts/backup-to-drive.sh; echo EXIT:$?; tail -n 20 x-digest/data/logs/backup.log; rclone check x-digest/data/ gcs:BUCKET_NAME/ --exclude "silver.sqlite*" --one-way --fast-list; rclone ls gcs:BUCKET_NAME/ | head -n 20; mkdir -p /tmp/x-digest-restore-test && rclone copy gcs:BUCKET_NAME/ /tmp/x-digest-restore-test/ --fast-list && XDIGEST_VAULT_PATH=/tmp/x-digest-restore-test uv run x-digest verify --full`
- Gate pass condition: Script exits `0`, `backup.log` ends with `backup end`, `rclone check` exits `0` with `0 errors`, `rclone ls` lists `silver.sqlite` and `bronze/`, and the final `verify --full` exits `0`.
- Stop condition: Any subcommand in the gate chain exits non-zero. Diagnose `backup.log` and `rclone` output before declaring delivery complete.

## Verification summary
Each step has a mandatory gate command and pass condition listed above. No next step starts with a failing gate. Evidence to retain after delivery is: `gcloud storage buckets describe` JSON, bucket IAM policy JSON, `rclone config show` snippet, `backup.log` tail, `rclone check` output, and `verify --full` output on the restore.

## Open decisions before delivery
- Final bucket name and region choice.
- Final project ID choice if more than one personal project exists.
- Retention of the old Drive remote `xdigest` in `rclone.conf` or its removal after the first successful GCS backup.

