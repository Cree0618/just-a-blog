---
date: 2026-02-11
title: "Search-mode tracking when most observations are negatives"
description: "Multi-camera association for Prague’s parking-validation cars: travel-time gates, CLIP in a subprocess, 11,180 crops, and no mAP. The failure mode is a confident false track."
tags:
  - computer vision
  - tracking
  - Prague
  - CLIP
  - multi-camera
---

City-scale multi-camera vehicle tracking (MTMC) is usually posed as a coverage problem: many cameras, overlapping views, a detector, a ReID model, then global association (Liu et al. 2021; Khorramshahi et al. 2022). The useful assumption in that literature is that the target actually appears. The Prague parking-validation fleet violates it.

City Sense runs a small set of hybrid Renault Capturs. The spec I built against — `overview.md` — is about twelve cars, public cameras with sparse coverage, most patrol time on side streets with no camera, and no ALPR. True positives are rare. Overview.md says the quiet part: even with thousands of crops you may have zero target cars. A tracker that always shows twelve identities is not conservative. It is inventing a fleet.

This post focuses on that constraint, and on a prototype (`ppc/`) that treats tracking as **search**: 0..N tracks, travel-time hard gates, CLIP ReID isolated in a subprocess, and an audit log of every association including the misses. Related work on road-graph particle filters, GCN tracklet association, and high-FPS overlapping cameras is in `papers.md`. Those systems are the right next model if the observations become reliable. They are not what the live worker runs.

I explicitly mention “search mode” because the layer between a crop and a dot on a map is as important as the detector. Hungarian assignment is not the hard part.

## The observation problem

“Twelve cars, forty cameras” is the problem statement in the README. The files are messier. `cameras_clean.txt` has 108 unique IDs with GPS. `cameras_raw.txt` has 59 image URLs. The generated registry `data/cameras.csv` is the intersection: **29 cameras** with both a URL and coordinates. A later JPEG sweep filled 109 camera folders. Default ingest polls **five** cameras every 30 seconds.

I did not “track forty cameras in production.” I wrote ~40 as the coverage I expected, then spent a long time learning which IDs returned an image.

Before `ppc/` existed, `parkauta-detect` was a fetch lab, not a product. Prague’s public camera endpoint often returns JSON with `contentBase64` instead of a JPEG. The later ingest worker (`ppc/ingest/poller.py`) does the same decode with User-Agent `prague-parking-cars/0.1`, a SHA-256 of the bytes, a JPEG on disk, and an `observations` row in SQLite. Cameras are staggered across the interval so five feeds at 30 seconds are not five simultaneous hits.

Once frames exist, three facts show up at once.

1. **Most vehicles are not the target.** YOLOv8n returns cars, vans, and buses. The object of interest is a specific small SUV with rooftop hardware, in a city full of small SUVs, with no plate.
2. **Coverage is sparse in time and space.** A car can spend minutes on a side street and cross one camera for a single 30-second snapshot. Association is not “track in the frame.” It is “this crop at camera A at t0 and that crop at camera B at t1 might be the same object if a car could drive there in t1 − t0.”
3. **A naive tracker teleports.** Treat every high-scoring crop as a hit and a white compact at I. P. Pavlova becomes the same track as a white compact at Barrandov one second later. Use the worker’s loop `dt` — time since the previous *observation event*, not since *this track* was last seen — and you get the opposite bug: a feasible five-minute hop is gated out because the last camera processed was one second ago.

`ppc/tracking/fleet.py` still has a `FleetConstraint` that tries to maintain `fleet_size=12`. That module is not what the live worker runs.

## Design patterns

Compared with “detector + ReID + always-on IDs,” the prototype is closer to a runtime: how the system observes, refuses, memorizes, and leaves an empty map when it should.

### Pattern 1: Detect vehicles, score targetness separately

Vision (`vision_v2`) drops a frame if it is too dark, too bright, or too blurry (mean brightness outside `[10, 245]`, Laplacian variance below 20), runs YOLOv8n for `{car, motorcycle, bus, truck}`, optionally filters by a per-camera ROI, embeds each crop, and writes detections.

