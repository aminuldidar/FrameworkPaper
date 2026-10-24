# Replication Package

## Overview

This replication package accompanies the paper:

> **License Compliance Debt: A Provenance-Based Framework for Quantifying License Compliance Debt**

This package is provided anonymized for double-blind review; the full artifact, including the mining pipeline source code, will be released as a permanent, citable archive on Zenodo once the paper is accepted.

**This is the clean, upload-ready copy of the package** (2026-10-24), matching the same anonymization pass done for this project's other replication package. Removed from the working directory before this copy was finalized: a live `.git/` history (real author name, email, and GitHub remote), `__pycache__/`, and a `LICENSE` file whose copyright line named the author directly -- none of these belong in a double-blind package. `verify_package.ipynb` was re-executed from this copy and runs clean (0 errors).

The package contains the implementation of the Directional Sink Rule (DSR) engine together with the datasets used to validate the proposed License Compliance Debt (LCD) classification framework.

The replication package enables reviewers and researchers to:

* reproduce the LCD classification workflow;
* verify the DSR-based license compatibility decisions;
* inspect the manually validated ground-truth dataset;
* reproduce the validation results reported in the paper.

---

## Package Structure

```text
.
├── verify_package.ipynb
├── DSR_engine.py
├── human_validation.csv
├── cohort_data/
│   ├── corporate_validation_source_sink.csv
│   ├── foundation_validation_source_sink.csv
│   └── individual_validation_source_sink.csv
├── README.md
├── MINER_SETUP.md
```

## `cohort_data/`

This directory contains the validation datasets used for the ecosystem-scale evaluation reported in the paper.

- **`corporate_validation_source_sink.csv`** — Method-level provenance relationships extracted from corporate open-source repositories.

- **`foundation_validation_source_sink.csv`** — Method-level provenance relationships extracted from repositories maintained by open-source foundations.

- **`individual_validation_source_sink.csv`** — Method-level provenance relationships extracted from independently maintained open-source projects.

Together, these three datasets comprise the 309-repository validation corpus used to evaluate the proposed License Compliance Debt (LCD) framework across diverse software ecosystem cohorts, totalling 107,208 method-level provenance relationships (45,472 corporate / 34,852 foundation / 26,884 individual) -- matching Table VI's category breakdown exactly (Category 1=15,901, Category 2=6,746, Category 3=13,123, Category 4=1,808, Category 5=44,804, Category 0=24,826).

**Columns**: `hash, source_project_id, sink_project_id, source_version, sink_version, source_license, sink_license, method_name, source_file_location, sink_file_location, source_repository_url, sink_repository_url, violation, automated_classifier_category`. Despite the column names, `source_repository_url`/`sink_repository_url` are full blob links down to the file and line (e.g. `.../blob/<sha>/path/to/file.c#L512`), not bare repository URLs -- `source_project_id`/`sink_project_id` are the only stable per-repository identifiers in the file and were kept for that reason even though they are opaque numeric IDs.

---

# File Descriptions

## `verify_package.ipynb`

Jupyter notebook demonstrating the complete verification workflow, in two parts:

* **Table IV** (classification fidelity): loads `human_validation.csv`, compares the classifier's
  `automated_classifier_category` against the reviewer-resolved `resolution_category` ground
  truth, and reports the per-category precision/recall/F1 and Cohen's kappa.
* **Table VI** (ecosystem feasibility results): loads the three `cohort_data/*.csv` files,
  aggregates per cohort (Corporate/OSS Fdn./Individual) and in total, and reproduces the paper's
  309-repository, 107,208-method LCD classification breakdown and $LCD_{\text{density}}$ figures
  exactly -- recomputed directly from the shipped data, not hardcoded.

This notebook serves as the primary entry point for reproducing the experimental results.

---

## `DSR_engine.py`

Python implementation of the proposed **Directional Sink Rule (DSR)** algorithm.

The implementation includes:

* license compatibility reasoning;
* provenance-aware source-to-sink analysis;
* LCD category assignment;
* rule evaluation used throughout the paper.

---

## `human_validation.csv`

Ground-truth dataset used for evaluating the classification accuracy of the LCD framework
(N=1,935 method pairs). Used to compute the precision, recall, F1, and Cohen's kappa reported in
the paper's Table IV.

**Columns renamed**

| Original name | New name |
| --- | --- |
| `LCD_category` | `automated_classifier_category` |
| `human_Annotation_1`, `human_Annotation_2` | `reviewer_agreement_1`, `reviewer_agreement_2` |

**Columns added**

