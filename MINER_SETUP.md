# Mining Pipeline: Setup and Operational Model

The datasets in this package are the output of a method-level mining and indexing pipeline. The pipeline's own source code is not included in this anonymized package (it is a separate, actively developed research artifact, shared across a set of companion submissions, and maintained outside the scope of this paper), but its operational model is described here so the process producing these datasets is transparent and, in principle, reproducible against a re-implementation.

## Operational modes

The pipeline exposes two operational modes via a command-line interface:

**1) Single-repository check**: takes a repository URL, extracts and hashes its methods, and reports which already-indexed methods match.

```text
<cli> check <repository-url> [--branch <branch>] [--tag <version>] [--github-token <token>] [--threads <n>]
<cli> checkupload <repository-url> [options]   # same as check, but also uploads if not already indexed
```

**2) Continuous mining mode**: runs as a long-lived loop, pulling repository-discovery and parsing jobs from a work queue (repositories are discovered and prioritized via the GitHub search API, sorted by star count), extracting methods, and uploading new or changed snapshots to the index.

```text
<cli> start [--miner-name <name>] [--github-token <token>] [--threads <n>]
```

## Extraction and hashing

Supported-language source files are parsed into ASTs; files in unsupported formats are skipped. Each extracted method is canonicalized (non-semantic identifiers stripped) and hashed with MD5 to produce the 128-bit structural signature carried as the `hash` column throughout this package's datasets.

## Configuration

Running the pipeline requires two kinds of credentials:

- **`GITHUB_TOKEN`**: a GitHub API token (for repository, commit, and tag metadata). This is a personal credential tied to each user's own GitHub account and rate limits, so it is **not** provided here; reviewers wishing to run the pipeline should supply their own (free, standard GitHub personal access token). Reproducing the full pipeline also requires installing [srcML](https://www.srcml.org/) (an established open-source AST-to-XML parser used for some supported languages) and Git locally, so a reviewer going through that setup already needs to provide their own token as part of the same process.
- **`JSON_TOKEN` / `JSON_URL`**: credentials for the project's own database API, used only by the continuous-mining mode to pull jobs and upload results. For reviewers who want to exercise this path during the review period, access can be requested from the authors via the anonymous review channel; this credential is revocable at any time and is deliberately **not** included in any permanent, non-anonymized archive (e.g., a future Zenodo release) of this package.

## Note on independent reproduction

Because both credentials above are either reviewer-supplied or deliberately excluded from permanent archives, the datasets already included in this package, rather than a live re-run, are the practical basis for verifying this paper's reported results.

## Revisit behavior

A repository already in the index is only re-processed if it has changed since its last recorded snapshot (tracked server-side), so repeated mining passes do not re-upload unchanged projects; see the paper's Methodology section for how this affects the versioning scope of the index.