The ROI file is `data/camera_roi.json`. It is `{}`. The loader and the point-in-polygon filter are real; `tests/test_roi.py` exercises them. The live config applies no mask.

ReID is CLIP when it works — `openai/clip-vit-base-patch32` (Radford et al. 2021), 512-D, L2-normalized. Importing `torch` and `transformers` has segfaulted at process start. The embedder therefore probes the import in a **subprocess** so a bad environment kills a child, not the worker.

```python
p = subprocess.run(
    [sys.executable, "-c", "import torch, transformers; print('ok')"],
    stdout=subprocess.DEVNULL,
    stderr=subprocess.DEVNULL,
    check=False,
)
if p.returncode != 0:
    self._deps_available = False
    return False
```

If CLIP is unavailable, the embedder hashes the crop bytes into a 512-D vector and keeps going. That fallback is plumbing, not ReID.

Scoring is dull on purpose. If a target classifier is present, `final_score = 0.7 * captur_prob + 0.3 * reid_sim`. Otherwise the score is similarity to reference embeddings. Below 0.35 the detection is dropped, unless `--store-all-detections` is on for the “pick a random car on the map” debug mode. No trained classifier and no reference images means `final_score` is ~0 and the worker emits nothing. I would rather store zero rows than a city’s worth of Hondas.

The classifier I have without weights is a heuristic stub in `CapturClassifier`: color stats, texture, edges. The docstring calls it a placeholder. The starting positives were news photos. Five refined references live in `target_cars/target_cars_refined/`. The default `data/reference/captur/` directory is empty, so perception had to work as retrieval first.

### Pattern 2: Belief over camera nodes, not a forced fleet

The live worker imports one tracker:

```python
from ppc.tracking.tracker import FleetTracker
```

`FleetTracker` keeps an appearance prototype and a distribution over camera IDs. On every observation it propagates that belief with a travel-time likelihood. Thirty-five percent of the mass stays put (parking, lights, a queue). The rest goes to cameras whose travel time is at most `dt + slack`. The likelihood is a Gaussian around elapsed time — sigma 30 seconds, slack 10 seconds — and only from the top three belief sources, so the distribution does not smear into uniform.

Travel time is an OSRM matrix if one has been loaded into SQLite. Otherwise it is haversine over 22 km/h. The prototype database I have locally has **zero** rows in `camera_travel_times`. A local run without a precompute is the haversine path.

`PPC_TRACK_MAX` defaults to 12. That is a cap on births, not a target. Unmatched detections only birth a track if `final_score >= 0.80`. Two hits inside 120 seconds confirm the track. Tentative tracks die quickly. Confirmed tracks can still die if existence probability collapses.

### Pattern 3: Physics as a hard gate, coasts as first-class outcomes

Association is a rectangular assignment with dummy **coast** columns, so several tracks can miss the same camera without stealing detections. `solve_assignments` is a thin wrapper around SciPy’s Hungarian solver (Kuhn 1955). Impossible cells get `big_m = 1e6` and are dropped. Coast cost is 1.0.

The hard gate is the point of the project:

```python
if dt_s > 0.0 and track.belief:
    tt_min = self._min_travel_time_from_belief(track, det_camera_id)
    if tt_min is not None and float(tt_min) > float(dt_s) + float(self._reach_slack_s):
        return float(big_m)
```

`dt_s` is **not** the worker loop delta. `_dt_track_s` uses time since that track’s last assigned hit. A regression test allows a 60-second hop after a 120-second gap even if the previous observation in the queue was one second ago. Another test forces a coast when travel time is 1,000 seconds and elapsed time is 1.

A hit blends 20% of the new embedding into the prototype, spikes belief at the camera, and writes a row. A miss applies negative evidence: downweight that camera, and decay existence using the belief mass *before* the downweight — order dependence kept false tracks alive; that is also a test.

`track_assignments` stores the detection id or NULL for a coast, plus cost, detector score, appearance similarity, existence probability, hits, misses, and whether the track is confirmed. If a track looks stupid on the map, the row is the artifact, not folklore.

Busy frames are capped at 80 detections, keeping each track’s top-K by sampled cosine so a 0.10-score match is not discarded behind fifty 0.95-score strangers.

