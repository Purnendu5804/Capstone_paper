# Notes to Authors

## 1. Conflicts and Gaps Found in `Capstone.pdf` (and vs. the Implemented Repo)

1. **Major Framing Conflict (Pages 1--3 vs. Pages 4--30):**
   * Pages 1--2 frame the paper around *cloud/ground-side satellite image processing* ("Your gap vs. the papers: TargetFuse, Kodan and EarthSight mostly target onboard compute and downlink limits. You're measuring the ground/cloud side, with a real containerized pipeline and open data").
   * Pages 4--30 pivot completely to an *emulated onboard satellite pipeline* (resource-capped container + simulated downlink) in response to supervisor feedback.
   * **Action:** `paper.tex` strictly follows the Capstone Team Brief (emulated onboard) framing and ignores Pages 1--2. Ensure no team member copies text from Pages 1--2 into the draft.

2. **Tile Array Shape and Dtype Mismatch (PDF vs. Repo):**
   * Ashutosh's roadmap (p. 22) suggests passing tiles as `(4, H, W)` floats after reflectance scaling.
   * The implemented repo uses immutable `(H, W, C)` `uint16` arrays with band names `(B02, B03, B04, B08)`.
   * **Action:** Vaibhav (tiler) and Ashutosh (S3 features / heavy model) must standardize on the repo's `(H, W, C)` `uint16` contract—or agree on a single conversion wrapper—before integration.

3. **Scorer Output Range Mismatch (PDF vs. Repo):**
   * The PDF (pp. 17, 18, 21, 24) specifies `score(tile) -> float in [0, 1]` and suggests percentile-rank normalization on training scenes.
   * The implemented repo specifies `score(tile) -> float` (any real number, comparable only within one scorer, since only ranking matters).
   * **Action:** `paper.tex` describes the implemented contract. Note that if Policy P2 (score-and-skip with a threshold) is added later, thresholds will need to be defined in each scorer's native scale (or calibrated per scorer on validation scenes).

4. **Sentinel-2 Reflectance Scaling Offset and S2 Threshold Calibration:**
   * Page 21 warns that newer Sentinel-2 Level-2A processing baselines add a radiometric offset (e.g., `-1000` [VERIFY, not in Capstone.pdf] in ESA Baseline 04.00+ [VERIFY, not in Capstone.pdf]) in addition to the `/10000` scaling factor.
   * An additive offset does not affect S1 (variance) or S2 (Sobel gradients), only the scale divisor and S3's NDVI/NDWI. The implemented S2 scorer divides raw `uint16` values by `10000`, and its default threshold `0.02` is uncalibrated.
   * **Action:** Check the metadata of the downloaded Sentinel-2 scenes, apply the offset consistently for S3's NDVI/NDWI (and any band-level features), and calibrate S2's threshold on validation scenes.

5. **Scene Size Ambiguity ($60\times60$ km vs. $110\times110$ km):**
   * The PDF alternates between referring to a "$60\times60$ km image" ($6{,}000\times6{,}000$ px, ~576 tiles at 256 px; pp. 4, 5, 10, 28) and standard Sentinel-2 granules of "$110\times110$ km" ($10{,}980\times10{,}980$ px, ~1,850 tiles at 256 px; p. 28).
   * **Action:** Decide whether Vaibhav's streaming tiler processes full $110\times110$ km granules or crops a $60\times60$ km sub-window, and state the exact choice in Section III of `paper.tex`.

6. **Policies P0--P3 vs. Currently Implemented Runner:**
   * The PDF defines P0 (bent pipe), P1 (process-everything onboard in capture order), P2 (score-and-skip), and P3 (priority processing + downlink), plus Oracle.
   * The implemented repo currently runs scorer-driven ordering plus `random` and `FIFO` baselines through the same runner, without heavy-model execution or threshold skipping yet.
   * **Action:** Purnendu needs to map `FIFO` cleanly to P0 (no heavy model, raw tile bytes) and add the heavy-model compute-budget worker for P1/P2/P3.

7. **Head-of-Line Blocking in Downlink Simulator:**
   * The implemented downlink simulator stops a contact pass as soon as the top-ranked tile does not fit in the remaining capacity. When all tiles are raw and identical in byte size, this is benign; however, if P3 mixes large raw tiles with small inference-result payloads in the same queue, head-of-line blocking could underutilize the tail of a pass.
   * **Action:** Documented in Section VIII (Discussion) of `paper.tex`. Decide whether to keep strict head-of-line blocking or note it as a deliberate design choice.

8. **Unverified Citations, Numbers, and Recent Venue (EarthSight MLSys 2026):**
   * Kodan and EarthSight lack author lists in the PDF; OEC survey and FOOL lack exact titles; TargetFuse's 3.4$\times$ / 9.6$\times$ numbers, PhiSat-1's ~30% and 100$\times$ vs. 3--5$\times$ figures, and ESA WorldCover's 74.4% accuracy are all unverified against primary sources.
   * **Action:** Vaibhav must pull full texts and publisher BibTeX for every entry in `refs.bib` and verify the research-gap statement against TargetFuse, Kodan, and EarthSight before submission.

9. **Unresolved Day 1 Technical Decisions:**
   * Heavy model checkpoint and task (note: EuroSAT models typically expect $64\times64$ inputs, which may mismatch a $256\times256$ tile size unless resized or sub-patched).
   * Frozen mission rule (`MISSION.md`) and prevalence of $V$.
   * Exact Docker CPU/RAM caps and downlink budget sweep values.

## 2. Five Questions to Ask Your Supervisor

1. **Clarifying "Tile Before Capture":**
   > "When you mentioned tiling before capture, do you mean tiling the sensor stream onboard right after sensing—before storage, inference, and downlink—using windowed reads so the full scene is never held in memory, or do you mean sensor-level selective acquisition? We can build and evaluate onboard stream tiling in a resource-capped container, whereas sensor-level selective acquisition requires hardware/mission-planning control we do not have; is treating sensor-level acquisition as future work acceptable?"

2. **Acceptability of Emulation and Replayed L2A Data:**
   > "Because we do not have flight hardware or raw onboard sensor streams, we plan to emulate the onboard computer using a CPU/RAM-capped Docker container and replay processed Sentinel-2 Level-2A granules as a stream. Will this emulation setup—stated transparently in the limitations—meet your expectations for a systems paper?"

3. **Fidelity-to-Reference Framing for Valuable Tiles ($V$):**
   > "Rather than claiming ground-truth accuracy, we plan to define the valuable tile set $V$ by running an unconstrained pretrained land-cover model on every tile under a frozen mission rule (e.g., urban/industrial classes), so our metrics measure fidelity to full processing. Does this framing work for you, or would you prefer we also cross-check against an external land-cover map such as ESA WorldCover?"

4. **Heavy Model Task and Tile-Size Alignment:**
   > "At 10 m per pixel (Sentinel-2), small-object detection is poorly suited, so we plan to use tile-level land-cover classification (e.g., a EuroSAT-pretrained model). If the pretrained model expects $64\times64$ inputs while our tiles are $256\times256$, do you have a preferred resolution strategy (matching tile size to $64\times64$, resizing, or aggregating sub-patches)?"

5. **Scope Priorities Under the 5--6 Day Timeline:**
   > "Given our tight schedule, our core pipeline simulator and non-ML scorers (S1/S2) are already implemented and tested, while the streaming tiler, Docker caps, S3 ML scorer, and heavy-model runs are still ahead. If time tightens, which deliverable matters most to you: a complete E1--E3 comparison using S1/S2 plus a simple logistic-regression S3 on ~20 scenes, or a larger scene count with fewer ablations?"