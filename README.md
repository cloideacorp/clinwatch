
**Find out when ClinVar reclassifies a variant your lab has already reported.**

A genetics lab reports a variant as *Uncertain significance* today. Months later ClinVar
reclassifies it as *Likely pathogenic*, and in most labs nothing connects that update back to the
report already issued, so the patient is never re-contacted.

`clinwatch` closes that loop. It compares two monthly ClinVar releases, keeps only the variants
on your lab's list of reported variants (the *panel*), and tells you, for every panel row, whether
ClinVar's classification changed and whether it now disagrees with what you reported.

> **Decision support only, not a diagnostic device.** clinwatch never changes or produces a
> classification. It reports "ClinVar says X now, your report said Y" with a link to the ClinVar
> record, and a qualified person decides what to do.

- **Typed changes, never a generic "changed":** `upgrade`, `downgrade`, `conflict-introduced`,
  `conflict-resolved`, `review-status-only`, `unclassifiable`.
- **Checked against your report:** a record that did not change but already differs from what you
  reported is listed too.
- **Nothing dropped silently:** every panel row ends up as found, unchanged, filtered out, or not
  checked (with the reason).
- **Your lab's own export works:** the panel CSV can be comma-, semicolon- or tab-separated, and
  its columns are mapped automatically ([details](#panel-file)).
- **Reports for people:** plain text, a self-contained interactive HTML page, and an Excel
  workbook with color-coded alert columns next to every panel row.
- **No patient-identifying data:** the panel holds pseudonymous references only, and the importer
  warns about values that look like personal data.
- **macOS app:** a Turkish/English wizard for Apple silicon with the CLI built in.

## Status

Working: panel import, release download and verification, `diff` with text, HTML and Excel output,
and the macOS app. Planned but **not implemented yet**: `releases`, `report` and `watch` (each
exits with an error). See [`PLAN.md`](PLAN.md).

## Install

### macOS app (Apple silicon)

Build the disk image, open it, and drag **Clinwatch** to Applications:

```bash
macos/ClinwatchMac/build-app.sh
open macos/ClinwatchMac/dist/Clinwatch-0.1.0-arm64.dmg
```

The `clinwatch` CLI is embedded in the app, so nothing else needs to be installed. The app is
signed ad hoc, not notarized: on another Mac, open it the first time with right-click → **Open**.
More in [`macos/ClinwatchMac/README.md`](macos/ClinwatchMac/README.md).

### CLI from source

Needs a stable Rust toolchain.

```bash
cargo build --release
./target/release/clinwatch --help
```

Always use the `--release` build for real ClinVar files: the debug build verifies and scans them
many times more slowly.

## Quick start (offline)

The repository includes a synthetic example panel and small slices of two real ClinVar releases,
so you can try the whole flow without downloading anything:

```bash
clinwatch init
clinwatch panel import examples/panel_synthetic_example.csv
clinwatch diff \
  --from tests/fixtures/slices/variant_summary_2025-10_slice.txt.gz \
  --to   tests/fixtures/slices/variant_summary_2026-10_slice.txt.gz \
  --format html --output report.html --xlsx report.xlsx
```

The text report (`--format text`, the default) starts like this:

```text
panel: 11 rows = 11 matched + 0 unmatched
findings: 8 shown, 0 filtered out (min stars 1)
already differing from reported: 1 shown, 0 filtered out

== ClinVar now differs from the reported classification (7)
[upgrade] line 6 RPT-0001 / SUBJ-0001 CYP4V2 VariationID 39274
    reported 2025-03-14: Uncertain significance
    variant_summary_2025-10_slice.txt.gz: Uncertain significance (1 star, criteria provided, single submitter; last evaluated Sep 18, 2024)
    variant_summary_2026-10_slice.txt.gz: Pathogenic (1 star, criteria provided, single submitter; last evaluated Oct 01, 2025)
    vs reported: upgrade; matched by coordinates, VariationID
    https://www.ncbi.nlm.nih.gov/clinvar/variation/39274/
```

With your own data, name the releases by month and clinwatch downloads them from the
[ClinVar archive](https://ftp.ncbi.nlm.nih.gov/pub/clinvar/tab_delimited/archive/), verifies them
and caches them:

```bash
clinwatch panel import my_panel.csv
clinwatch diff --from 2025-10 --to 2026-10 --format html --output report.html --xlsx report.xlsx
```

A full `variant_summary` release is several hundred MB. It is streamed, never loaded into memory,
and downloaded only once.

## Panel file

The panel is a CSV with a header row, one row per reported variant. Column order does not matter.

| Field | Required | Content |
|---|---|---|
| `report_id` | yes | your internal report reference |
| `subject_ref` | yes | a **pseudonymous** internal reference, never a name or national ID |
| `reported_classification` | yes | what you reported, e.g. `Uncertain significance` |
| `reported_date` | yes | `YYYY-MM-DD` |
| `assembly`, `chromosome`, `position`, `ref`, `alt` | one key group | all five together; `GRCh37` or `GRCh38` |
| `clinvar_variation_id` | one key group | the ClinVar VariationID, e.g. `39274` or `VCV000039274` |
| `gene`, `hgvs_c`, `hgvs_p`, `notes` | no | display, best-effort HGVS matching, free text |

**Your headers do not have to match these names.** For each field, clinwatch looks for:

1. a column you chose with `--map` (`--map clinvar_variation_id=7` by column number, or
   `--map clinvar_variation_id="ClinVar No"` by header);
2. a header with the field's name or a known English or Turkish alias (`Chr`, `#CHROM`,
   `Kromozom`, `Pozisyon`, `HGVS.c`, `Rapor Tarihi`, `ClinVar No`, …);
3. values in a format only that field uses: `VCV…` accessions, `GRCh38`, ClinVar classification
   terms, ISO dates, `c.` / `p.` HGVS.

A column of plain numbers is **never** guessed, because a VariationID, an AlleleID and a position
look the same and a wrong guess would match the wrong variant. If two columns fit one field, the
import stops and names both. Preview the mapping without importing anything:

```bash
clinwatch panel inspect my_panel.csv
```

Every import records which column each field came from. Rows that cannot be used are kept and
reported with the reason, never dropped. Columns that are not mapped to a field are never stored.

## Reports

| Output | Option | Best for |
|---|---|---|
| Text | `--format text` (default) | terminal, logs, scripts |
| HTML | `--format html --output FILE` | reading and searching; one self-contained file |
| Excel | `--xlsx FILE` (in addition to the above) | working through the list |

The Excel workbook lists **every panel row**, with these columns appended to its own:

| Alert | Meaning |
|---|---|
| **ACTION** | ClinVar's current classification differs from the one you reported |
| **REVIEW** | ClinVar moved in agreement with your report, or the record was withdrawn, is new, or is ambiguous |
| **INFO** | a change with no direction relative to your report, or one hidden by your filters |
| **OK** | no change, and ClinVar agrees with your report |
| **NOT CHECKED** | the row could not be compared (unusable, or not in either release) |

The workbook also has the status in words, ClinVar's classification before and now, star rating,
review status, last-evaluated date and a link to the ClinVar record. A second sheet records the
releases, their sha256 checksums and the filters used.

Narrow the findings with `--min-stars`, `--only upgrade,downgrade`, `--since YYYY-MM-DD` and
`--gene SYMBOL`. Filtered findings are still counted, and in Excel they appear marked as hidden.

## Exit codes

For use in scripts and scheduled jobs:

| Code | Meaning |
|---|---|
| `0` | ran successfully, no findings |
| `10` | ran successfully, findings present |
| `1` | runtime error |
| `2` | usage error |

## Data and privacy

- ClinVar data is used as published by NCBI; clinwatch reads `variant_summary.txt.gz` and never
  modifies classifications.
- Each downloaded release gets a sha256 checksum at download time, and the checksum is checked on
  every later use. Reports show both releases' checksums.
- The local SQLite store holds release metadata, imported panels and the column mappings. It does
  not hold ClinVar itself.
- The PII guard warns when `report_id`, `subject_ref` or `notes` contain something that looks like
  a national ID (11 digits), an email address, a date of birth, or a name. Use `--strict-pii` to
  refuse such files. This is a safety net, not a compliance guarantee.

## Development

```bash
cargo test                                   # fully offline
cargo clippy --all-targets -- -D warnings
cargo fmt --check
swift test --package-path macos/ClinwatchMac
```

Contributor rules are in [`CLAUDE.md`](CLAUDE.md), milestones and decisions in
[`PLAN.md`](PLAN.md), and the provenance of the test fixtures in
[`tests/fixtures/README.md`](tests/fixtures/README.md).

## License

MIT. See [`LICENSE`](LICENSE).
