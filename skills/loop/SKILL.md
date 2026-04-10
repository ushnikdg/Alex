---
name: alex-loop
description: >
  Start a timed auto-ingest loop in the current terminal session. Checks raw/
  for new files every X hours or minutes and ingests them silently. Use when
  the user wants to automate ingestion, says "loop every hour", "auto-ingest
  every 2 hours", "keep checking for new files", or "run in the background".
allowed-tools: Bash Read Write Edit Glob Grep
---

# alex — Loop

Run a timed ingest loop in the current terminal session. Every X interval,
check `raw/` for new unprocessed files and ingest them silently (no discussion).

## Step 1: Get Interval

Ask:
> "How often should I check for new files? Enter an interval like `1h`, `2h`,
> `30m`. Default: `1h`."

Parse the input:
- `Nh` or `N h` → N × 3600 seconds
- `Nm` or `N m` → N × 60 seconds
- Bare number → treat as hours
- Default (empty) → 3600 seconds (1 hour)

Store as INTERVAL_SECS and INTERVAL_LABEL (human-readable, e.g. "1 hour").

## Step 2: Locate Vault

Check that `wiki/log.md` exists in the current directory. If not, tell the
user to run this skill from their vault root and stop.

## Step 3: Start the Loop

Tell the user:
> "Starting alex loop. Checking every INTERVAL_LABEL. Press Ctrl+C to stop."

Then run the following loop using Bash. Keep a counter of files processed
across all iterations.

```bash
PROCESSED_TOTAL=0

while true; do
  TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
  echo "[$TIMESTAMP] Checking raw/ for new files..."

  # Find all .md files in raw/ (excluding assets/)
  RAW_FILES=$(find raw/ -maxdepth 1 -name "*.md" -type f 2>/dev/null | sort)

  if [ -z "$RAW_FILES" ]; then
    echo "  No files in raw/. Sleeping INTERVAL_LABEL..."
    sleep INTERVAL_SECS
    continue
  fi

  # Extract filenames already logged
  INGESTED=$(grep -oP 'Processed \K[^\s.]+\.md' wiki/log.md 2>/dev/null || true)

  # Find unprocessed files
  NEW_FILES=()
  while IFS= read -r filepath; do
    fname=$(basename "$filepath")
    if ! echo "$INGESTED" | grep -qF "$fname"; then
      NEW_FILES+=("$fname")
    fi
  done <<< "$RAW_FILES"

  if [ ${#NEW_FILES[@]} -eq 0 ]; then
    echo "  No new files. Sleeping INTERVAL_LABEL..."
    sleep INTERVAL_SECS
    continue
  fi

  echo "  Found ${#NEW_FILES[@]} new file(s): ${NEW_FILES[*]}"
  echo "  Ingesting silently..."
  # (LLM takes over here — see Silent Ingest below)
done
```

## Step 4: Silent Ingest

When the Bash loop detects new files, pause the loop and run the ingest
workflow from `/alex-ingest` for each detected file, **in silent mode**:
- Skip step 2 (discuss key takeaways) entirely
- Process all detected files automatically
- Write wiki pages, update index.md and log.md as normal

After ingest completes, add the count to PROCESSED_TOTAL, print a one-line
summary, then resume the loop (`sleep INTERVAL_SECS`).

One-line summary format:
```
  Done. Created N pages, updated M pages. Total this session: T files.
```

## Step 5: On Stop (Ctrl+C)

When the user stops the loop, print:
> "Loop stopped. Processed T file(s) this session."

## Notes

- The loop runs **foreground** — the terminal window must stay open.
- The interval timer starts **after** each ingest finishes, not on a fixed clock.
- Files already in `wiki/log.md` are never re-ingested.
- If `raw/` is empty on every check, the loop still runs — it just sleeps.

## Related Skills

- `/alex-ingest` — manual ingest with interactive discussion
- `/alex-query` — ask questions against the wiki
- `/alex-lint` — health-check the wiki
