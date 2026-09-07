# two-clock-audit

**When laboratory results become visible: a two-clock audit of ICU deterioration alerts.**

Every laboratory result carries two clinically distinct times. One is when the specimen was
collected. The other is when the result was released into the record and became visible to staff.
Almost all published intensive care prediction work uses the first. Deployed clinical decision
support can act only on the second.

This repository contains the full analysis behind a study that measures the interval between those
two timestamps across 15.9 million laboratory results, quantifies how much of the laboratory record
is actually visible when a model scores, and tests whether alert timing and detection change when a
deterioration model is restricted to results that had genuinely been released.

We call the two timestamps the **observation clock** and the **availability clock**, and the
comparison between them a **two-clock audit**.

---

## Key findings

| | Result |
|---|---|
| Median result availability latency | 52 minutes |
| By turnaround class | 3 min point-of-care, 60 min core laboratory, 72 min longer-turnaround |
| Record invisible at scoring hour 6 | 9.0% of collected results |
| Median alert lead time lost | 0.00 h (95% CI 0.00–0.00), both cohorts |
| Detections lost, external cohort | 2.00% of event stays (95% CI 1.66–2.36) |
| Median culture collection-to-result | 57.8 h (blood cultures 137.4 h) |
| Cultures resulting **after** the escalation they preceded | 88.6% of stays |

**The takeaway.** Exposure to result availability latency is governed by whether an input's
turnaround exceeds the interval at which the model runs. Against an hourly scoring interval, 99.8%
of point-of-care and 53.2% of core laboratory results clear that bar, so routine chemistry costs an
hourly-scored alert essentially nothing. Microbiology falls entirely outside it, and there the
result routinely arrives after the decision it was meant to inform.

That condition can be checked by any team before deployment using timestamps their laboratory
information system already holds.

---

## Repository structure

```
two-clock-audit/
├── notebooks/
│   ├── Part1_Result_Availability_Audit.ipynb   Latency measurement, cohorts, visibility
│   ├── Part2_Two_Clock_Scoring.ipynb           Escalation events, two-clock features, models
│   └── Part3_Lead_Time_And_Detection.ipynb     Alert timing, detection, burden, net benefit
├── outputs/
│   ├── figures/                                 15 figures (PNG)
│   ├── tables/                                  45 result tables (CSV)
│   └── manifests/                               Hash-chained provenance records
├── requirements.txt
├── .gitignore
├── LICENSE
└── README.md
```

No patient-level data is included in this repository, and none may be added to it. See
**Data access** below.

---

## Data access

The analysis uses two publicly available deidentified databases distributed by PhysioNet under
credentialed access:

- **MIMIC-IV v2.2** — https://physionet.org/content/mimiciv/2.2/
- **eICU-CRD v2.0** — https://physionet.org/content/eicu-crd/2.0/

Access requires PhysioNet credentialing, completion of CITI "Data or Specimens Only Research"
training, and acceptance of the relevant data use agreement. Neither database is redistributed here
in any form, including derived record-level files.

Once you have the data, point the notebooks at it by editing the three path fields in the `Config`
cell near the top of each notebook:

```python
mimic_root: str = r"/path/to/mimic-iv-2.2"
eicu_root:  str = r"/path/to/eicu-collaborative-research-database-2.0"
out_root:   str = r"/path/to/output"
```

The path resolver handles the flat layout, the nested `<name>.csv/<name>.csv` layout produced by
some download tools, and gzipped variants, so no restructuring of the download is needed.

---

## Requirements

Python 3.10 or later. Install with:

```bash
pip install -r requirements.txt
```

**Install DuckDB.** It is listed as a dependency and the notebooks will fall back to chunked pandas
without it, but that fallback is roughly an order of magnitude slower on the two large scans
(`labevents.csv` at ~13 GB and `chartevents.csv`, larger still). Both engines were verified to
produce identical results.

Approximate runtimes on a consumer laptop (Intel i7, 16 GB RAM, SSD), first run:

| Notebook | With DuckDB | Cached re-run |
|---|---|---|
| Part 1 | 10–25 min | under 1 min |
| Part 2 | 25–60 min | 3–5 min |
| Part 3 | 5–15 min | 5–15 min |

Every expensive stage caches to Parquet and is skipped on re-run unless `force_rebuild = True`.

---

## Running the analysis

Run the notebooks in order. Part 2 depends on Part 1's cached cohorts and laboratory extraction;
Part 3 depends on Part 2's cached predictions.

**Part 1** measures the collection-to-release interval, builds the analytic cohorts, and computes
record visibility. It ends with six blocking checks, the most important of which is an internal
control: point-of-care results must post far faster than central chemistry within the same patients
and the same timestamp field. If that check fails, the timestamp is measuring something clerical and
the study stops there.

**Part 2** defines the clinical endpoint (first documented escalation of care: vasopressor
initiation, invasive ventilation, or renal replacement therapy), builds the hourly scoring grid, and
constructs the same laboratory features twice, once under each clock. Two gradient-boosted models
are fitted and each is scored under both clocks, giving a 2×2 design. Early stopping is evaluated
against a stay-disjoint validation era rather than a random row split, because hourly rows within a
stay are strongly correlated and a random split lets the model memorize across the boundary.

**Part 3** simulates alerting with a suppression window, calibrates thresholds on the observation
clock and applies them unchanged to the availability clock, and reports lead time, detection status,
alert burden, number needed to alert, and net benefit. It also contains the microbiology analysis,
which requires no model at all.

---

## Reproducibility

Every source file, cached dataset, table, and figure is hashed with SHA-256 and chained, so a single
terminal digest fixes the entire analysis. The three manifests chain onto one another, so Part 3's
digest covers the full sequence from the source CSVs through to every reported number.

Analyte and item identifiers are resolved by matching each database's own dictionary at runtime
rather than from hard-coded lists, and the resolved mappings are printed for inspection. Random
seeds are fixed throughout.

Reporting follows TRIPOD+AI.

---

## Citation

If you use this code or the two-clock construct, please cite:

> [Author list]. When laboratory results become visible: a two-clock audit of ICU deterioration
> alerts. *Applied Clinical Informatics*. [Year];[Volume]:[Pages]. doi:[DOI]

Please also cite the underlying databases:

> Johnson AEW, Bulgarelli L, Shen L, et al. MIMIC-IV, a freely accessible electronic health record
> dataset. *Sci Data*. 2023;10:1
>
> Pollard TJ, Johnson AEW, Raffa JD, Celi LA, Mark RG, Badawi O. The eICU Collaborative Research
> Database, a freely available multi-center database for critical care research. *Sci Data*.
> 2018;5:180178

---

## License

MIT — see [LICENSE](LICENSE). This covers the code in this repository only. The underlying
databases are governed by their own PhysioNet data use agreements.

---

## Disclaimer

This is retrospective research code. It is not a medical device, has not been validated for clinical
use, and must not be used to guide patient care. The models here are analytical instruments for
studying timestamp behavior, not deployable prediction tools.

Nothing in this repository constitutes an attempt to reidentify any individual, and no such attempt
may be made using it.
