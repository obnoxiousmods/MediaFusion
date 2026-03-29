# DMM Hashlist Scraper — How It Works

## Overview

The DMM (Debrid Media Manager) hashlist scraper ingests torrent metadata from the public GitHub repo `debridmediamanager/hashlists`. Each commit in that repo contains HTML files with LZ-compressed JSON payloads of torrent info (filename, info_hash, size). The scraper walks through commits, decodes the payloads, resolves metadata (title/year -> TMDB), and stores the torrent streams in PostgreSQL.

This is the single most valuable source for MediaFusion — it contains millions of curated torrent hashes with filenames that can be parsed into structured metadata.

---

## Two Entry Points

### 1. Scheduled: `run_dmm_hashlist_scraper()`
- Runs on cron (default: `0 * * * *` = every hour)
- Processes only **new commits since last run** (incremental) + a small backfill batch
- Conservative: 200 commits incremental + 200 backfill per run
- Guarded by `minimum_run_interval` to prevent overlapping runs

### 2. Manual: `run_dmm_hashlist_full_ingestion()`
- Triggered from admin panel ("Run Full")
- Loops up to 200 iterations of `scraper.run()`
- Stops when: backfill completes, no progress made, or max iterations hit
- Used for initial population of the database

---

## Data Flow

```
GitHub Commits API
       |
       v
+------------------+
| Fetch Commits    |  GET /repos/debridmediamanager/hashlists/commits
| (incremental +   |  Rate: 60/hr unauth, 5000/hr with token
|  backfill)       |
+------------------+
       |
       v
+------------------+
| For each commit: |  GET /repos/.../commits/{sha}
| Fetch file list  |  Returns: modified .html files with blob SHAs
+------------------+
       |
       v
+------------------+
| Blob SHA dedup   |  Redis SET: skip files already processed
+------------------+
       |
       v
+------------------+
| Fetch HTML       |  Raw GitHub URL -> HTML content
| Extract iframe   |  Regex: <iframe src="...#ENCODED_PAYLOAD">
| Decompress LZ    |  LZString URI component decompression
| Parse JSON       |  {"torrents": [{"filename", "hash", "bytes"}, ...]}
+------------------+
       |
       v
+------------------+
| Validate entries |  - Info hash must be 40 hex chars (SHA1)
|                  |  - Must have filename + hash
|                  |  - Deduplicate by info_hash
+------------------+
       |
       v
+------------------+
| Filter           |  - Skip adult content (keyword list)
|                  |  - Skip sports broadcasts (for movies)
+------------------+
       |
       v
+------------------+
| Parse title      |  PTT (parsett) library extracts:
| with PTT         |  title, year, resolution, codec, quality,
|                  |  seasons, episodes, languages, audio, HDR
|                  |
|                  |  Media type = "series" if seasons/episodes
|                  |  found, else "movie"
+------------------+
       |
       v
+------------------+
| Resolve metadata |  Two-tier lookup (per unique title+year+type):
|                  |
|                  |  Tier 1: Search existing PostgreSQL database
|                  |    -> search_media() by title, validate match
|                  |    -> Return canonical external ID if found
|                  |
|                  |  Tier 2: External API search (TMDB/IMDB)
|                  |    -> meta_fetcher.search_multiple_results()
|                  |    -> 8-second timeout per search
|                  |    -> 87% minimum title similarity
|                  |    -> Creates new Media record if matched
|                  |    -> Returns canonical external ID
|                  |
|                  |  Concurrency: 8 parallel resolutions
+------------------+
       |
       v
+------------------+
| Build stream     |  TorrentStreamData with:
| objects           |  - info_hash, meta_id, filename, size
|                  |  - source="DMM Hashlist"
|                  |  - resolution, codec, quality, bit_depth
|                  |  - audio_formats, channels, HDR, languages
|                  |  - season/episode (for series)
|                  |  - release_group, boolean flags
+------------------+
       |
       v
+------------------+
| Batch store      |  crud.store_new_torrent_streams():
| in PostgreSQL    |  - Skip existing info_hashes
|                  |  - Resolve meta_id -> Media record
|                  |  - Create Stream + TorrentStream + StreamMediaLink
|                  |  - Create file entries for series
|                  |  - Link languages, audio formats, catalogs
+------------------+
```

