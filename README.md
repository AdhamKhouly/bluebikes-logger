# Bluebikes availability logger

Records how many bikes and open docks every Bluebikes station has,
every ~10 minutes, into daily CSV files. This is the "recorder" for our
project — it builds the ground-truth dataset we validate against.

**Every day it isn't running is validation data we can never get back,
so set this up first.**

## Setup (one person does this once, ~10 minutes)

1. Create a **new GitHub repository** (public or private, either works).
   Name it something like `bluebikes-logger`.

2. Upload everything in this folder to the repo, keeping the structure:
   ```
   logger.py
   README.md
   .github/workflows/log.yml     <- the folder names matter exactly
   data/                          <- can start empty
   ```
   Easiest way: on the repo page click "Add file" -> "Upload files" and
   drag the whole folder in. Make sure `.github/workflows/log.yml` ends up
   at that exact path (GitHub's uploader preserves folders if you drag
   the folders themselves).

3. Turn on write permission for workflows:
   repo **Settings -> Actions -> General -> Workflow permissions ->
   "Read and write permissions"** -> Save.

4. Start it: go to the **Actions** tab -> "Log Bluebikes station status"
   -> **Run workflow**. (If Actions shows a "enable workflows" button
   first, click that.) After ~1 minute, a new CSV should appear in `data/`.

That's it. GitHub now runs the logger on its own servers every ~10-20
minutes, forever, for free. Your laptop can be off.

## What you get

One CSV per day in `data/`, named `station_status_YYYY-MM-DD.csv`.
One row = one station at one moment (~600 stations x ~100-140
snapshots/day ≈ 60-85k rows/day, ~5 MB/day).

| column | meaning |
|---|---|
| snapshot_utc | when the feed was last updated (UTC) |
| fetched_utc | when we fetched it (UTC) |
| station_id | station's GBFS id |
| legacy_id | station's old numeric id (joins to some datasets) |
| num_bikes_available | rentable bikes right now (includes e-bikes) |
| num_ebikes_available | of those, how many are e-bikes |
| num_bikes_disabled | broken/locked bikes |
| num_docks_available | open docks right now |
| num_docks_disabled | broken docks |
| is_installed / is_renting / is_returning | 1 = station active for that function |
| last_reported | unix time the station last phoned home |

## Definitions we'll use in analysis

- **Empty (stockout):** `num_bikes_available == 0` and `is_renting == 1`
- **Full (dockout):** `num_docks_available == 0` and `is_returning == 1`
- Ignore rows where `is_installed == 0` (station removed/offline).
- Timestamps are UTC; Boston is UTC-4 (EDT). Convert before comparing
  to trip data, which is in local time.

## Loading it later

```python
import pandas as pd, glob
df = pd.concat(pd.read_csv(f) for f in sorted(glob.glob("data/*.csv")))
df["snapshot_utc"] = pd.to_datetime(df["snapshot_utc"])
```

## Notes

- GitHub's scheduler is best-effort: expect ~10-20 min between snapshots,
  occasionally longer. That's fine — shortage windows last much longer
  than 20 minutes.
- To test locally instead: `pip install requests` then `python logger.py`
  (appends one snapshot to `data/`).
- To stop it: disable the workflow in the Actions tab.