| Column | Type | Description |
| --- | --- | --- |
| `resolution_category` | Integer (1-5) | The row's final, reviewer-confirmed category. Populated for all 1,935 rows. |
| `resolution_method` | String | How the row was resolved -- see table below. |
| `pending_review_note` | String | Plain-language note explaining the resolution, especially for rows not resolved by simple unanimous agreement. |

### How `human_validation.csv` was built

Ground truth was established through a two-round, two-reviewer process. Round 1 asked reviewers
to accept or reject each pair's pre-assigned category in a binary task (Cohen's kappa = 0.8372 on
this binary judgment, from the raw `reviewer_agreement_1`/`reviewer_agreement_2` columns). This
surfaced a task-design problem, not a classification one: a bare accept/reject gave reviewers no
mechanism to investigate a counterintuitive label, so Category 4 was reflexively rejected in
round 1 (0% agreement across all 341 cases) despite its underlying rule being sound. Round 2 asked
reviewers to independently assign their own category for every disputed pair, informed by live
verification of the underlying repository and license metadata, rather than accepting or
rejecting a suggestion -- resolving all 1,935 rows into the final `resolution_category`, with
96--100% reviewer agreement across every disputed subset. Per-pair resolution records, including
how each case was resolved and why, are in `resolution_method`/`pending_review_note` (value counts
below).

### Annotation guidelines and consensus procedure

Round 2 reviewers worked from a small set of explicit decision rules, not open-ended judgment.
These are the rules as actually applied, confirmed by round 2's reviewer agreement rates,
documented here since the paper itself doesn't have room to state them in this much detail:

1. **"Same License" (Category 1) requires identical license strings, not merely compatible ones.**
   `MIT` source into an `Apache-2.0` sink is Category 2 (Compatible), not Category 1, even though
   both are permissive. This is the rule a historical mislabel (13 pairs total, now corrected)
   violated.
2. **Risk-first principle (Category 4): symmetric uncertainty is treated as active risk.** When
   *both* sides are unverified (source license `Unknown`/`Unlicensed` AND sink license
   `Custom`/`Proprietary`), the pair is Category 4, not Category 5 -- not knowing what the sink's
   terms actually permit is itself the risk signal. This rule is what `DSR_engine.py`'s own
   high-risk check implements, and reviewers confirmed it directly (96.1% round-2 agreement on the
   258 re-judged Category 4 cases), reversing round 1's misleadingly alarming 0% figure.
3. **Symmetric uncertainty, not partial uncertainty, is what triggers Category 4.** If the source
   is `Unknown` but the sink license is a *confirmed, known* permissive license (not `Custom`),
   the pair is Category 5 (Latent Debt), not Category 4 -- one side being confirmed-open breaks the
   risk-first reasoning, which depends on uncertainty on *both* sides. This was the single
   highest-volume rule in round 2, and reviewers confirmed it unanimously.
4. **A confirmed structural incompatibility is not overridden by one side being `Custom`.** A
   `Custom`-licensed source into a confirmed, structurally incompatible sink (e.g. `BSD-3-Clause`)
   stays Category 3 (Violation); the risk-first principle (rule 2) does not reach back and
   reclassify an already-structural conflict as mere uncertainty.
5. **Live verification is required before accepting a repository-level detection-failure value at
   face value.** Where `source_repo_level_license`/`sink_repo_level_license` showed a value like
   `No License File Found` or `NOASSERTION`, reviewers checked the live GitHub API (license field,
   with a root-directory file listing fallback) before treating the absence as confirmed, to catch
   the known Licensee-tool limitation of missing a real but non-standard LICENSE file.

**Consensus procedure**: all round-2 cases were judged independently by the same two reviewers who
did round 1 (no third party was introduced). Disagreements were not adjudicated case-by-case;
instead, once a rule above was confirmed by the dominant pattern in a batch, it was applied
consistently to the rest of that batch, and any single remaining split vote was resolved to the
majority answer (`resolution_method = resolved_by_batch_consistency` /
`resolved_by_reannotation_principle`) with both reviewers' original answers kept on record in
`pending_review_note` rather than discarded.

### `resolution_method` value counts

| Value | Count |
| --- | --- |
| `unanimous_human_agreement` | 1,267 |
| `unanimous_reviewer_reannotation` | 560 |
| `auto_resolved_undefined_source_url` | 84 |
| `corrected_by_engine_audit` | 13 |
| `resolved_by_reannotation_principle` | 9 |
| `corrected_for_batch_c_consistency` | 1 |
| `resolved_by_batch_consistency` | 1 |

### Validation results

Reproduced by `verify_package.ipynb`, computed directly from this file (`automated_classifier_category`
vs. `resolution_category`, N=1,934, Category 0 excluded):

