---
layout: post
title:  "Reviving Percy — from dusty scripts to a Mars 2020 playground"
date:   2026-08-08 22:00:00 -0700
categories: astronomy opensource mars
author: saketharamanujam
comments: true
---

<figure class="d-flex justify-content-center">
  <img src="/assets/img/percy-revival/02-stop-cameras.png" width="100%" alt="Percy Metadata Playground — stop camera view with photo world and rover model">
  <figcaption style="font-size:0.85rem;color:#8b9bb0;margin-top:0.4em">
    Body-frame photo world at a real (site, drive) stop — image planes at true CAHVOR poses, map inset, stereo pairs, the works.
  </figcaption>
</figure>

A few years ago I wrote a small pile of scripts so I wouldn’t have to click through NASA’s raw-image pages like a medieval monk. The idea was simple: **pull Perseverance (Mars 2020) image metadata, keep a catalogue, publish it somewhere people could actually use it**. That somewhere became [Kaggle](https://www.kaggle.com/datasets/sakethramanujam/mars2020imagecatalogue). The code sat under names that still embarrass me slightly (`percy-metadata-download-automator` is a mouthful and a half), and then life, jobs, and the usual “I’ll refactor this later” pile took over.

This summer I opened that drawer again.

What came out is not just “scripts that still run.” It’s a full **exploration engine** for half a million images, with a 3D playground that finally makes the catalogue feel like standing next to the rover instead of staring at a spreadsheet.

Repo (public): [sakethramanujam/percy-metadata-download-automator](https://github.com/sakethramanujam/percy-metadata-download-automator)

---

### How it started

Landing day for Perseverance was pure dopamine for anyone who grew up on Pathfinder GIFs and Opportunity sol summaries. NASA’s raw image site is a goldmine — Navcams, Mastcams, Hazcams, pose models, sols, sites, drives — but the UX is “browse one product at a time and hope your tabs survive.”

So v0 of this project was pragmatic and a little scrappy:

- scrape / page through the public NASA metadata feed  
- dump rows into a CSV  
- occasionally ship a version to Kaggle  

It was never meant to be a product. It was a **personal antenna** aimed at Jezero: keep the metadata close, keep it current, let other people download without reinventing the crawler.

That catalogue quietly did its job. Hundreds of downloads on Kaggle later, the repo itself mostly collected dust.

<figure class="d-flex justify-content-center">
  <img src="/assets/img/percy-revival/01-mission-path.png" width="100%" alt="Mission path view with MMGIS traverse">
  <figcaption style="font-size:0.85rem;color:#8b9bb0;margin-top:0.4em">
    Mission path mode: MMGIS Jezero traverse, stop list ordered by sol, basemap hooks for CTX / HiRISE / HRSC.
  </figcaption>
</figure>

---

### Why revive it?

Two reasons, both stubborn:

1. **The data kept growing.** Percy didn’t stop driving. Metadata didn’t stop landing. A seed from Kaggle plus a half-finished catch-up is how catalogues go stale and guilt accumulates.  
2. **CSV is not a way to feel Mars.** Once you have poses — real CAHVOR look vectors, sites and drives, stereo-capable pairs — leaving them in columns feels like owning a telescope and only reading the serial number.

So the revival had two halves: **finish the automation honestly**, and **build something you can walk around in**.

---

### Half one: the pipeline, un-janked

The modern CLI (`scripts/metadata.py`) is deliberately boring in the good way:

- seed from Kaggle when you can  
- catch up from NASA’s public API when you must  
- soft-fail flaky pages, checkpoint, resume  
- merge patches without heroic one-liners  
- publish a new Kaggle version when the local catalogue actually moves  

After the latest full catch-up pass:

- **~559k unique imageids** in `full-metadata.csv`  
- NASA listing total around **~1.02M** (listings ≠ unique products; the “behind by 0” state means we’ve walked the feed)  
- playground index rebuilt with **~526k posed** images, **1217 stops**, sols **0–1941**  
- MMGIS waypoints joined for map mode (~44 km of traverse story so far)

Catalogue lives on disk and on Kaggle — **not** in git. We even scrubbed old sample CSVs and machine paths out of history so the public repo doesn’t leak laptop archaeology.

<figure class="d-flex justify-content-center">
  <img src="/assets/img/percy-revival/04-equirect-pano.jpg" width="100%" alt="Pose + feature refined equirectangular panorama">
  <figcaption style="font-size:0.85rem;color:#8b9bb0;margin-top:0.4em">
    Equirect stitch for a stop: body-frame poses first, then hybrid residual align (ECC / phase / SIFT) and exposure match. Not magic — better seams than pure pose splat.
  </figcaption>
</figure>

---

### Half two: the playground gets a spine

The fun part is the **local playground** under `playground/`:

- **FastAPI** over a Parquet index  
- **Vite + React + Three.js** for the UI  

Modes that actually map to how mission people talk:

| Mode | What you get |
|------|----------------|
| **Path** | Jezero traverse from MMGIS, orbital basemap (FU Berlin WMS), click / double-click stops |
| **Stop** | Rover body frame, GLB model at origin, photo-world planes, rays, stereo pairs, depth → point cloud |
| **Site** | Multiple drives at one site in shared EN frame |
| **Photo sphere** | Equirect stitch → full-screen pan (3D sphere optional) |
| **Tour** | Sol-ordered guided walk with shareable deep links |

Coordinate systems were the character development arc of this rebuild. Map frame is easting/northing. Stop frame is rover body (+X forward, +Y right, +Z down) mapped into the official Perseverance GLB axes. Mixing them is how basemaps “vanish” when you open a stop — so the map stays as a **corner inset** instead of a lie in the 3D scene. That was a design choice, not a bug (even when it felt like one).

Stereo depth can run on GPU when CUDA is around; point clouds land in the same body frame as the cameras. Coverage heatmaps show az×el occupancy. Stitch align is a UI switch: **pose only** vs **hybrid** (pose layout + feature residual + exposure).

<figure class="d-flex justify-content-center">
  <img src="/assets/img/percy-revival/03-site-world.png" width="100%" alt="Site multi-drive world">
  <figcaption style="font-size:0.85rem;color:#8b9bb0;margin-top:0.4em">
    Site world: stack drives at one site in map EN so outcrops can meet each other across sols.
  </figcaption>
</figure>

---

### What I deliberately did *not* do

I did **not** animate the mast and arm to fake every pointing. The official GLB stays in rest pose. Rays and photo planes use true image poses; when many mast pointings pile up, the cloud around the static head is honest physics meeting a static mesh. That’s the trade.

I also did **not** pretend pure feature stitching alone is the one true path for Navcam sets. Hybrid means: trust the metadata for the sphere, then let ECC/SIFT clean residuals where overlap exists.

---

### How to poke it

```bash
git clone git@github.com:sakethramanujam/percy-metadata-download-automator.git
cd percy-metadata-download-automator
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# data
python scripts/metadata.py init --from-kaggle   # or update if you already have a seed
python -m playground.pipeline.build_index
python -m playground.pipeline.fetch_mmgis

# api + web
uvicorn playground.api.main:app --host 0.0.0.0 --port 8000
cd playground/web && npm install && npm run dev -- --host 0.0.0.0 --port 5173
```

Deep links work, because shareable state is half the fun:

```text
?view=stop&site=4&drive=136&image=<imageid>
?view=path
?tour=mission-highlights&step=0
```

Kaggle catalogue (the original public face of this project):  
[mars2020imagecatalogue](https://www.kaggle.com/datasets/sakethramanujam/mars2020imagecatalogue)

---

### Closing the loop

v1 was “don’t lose the metadata.”  
v2 is “don’t lose the *feeling* of the metadata.”

Same mission, same public NASA feed, same stubborn desire to keep a personal mirror of Perseverance’s eyes. New skin: path, stop, site, sphere, stereo cloud, guided tour — and a repo that’s public without smuggling laptop paths into git history.

If you download the Kaggle set, open an issue, or just wander a stop on a weekend: hi. Bring coffee. Jezero is still weird and wonderful.

— Saketh
