# Dopplerganger — acoustic geolocation from aircraft Doppler curves and ADS-B data

**Role:** sole researcher. I set the method, the go/no-go gates, the evaluation rules and every keep/revert
decision. Most of the code was written with AI coding assistants under my review.
**Duration:** November 2026, ongoing
**Stack:** Python · NumPy · SciPy · librosa · DuckDB · Parquet · pyproj/rasterio · pytest + hypothesis
**Code:** [github.com/bulevill/dopplerganger](https://github.com/bulevill/dopplerganger)

---

## Summary

A pipeline that estimates where an audio recording was made. It detects the Doppler curves of small aircraft
in the audio, matches them against a day of US ADS-B flight data and terrain, and outputs a probability heat
map with credible regions. On realistic renders built from real flight tracks around New York City and the
Hudson Valley it locates 19 of 24 runs, with a median error of about 100–300 m. It has not located any of
four real field recordings. Separately measured: 0.8% of US land gives a 30-minute recording a ≥ 50% chance
of containing the two aircraft passes the method needs.

---

## What I built

| Layer | What it does |
|---|---|
| **Detection (M1)** | Finds aircraft Doppler curves in audio and estimates time, speed and closest distance with uncertainties. Whole-curve matched filter |
| **Flight data (M2)** | Converts a raw adsb.lol day (3.8 GB, 75k aircraft) into Parquet tracks with acoustic class (prop, turboprop, rotor, jet) |
| **Terrain** | Ground height above the WGS84 ellipsoid from Copernicus DEM + EGM2008 geoid, consistent with ADS-B GPS altitudes |
| **Physics (M4)** | Forward model integrating from emission time to arrival time, with wind. < 0.1 Hz vs. closed form |
| **Search (M3–M6)** | Reduction funnel from grid cells to flight-pair hypotheses, spectrogram scoring, posterior heat map, credible regions |
| **Evaluation** | Gate drivers, a labelled detector benchmark with a held-out real set, a scale benchmark, and a national coverage map |

**Main design decision:** the search reduces to flight-pair hypotheses before any audio scoring. Scoring every
grid cell against the spectrogram would take on the order of billions of physics evaluations for one
country-day. Geometry and timing gates (distance, speed, time gap between events, expected silence) are cheap
and remove most candidates. Neighbouring cells share the same flights, so scoring is done once per hypothesis
and spatial refinement only inside the top hypotheses.

---

## Results

| Test | Result |
|---|---|
| Physics vs. closed form | < 0.1 Hz; speed, f₀, distance within 2% |
| Synthetic end-to-end | 8/8 located, 2–13 m |
| Realistic renders from real flight tracks (MP3, wind, sound-speed mismatch, ADS-B clock error) | 19/24 (previous detector: 13/24) |
| Detector on real phone flyovers, held out, run once | 22/31 usable (previous detector: 5/31) |
| Field recordings | 0 of 4 located. Three contain no aircraft pass; the fourth contains one pass, missed by the detector version in use at the time |
| National coverage | 0.8% of US land for a 30-min recording at ≥ 50% probability |

Success criteria set at the start: median error ≤ 500 m on real recordings, and calibrated credible regions.
Neither has been measured, because no real recording has been located.

The source technique's claim of sub-metre precision is not supported: ADS-B position error, 1 s timestamps and
sound-speed uncertainty give a floor of about 25–100 m.

---

## Detector evaluation and rewrite

1. **Finding.** End-to-end tests showed detection was the bottleneck: when two close passes were detected,
   the search located the recording in 5 of 6 runs. No single fit tolerance worked: 0.5 Hz rejected real
   passes from irregular flight tracks, and 2 Hz produced 6–24 false events per recording in New York airspace.
   A two-tier detector (tight fits plus gated loose fits) tuned on the 24 rendered recordings raised the
   end-to-end result from 11 to 13 of 24.

2. **Held-out test.** Those gates had been tuned and scored on the same 24 recordings. I built a benchmark with
   labelled tuning sets and a held-out set of 40 real phone recordings of flyovers from a public dataset, to be
   run once with frozen settings. The two-tier detector found 8 of 40, including failures on passes under 2 km.

3. **Diagnosis.** In 37 of the 40 clips a spectral ridge covering the true pass existed and was rejected by the
   gates. Close, fast passes smear within the one-second analysis window, which degrades the fit, and the clips
   were shorter than the gates' minimum ridge length. The gates were fitted to properties of the synthetic renders.

4. **Change.** The per-frame ridge tracker was replaced with a whole-curve matched filter that scores the full
   Doppler template, so smeared or partly masked passes still accumulate evidence.

5. **Outcome.** Held out, run once: 22 of 31 usable detections, against 5 (31 clips had reliable labels).
   End to end: 13/24 → 19/24 at the same false-positive rate (3.7 vs. 3.4 per hour). A follow-up change that
   searched for aircraft hidden under another pass's harmonics found 6 more true passes but roughly doubled
   false positives and lowered the end-to-end result to 18/24. It was reverted.

---

## Engineering problems

- **Distance bias from turning aircraft.** About 1 in 6 detected distances were 2× or more too close. Fitting
  the straight-pass model to each real track's noise-free Doppler curve showed the detector agreed with the
  sound to 5.5% (median). Turning or accelerating aircraft, mostly helicopters, produce curves that fit a
  closer straight pass. The fix belongs in the search: compare whole curves instead of closest-approach distances.
- **Density limit.** At 5× New York traffic density, the true location dropped out of a fixed top-20 scoring
  budget. Peak memory was about 0.8 GB, so raising size caps would not have helped. The scoring budget now
  scales with the number of candidates.
- **Truncation accounting.** The search report records every cap hit, dropped candidate and refinement error,
  so capacity limits appear as counts rather than as unexplained accuracy loss.
- **Datums and time bases.** ADS-B GPS altitudes are ellipsoidal and DEM heights geoid-based (about 30 m apart in
  the US), and the recording and flight-data clocks are separate time bases. Each is several hundred metres of
  error if ignored.

---

## What I'd do differently

- **Measure national coverage first.** The first field recordings were made indoors where the nearest small
  aircraft was 12–16 km away. The coverage map would have ruled that location out in advance.
- **Build the held-out benchmark before the first tuning pass.** It would have ruled out the two-tier detector
  directly.
- **Use real background noise in renders from the start.** Clean-noise renders overstated detector performance;
  a single noise burst splitting the curve caused the miss on the one real pass.
- **Compare whole curves in the search from the start.** Closest-approach distance discards information that
  turning aircraft make necessary.

---

## Scope

Personal research prototype, about one week of work at the time of writing, built with AI coding assistants.
Results are on realistic simulations from real flight tracks, plus a public set of real flyover recordings for
the detector. Real-world localisation has not been demonstrated. Development and testing used only synthetic
audio, public research data and my own recordings.
