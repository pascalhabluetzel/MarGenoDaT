# MarGenoDaT
Workflows for the MARine GENOmics DAta managemenT

## MARGENODAT prototype: metabarcoding output -> Darwin Core Archive

`margenodat` reads the output of a metabarcoding pipeline, combines it with sample and project
metadata, and writes a validated Darwin Core Archive (Event core + Occurrence + DNA-derived data
[+ eMoF]) in the layout recommended by OBIS/GBIF. It is built on the eDNAqua-Plan recommendations:
term definitions come from the GBIF DNA-derived data extension and the eDNAqua-Plan SSSOM
crosswalk to MIxS (vendored in `src/margenodat/data/`, refresh with `python tools/vendor_terms.py`).

**Status: early prototype (steps 4, 5, 7, 8 of the concept note). It does not publish anything.**

### Install and run

```bash
pip install -e ".[dev]"
margenodat inspect examples/synthetic/seqtab.tsv          # how was the count table recognised?
margenodat build \
  --counts examples/synthetic/seqtab.tsv --taxonomy examples/synthetic/taxonomy.tsv \
  --samples examples/synthetic/samples.csv --project examples/synthetic/project.yaml \
  --taxon-cache examples/synthetic/taxon_cache.tsv -o out
margenodat validate out/dwca.zip
pytest
```

`build` writes `out/dwca.zip` (and the unpacked `out/dwca/`), `out/qc_report.md` and, when needed,
`out/names_to_review.tsv`. Exit code 0 = archive passes all checks, 1 = validation errors, 2 = input problem.

### What is automated

| Step | What the code does |
|---|---|
| Recognise the pipeline output | Count tables: features x samples (QIIME 2/BIOM, ...), samples x sequences (DADA2), or long format; R-style headers, comment lines, `.csv`/`.tsv`. Taxonomy: QIIME 2 `Taxon` strings or Kingdom...Species columns, embedded or separate. FASTA optional. Features identified by sequence get an MD5 key. |
| Sample metadata | Required: sample id, ISO 8601 date, WGS84 coordinates. Everything is checked and reported at once. Non-ISO dates are rejected unless `sample_sheet.date_format` is set. Negative controls (`sample_type` containing blank/control/negative/NTC) are excluded and checked against the real samples. |
| Project metadata | `project.yaml`: dataset/EML fields, defaults for DNA-derived terms (primers, target gene, pipeline...), sample-sheet columns to publish as eMoF. A sample-sheet column with the same name overrides the default. |
| Taxonomy | Names resolved through a lookup table (`--taxon-cache`). Unresolved names fall back to the next higher rank, then to Biota; the original assignment is always kept in `verbatimIdentification`, changes in `identificationRemarks`. |
| Archive | `event.txt`, `occurrence.txt`, `dna_derived_data.txt`, optional `extended_measurement_or_fact.txt`, `meta.xml`, `eml.xml`. Stable occurrenceIDs (`<eventID>_<sequence hash>`). |
| Validation | Structure and identifiers, ISO dates, coordinates, WoRMS LSID format, DNA sequence alphabet, term names against the extension, completeness of recommended DNA-derived terms. |

### Checked against real data

On the OBIS reference eDNA dataset (MBARI 18S, `iobis/dataset-edna`: 4,711 features, 70 samples) the tool
recognised the layout and embedded taxonomy, and for the 58 samples it could match, the number of
occurrences and the read sums per sample are identical to the hand-formatted OBIS version. It stopped on
sample names that differ only in letter case (`--ignore-case` fixes this explicitly) and refused to guess
one irregular name (`24416c01_12_2`), which the OBIS curators matched by hand.

### Known limits (next steps)

- **No live WoRMS/GBIF lookups.** The name lookup table has to be supplied. Without it everything is published as
  Biota (the QC report says so). A cached WoRMS matching step is the most important next feature.
- The example lookup table contains illustrative identifiers for test data only.
- eMoF `measurementTypeID`/`measurementUnitID` must be supplied in `project.yaml`; none are invented.
- The "recommended terms" list is the field set of the OBIS reference dataset; it should be replaced by the
  eDNAqua-Plan / FAIRe requirement levels.
- No metadata entry interface yet (steps 1-3), no ENA submission sheet, no IPT upload (step 9).
- The local validator does not replace the GBIF Data Validator or the OBIS quality checks.
- The example sequences are random and do not represent real organisms.