{{< responsive-image src="images/prague-pipeline.png" alt="Ingest, vision, FleetTracker, SQLite, map pipeline" maxWidth="800px" >}}

*The live worker is FleetTracker. Experimental MHT and road-graph modules exist and are not imported.*


## Case study: what the prototype actually contains

What I can stand behind:

- `data/datasets/yolo_cropped_v1/meta.json` records `images_seen: 11180` and `rows_written: 11180`, across 109 camera IDs. The JPEGs currently sit in `data/yolo/raw_frames/`. The path stored in `meta.json` (`yolo_cropped/raw_frames`) is no longer on disk.
- CLIP mining against the five refined references produced a 500-row `candidates_by_ref.csv` and a 500-image review folder. The top row has cosine 0.762 versus `maxresdefault.jpg` at camera 500234. That is a retrieval rank, not precision at 1. I have not labeled the 500.
- 29 `test_*.py` files, 94 `def test_` functions. The ones that carry this story are FleetTracker lifecycle, per-track gating, negative-evidence order, assignment uniqueness, detection capping, ROI geometry, and persistence of a dead track’s last belief.
- The map is a Leaflet page in `ppc/api/server.py`. “Random Track” seeds an `adhoc:` session from a random stored crop — how I test association when no Captur is in view. A Chinese-postman planner can cover a drawn polygon. That planner is a separate module. It is not the vehicle tracker.

{{< responsive-image src="images/prague-cameras.png" alt="Twenty-nine Prague camera positions from cameras.csv" maxWidth="720px" >}}

*29 cameras with both a URL and coordinates. Not a street map — the graph the tracker believes over.*


What I cannot stand behind:

- **There is no measured precision, recall, or mAP.** `evaluation.md` is titled “Evaluation Plan.” It lists metrics I should compute — false-track rate on target-free data, time-to-die, time-to-confirm, candidate precision@K. It does not contain a results table.
- `ppc/sim/evaluate.py` prints MOTA 0.85 and IDF1 0.82. Those are hardcoded placeholders on the experimental `IntegratedTracker`. The unit test only checks that the number is between 0 and 1.
- I do not claim the live system maintains twelve tracks. A local prototype DB I inspected had 2 ad-hoc tracks, 40 dead, and 0 active. That is a snapshot, not a score.

{{< responsive-image src="images/prague-reference-capturs.png" alt="Five news-photo reference crops of City Sense Renault Capturs" maxWidth="800px" >}}

*The starting positives were public news photos. I am not publishing unlabeled CCTV review crops here.*


## What did not ship onto the worker

**Forcing a fleet of twelve.** `IntegratedTracker` plus `FleetConstraint` will create tracks to fill `fleet_size`. Grep the live worker for `mht.py`, `road_tracker.py`, or `integrated_tracker.py` and you will not find them. They have tests. They are not imported.

**A road-graph particle filter as the first tracker.** `algorithms.md` lists it as future work. Shipping it first would have been a research costume on missing labels. The production state is a belief over cameras. `docs/tracking_architecture.md` says that out loud.

**Empty ROIs and a missing reliability file.** `data/camera_reliability.json` does not exist, so every camera uses a default reliability of 0.5. The code path is tested. The config is not done.

**Hash embeddings as ReID.** They keep the schema happy. They do not retrieve a rooftop light bar.

## Implications

Rare-event tracking is a search problem. The failure mode is not a missed Captur. It is a confident false track that lives for twenty minutes and teaches you nothing.

Physics is a better first prior than appearance. CLIP is useful for mining a review folder from five news photos. It is not allowed, by itself, to jump a car across the river in one second. The hard gate is one `if`. The important part is *which* `dt` you pass it.

If the import can kill the process, import in a child. Persist the coasts. Write the evaluation file *as a plan* until you have the measurement.

The next useful work is not another tracker module. It is labeling the 500-image review set, filling `camera_roi.json`, computing the target-free false-track rate, and only then deciding whether a road-graph model has anything to eat. The prototype is a loop I can run, inspect, and disagree with. It is not a claim that twelve parking cars are on the map.