---

## Incremental vs Backfill

The scraper maintains two "cursors" in Redis:

### Incremental (newest -> known)
- Fetches latest commits from GitHub
- Walks backwards until hitting the last-known commit SHA
- Processes in chronological order (oldest first)
- Updates the "latest known SHA" after processing

### Backfill (known -> oldest)
- Walks backwards from the oldest incremental commit
- Processes historical commits that predate the first run
- Stores progress as "next commit to process"
- Marks "__done__" when reaching the very first commit (no parents)

```
Newest commit  <-- Incremental starts here, walks back
     |
     v
Last known SHA  <-- Incremental stops here
     |
     v
Backfill cursor <-- Backfill starts here, walks back
     |
     v
First commit    <-- Backfill marks "__done__"
```

---

## Redis State Keys

| Key | Type | Purpose |
|-----|------|---------|
| `dmm_hashlist_scraper:latest_commit_sha` | String | Head commit SHA from last incremental run |
| `dmm_hashlist_scraper:backfill_next_commit_sha` | String | Next commit to process in backfill ("__done__" when finished) |
| `dmm_hashlist_scraper:processed_file_shas` | Set | Blob SHAs of all processed HTML files (prevents reprocessing) |

---

## Why Streams Might Not Appear

1. **Metadata resolution fails** — TMDB can't find the title (non-English titles, obscure content, mangled filenames). These are logged as "DMM entry NO MATCH".

2. **Already processed** — Blob SHA is in the processed set. The file is skipped entirely (no log unless debug).

3. **Duplicate info_hash** — Stream already exists in PostgreSQL from a previous run. `store_new_torrent_streams` skips it silently.

4. **Adult/sports filtered** — Entry matched adult keywords or sports broadcast patterns.

5. **GitHub rate limit** — Without auth token: 60 requests/hour. Each commit needs 1 request for the file list + 1 per HTML file. Exhausts quickly.

6. **8-second metadata timeout** — API search took too long. Entry dropped.

7. **Title similarity < 87%** — TMDB result didn't match the parsed title closely enough.

---

## Key Configuration

| Setting | Default | Current (.env) |
|---------|---------|----------------|
| `IS_SCRAP_FROM_DMM_HASHLIST` | False | True |
| `DMM_HASHLIST_REPO_OWNER` | debridmediamanager | debridmediamanager |
| `DMM_HASHLIST_REPO_NAME` | hashlists | hashlists |
| `DMM_HASHLIST_BRANCH` | main | main |
| `DMM_HASHLIST_SYNC_INTERVAL_HOUR` | 6 | 1 |
| `DMM_HASHLIST_COMMITS_PER_RUN` | 20 | 200 |
| `DMM_HASHLIST_BACKFILL_COMMITS_PER_RUN` | 20 | 200 |
| `DMM_HASHLIST_GITHUB_TOKEN` | None | Set (5000 req/hr) |
| `DMM_HASHLIST_SCRAPER_CRONTAB` | 0 * * * * | 0 * * * * |

---

## Monitoring

```bash
# Watch DMM scraper in real-time
journalctl -u mediafusion-taskiq-scrapy -f | grep -i dmm

# Check DMM store summaries (how many streams stored per batch)
journalctl -u mediafusion-taskiq-scrapy --since "1 hour ago" | grep "DMM store summary"

# Check metadata failures
journalctl -u mediafusion-taskiq-scrapy --since "1 hour ago" | grep "DMM entry NO MATCH" | wc -l

# Check Redis checkpoint state
redis-cli GET dmm_hashlist_scraper:latest_commit_sha
redis-cli GET dmm_hashlist_scraper:backfill_next_commit_sha
redis-cli SCARD dmm_hashlist_scraper:processed_file_shas
```