| Category | Precision | Recall | F1 | Support |
| --- | --- | --- | --- | --- |
| 1 (Same) | 96.9% | 100.0% | 98.4% | 410 |
| 2 (Compatible) | 100.0% | 96.5% | 98.2% | 400 |
| 3 (Violation) | 100.0% | 100.0% | 100.0% | 389 |
| 4 (Active-Risk) | 75.4% | 100.0% | 86.0% | 257 |
| 5 (Latent Debt) | 99.7% | 82.4% | 90.3% | 478 |
| **Macro avg** | **94.4%** | **95.8%** | **94.6%** | 1,934 |

Cohen's kappa (automated vs. reviewer-resolved ground truth) = **0.9365**.

**Also verified**: `verify_package.ipynb`'s DSR Engine Self-Consistency Check re-runs
`classify_violation()` directly against all 82,382 non-Category-0 pairs in `cohort_data/` (the
full corpus behind Table VI) and compares to the recorded category: mismatch rate is 0.000%
(0/82,382), confirming Table VI's headline `LCD_density` (18.1%) and latent-risk (41.8%) figures
are consistent with the shipped engine.

---

## `corporate_validation_source_sink.csv`

Validation dataset containing method-level provenance relationships extracted from corporate open-source repositories.

This dataset supports the ecosystem validation presented in the paper.

---

## `foundation_validation_source_sink.csv`

Validation dataset containing provenance relationships collected from open-source foundation projects.

Examples include repositories maintained by established OSS foundations.

---

## `individual_validation_source_sink.csv`

Validation dataset containing provenance relationships extracted from independently maintained open-source projects.

Together with the corporate and foundation datasets, this corpus forms the validation cohorts reported in the paper.

---

# Illustrative Usage: The Mining and Classification Pipeline

The datasets in this package (`cohort_data/`, `human_validation.csv`) are the output of a two-stage pipeline: a method-level mining/indexing stage that builds the provenance index these datasets are drawn from, and the classification stage (`DSR_engine.py`, included in full in this package) that assigns the LCD categories. The mining/indexing stage's own source code is not included here, but its operational model — CLI usage, credential policy, hashing, and revisit behavior — is documented in **[`MINER_SETUP.md`](./MINER_SETUP.md)**, shared across this project's companion replication packages so the description stays identical everywhere it appears.

`DSR_engine.py`, included in full here, takes each mined source/sink license pair and assigns the LCD category (the `hash` column in `cohort_data/` and `human_validation.csv` is the structural signature `MINER_SETUP.md` describes) -- this classification step is completely reproducible from this package alone; only the upstream mining/indexing step requires the credentials `MINER_SETUP.md` describes. `verify_package.ipynb` reproduces the classification and validation statistics directly from the included datasets without needing either credential. Note also that `source_version`/`sink_version` are mining-time snapshot timestamps, not per-commit VCS timestamps -- see `MINER_SETUP.md`'s revisit-behavior note and the paper's methodology section.

---

# LCD Categories

The DSR engine assigns one of five License Compliance Debt (LCD) categories:

| Category | Description              |
| -------- | ------------------------ |
| 1        | Same License             |
| 2        | Compatible License Reuse |
| 3        | License Incompatibility  |
| 4        | Active Compliance Risk   |
| 5        | Latent Compliance Risk   |

Category 0 represents methods for which no provenance match was identified and is excluded from LCD density calculations.

---

# Requirements

The replication package requires:

* Python 3.10+
* Jupyter Notebook

Python packages:

```
pandas
numpy
```

Install dependencies using:

```bash
pip install pandas numpy notebook
```

---

# Running the Replication

1. Install the required Python packages.

2. Launch Jupyter Notebook:

```bash
jupyter notebook
```

3. Open

```
verify_package.ipynb
```

4. Execute all notebook cells.

The notebook will:

* load the validation datasets;
* execute the DSR engine;
* classify each provenance relationship;
* compare predictions against the reviewer-resolved ground truth in `human_validation.csv`.

---

# Validation Corpus

The validation corpus consists of:

* 309 repositories
* 107,208 method-level provenance relationships
* three ecosystem cohorts:

  * Corporate
  * OSS Foundation
  * Individual

A manually verified subset was used to evaluate classification performance, as described in the paper.

---

# License

This replication package is released solely for research and reproducibility purposes accompanying the associated publication.

Repository URLs contained within the datasets refer to their original upstream open-source projects and remain subject to their respective licenses.

---

# Citation

If you use this replication package, please cite the accompanying paper describing the License Compliance Debt (LCD) framework.
