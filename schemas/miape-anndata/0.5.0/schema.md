# MIAPE-AnnData Schema

Contact: lboatner@chanzuckerberg.com (Chan Zuckerberg Initiative; HUPO-PSI AI Readiness Working Group)

Document Status: _Draft_ (first cut, 2026-11-17; see [Status of this draft](#status-of-this-draft))

Version: 0.5.0

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED" "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://tools.ietf.org/html/bcp14), [RFC2119](https://www.rfc-editor.org/rfc/rfc2119.txt), and [RFC8174](https://www.rfc-editor.org/rfc/rfc8174.txt) when, and only when, they appear in all capitals, as shown here.

## Schema versioning

The MIAPE-AnnData schema version is based on [Semantic Versioning](https://semver.org/).

**Major version** is incremented when schema updates are incompatible with the AnnData data encoding or with downstream MIAPE-AnnData consumers. Examples include:
  * Renaming metadata fields
  * Deprecating metadata fields
  * Changing the type or format of a metadata field

**Minor version** is incremented when schema updates are additive. Examples include:
  * Adding a new modality (e.g. peptide, peptidoform, site)
  * Adding metadata fields
  * Updating pinned ontologies
  * Changing the validation requirements for a metadata field

**Patch version** is incremented for editorial updates that do not change validation behavior.

All changes are documented in the schema [Changelog](#appendix-a-changelog). Versions 0.1.0 through 0.4.0 were never released outside the maintaining subgroup; v0.5.0 is the first version circulated for implementation, and it absorbs the field renames and format changes that the earlier drafts would otherwise have carried as major-version bumps.


## Background

MIAPE-AnnData specifies the minimum information that a proteomics quantification dataset MUST carry when serialized as an [AnnData](https://anndata.readthedocs.io/en/latest) artifact, so that datasets produced by different search engines and analysis tools are mutually interpretable, tool-agnostic, and suitable for public deposition and cross-study integration.

This document is maintained by the HUPO-PSI AI Readiness Working Group, Software Engineering Subgroup (the "PSI SWE Subgroup"). The subgroup represents the proteomics standards community and is the ongoing maintainer of this specification. Downstream consumers — including search-engine developers, scverse maintainers, and repository operators — adopt this specification; they do not author or maintain it.

The schema was phased across minor versions, each adding a single new modality so that each ratification cycle is small and focused. v0.1.0 covered the **protein-group** modality; v0.2.0 added the bare-sequence **peptide** modality and the multi-modality `.h5mu` artifact shape; v0.3.0 added the **peptidoform** modality; v0.4.0 added the **site** modality and the **PTM-stoichiometry-adjustment** provenance pattern. **v0.5.0 (this version)** is the first cut written against what six current producers actually emit (alphapepttools, msmu, apb2, proteopy, QPX, protdata). It adds the **peptidoform ion** modality (precursor level, PSI-MS `peptidoform ion`), frees the grammar of every feature index and of the `obs` index while moving identity into normalized, validator-checkable columns, fixes what one `obs` row is for multiplexed and fractionated data, adopts the sdrf-pipelines channel-label vocabulary, binds the peptidoform string to ProForma 2.1, stores provenance as a single JSON string that survives HDF5, excludes decoy features from the canonical surface, and adds a per-feature `q_value`. The design record is `docs/designs/RFC-002-miape-anndata-v050-design.md` and ADR-005 … ADR-016 in `docs/DECISIONS.md` of the maintaining repository.

Datasets often carry additional metadata beyond this specification. MIAPE-AnnData defines the minimum required surface; producers MAY preserve engine-native columns alongside the canonical MIAPE-AnnData columns.

### Status of this draft

This is the **first cut** of v0.5.0 (2026-11-17). It is complete for feature identity, observation identity, the matrix and layer rules, the `obs` metadata surface, the `uns` scalar fields, the provenance container, decoys and per-feature confidence. The following items are **deferred** to a second pass and are marked `DEFERRED` where they would appear in the text. Each will land as a SHOULD, a MAY or a loosening of an existing rule, by in-place patch of this Draft, so an artifact written against this cut remains valid:

- Provenance step order and DAG edges (`id`, `inputs`); whether `step` stays.
- The step-1 anchor and the recording of pre-`raw` operations (engine-side FDR, match-between-runs, engine normalisation, engine-side fraction aggregation).
- Field-level provenance (which source column became which canonical column).
- Binding `TransformationStep.params` keys to Quantification CV terms.
- Declared, typed score layers (per-run q-values, PEP matrices).
- The `missingness_mask` clarification (when the mask carries non-derivable information).
- The PTM-stoichiometry adjustment: membership-match rule, `adjustment_status` column, an `enrichment_process` `obs` column; terminal modification positions at site level.
- Result slots (differential-abundance tables) and gene-level matrices.
- A multiplexed, fractionated worked example.

### Normative tiers

MUST is reserved for what a validator can check mechanically or what a consumer needs to locate data: matrix dtype and NaN encoding, shape and label identity across layers, index-equals-column identity, the enumerated values of `uns['level']`, `uns['labeling_type']`, `uns['linkage_convention']` and the modality keys, the shared `obs` index in `.h5mu`, cross-modality foreign-key resolution, the `(raw_file, label)` observation identity, and a non-empty provenance container. SHOULD carries the conventions the current producers taught us. Where producers differ, the schema does not pick a winner by MUST: it either makes a requirement **conditional** (`missingness_mask` is MUST only when `imputed` is present; `protein_group` on peptide rows is MUST only when a protein modality is present) or asks the producer to **declare the convention** in use (`linkage_convention`). Flexible is not unspecified: every permitted option is specified well enough to validate.

### Delimiters

Two delimiters are used throughout, with fixed meanings. **`;` separates the members of a list** held in one string cell (`protein_ids`, `residue_positions`, the member files of an aggregated `raw_file`, the aligned `fraction` values). **`|` separates the fields of a composite key** (the recommended multiplexed `obs` index `raw_file|label`, the recommended `site_id` form, the factor values composed into `condition`). Neither delimiter may appear inside a list member or a key field.


## Overview

This schema supports the protein-group, bare-sequence peptide, peptidoform, peptidoform-ion (precursor) and PTM-localized site quantification modalities of mass-spectrometry-based proteomics experiments. A compliant artifact is either a single-modality AnnData object (`.h5ad` file) whose `X` matrix is observations × features at one modality level, or a multi-modality MuData object (`.h5mu` file) carrying two or more of these modalities in a single container with cross-modality feature linkage recorded in `varp`.

This document is organized by:

* [General Requirements](#general-requirements) — file format, dtype, and reserved-name conventions
* [Required Ontologies](#required-ontologies) — the controlled vocabularies referenced by the schema
* [`X` (Matrix Layers)](#x-matrix-layers) — the primary quantification matrix
* [`layers`](#layers) — the canonical secondary matrices (raw, log2, normalized, imputed, etc.)
* [`obs` (Sample Metadata)](#obs-sample-metadata) — one row per quantification channel (one `label` within one acquisition or declared aggregate of acquisitions; sample-level metadata is duplicated across the rows of a sample)
* [`var` (Feature Metadata)](#var-feature-metadata) — one row per quantified feature, per modality
* [MuData Composition](#mudata-composition) — multi-modality `.h5mu` artifact shape and modality keys
* [`varp` (Cross-modality feature linkage)](#varp-cross-modality-feature-linkage) — sparse adjacency between modalities
* [`uns` (Dataset Metadata)](#uns-dataset-metadata) — dataset-level provenance and version fields
* [SDRF Integration](#sdrf-integration) — informational mapping from SDRF-Proteomics to `obs`
* [Appendix E](#appendix-e-label-vocabulary) — the `label` vocabulary

## General Requirements

**AnnData.** The canonical on-disk format for MIAPE-AnnData is HDF5-backed [AnnData](https://anndata.readthedocs.io/en/latest) as written by AnnData version 0.10.0 or greater. The on-disk format MUST conform to the [AnnData specification (v0.1.0)](https://anndata.readthedocs.io/en/latest/fileformat-prose.html#anndata-specification-v0-1-0).

**Modality.** A v0.5.0-compliant artifact MUST be either:
1. A single `.h5ad` file containing **one** of the supported modalities. The value of `uns['level']` MUST be one of `"protein"`, `"peptide"`, `"peptidoform"`, `"peptidoform_ion"`, or `"site"`.
2. A `.h5mu` file ([MuData specification](https://mudata.readthedocs.io/)) containing **two or more** modality keys drawn from `{"protein", "peptide", "peptidoform", "peptidoform_ion", "site"}`. The value of `mdata.uns['level']` MUST be `"multi"`, and the per-modality `mdata.mod['<level>'].uns['level']` MUST be the modality-specific value. Cross-modality feature linkage MUST be recorded per [`varp`](#varp-cross-modality-feature-linkage).

The value of `uns['miape_anndata_spec']` (at the MuData level for multi-modality artifacts, at the AnnData level otherwise) MUST be `"0.5.0"`.

**Reserved Names**. The names of metadata fields MUST NOT start with `"__"`. The names of the metadata fields specified by this schema are **reserved** for the purposes and specifications described in the schema. Producers MAY preserve engine-native column names as additional, non-reserved columns alongside the canonical MIAPE-AnnData columns; where a conflict exists between a canonical name and an engine-native name, the canonical name takes precedence.

**Unique Names**. The names of metadata fields in `obs` and `var` MUST be unique. Duplicate canonical-name columns are not allowed.

**Redundant Metadata**. It is STRONGLY RECOMMENDED to avoid multiple metadata fields containing identical or similar information.

**No Personal Identifiable Information (PII)**. Producers MUST NOT include direct personal identifiers (names, emails, contact information) in the metadata portions of a MIAPE-AnnData artifact. Subject-level identifiers MUST be limited to pseudonymous donor identifiers (`donor_id`).

**Decoy features.** A compliant artifact MUST NOT carry decoy (reversed, shuffled or knock-off) features in any `var` or in any matrix. The feature population of every modality is the identified population that passed the recorded filters. Decoy evidence used to compute or recompute error rates MAY be stored in `uns` under a producer-chosen key; it is outside the canonical surface and validators ignore it.

**Reserved Names from previous schema versions** that have since been deprecated MUST NOT be present in datasets. As of v0.5.0 the deprecated names are `obs['replicate']` (renamed `technical_replicate`) and the site modality's `var['residue_position']` (replaced by the list column `residue_positions`).

#### *Note on types*
The types below are Python 3 types. A Python 3 `str` is a sequence of Unicode code points, stored UTF-8-encoded by AnnData. A `CURIE` is a colon-delimited compact identifier of the form `Ontology:Identifier` (e.g. `EFO:0022605`); see [Required Ontologies](#required-ontologies) for prefix conventions.

## Required Ontologies

Ontology terms for metadata MUST use [OBO-format identifiers](http://www.obofoundry.org/id-policy.html), meaning a CURIE (prefixed identifier) of the form **Ontology:Identifier**, with the exception of Cellosaurus, which requires the form **Ontology_Identifier** (e.g. [`CVCL_1P02`](https://www.cellosaurus.org/CVCL_1P02)).

If a required ontology is missing a needed term, ontologists are responsive to New Term Requests (NTRs). The PSI SWE Subgroup will coordinate ontology-extension requests on behalf of the proteomics community.

The following ontologies are referenced in this schema:

| Ontology | Prefix |
|:--|:--|
| [BRENDA Tissue Ontology] | BTO: |
| [Cell Ontology] | CL: |
| [Cell Line Ontology] | CLO: |
| [Cellosaurus] | CVCL_ |
| [ChEBI] | CHEBI: |
| [Disease Ontology] | DOID: |
| [Experimental Factor Ontology] | EFO: |
| [Gene Ontology] | GO: |
| [Human Developmental Stages] | HsapDv: |
| [Mondo Disease Ontology] | MONDO: |
| [Mouse Developmental Stages] | MmusDv: |
| [NCBI Organismal Classification] | NCBITaxon: |
| [PSI Mass Spectrometry Ontology] | MS: |
| [PSI Modification Ontology] | MOD: |
| [Phenotype And Trait Ontology] | PATO: |
| [Quantification Mass Spectrometry CV] | QCMS: |
| [Uberon Multi-Species Anatomy Ontology] | UBERON: |
| [UniProt Knowledgebase] | UniProtKB: |
| [Unimod] | UNIMOD: |
| | |

[BRENDA Tissue Ontology]: https://www.brenda-enzymes.org/ontology.php
[Cell Ontology]: http://obofoundry.org/ontology/cl.html
[Cell Line Ontology]: https://www.ebi.ac.uk/ols4/ontologies/clo
[Cellosaurus]: https://www.cellosaurus.org/
[ChEBI]: https://www.ebi.ac.uk/chebi/
[Disease Ontology]: https://disease-ontology.org/
[Experimental Factor Ontology]: http://www.ebi.ac.uk/efo
[Gene Ontology]: https://geneontology.org/
[Human Developmental Stages]: http://obofoundry.org/ontology/hsapdv.html
[Mondo Disease Ontology]: http://obofoundry.org/ontology/mondo.html
[Mouse Developmental Stages]: http://obofoundry.org/ontology/mmusdv.html
[NCBI Organismal Classification]: http://obofoundry.org/ontology/ncbitaxon.html
[PSI Mass Spectrometry Ontology]: https://www.psidev.info/groups/controlled-vocabularies
[PSI Modification Ontology]: https://www.psidev.info/MOD
[Phenotype And Trait Ontology]: http://www.obofoundry.org/ontology/pato.html
[Quantification Mass Spectrometry CV]: https://github.com/HUPO-PSI/psi-ms-CV
[Uberon Multi-Species Anatomy Ontology]: http://www.obofoundry.org/ontology/uberon.html
[UniProt Knowledgebase]: https://www.uniprot.org/
[Unimod]: https://www.unimod.org/

The `label` column uses a vocabulary that is not an ontology: the canonical channel labels of the [sdrf-pipelines](https://github.com/bigbio/sdrf-pipelines) channel map, reproduced in [Appendix E](#appendix-e-label-vocabulary).

## `X` (Matrix Layers)

The data stored in the `AnnData.X` data matrix is the primary quantification matrix of the artifact. Rows correspond to observations (indexed by `obs.assay_id` per [§obs](#obs-sample-metadata)). Columns correspond to features at the modality's level, indexed by the modality's `var` index — `var.protein_group` for the protein modality, `var.sequence` for peptide, `var.peptidoform` for peptidoform, `var.peptidoform_ion` for peptidoform ion, `var.site_id` for site (in `.h5mu` artifacts the active column index is determined by which `mdata.mod['<level>']` the matrix lives under).

MIAPE-AnnData's matrix layer requirements are tailored to preserve the missing-value semantics that are central to proteomics data analysis:

* `AnnData.X` MUST be a dense [`numpy.ndarray`](https://numpy.org/doc/stable/reference/generated/numpy.ndarray.html) of dtype `numpy.float32` or `numpy.float64`.
* Missing values MUST be encoded as `numpy.nan`. Zero-imputed missing values are non-compliant; sparse encodings (CSR/CSC/COO) are non-compliant.
* `AnnData.X` SHOULD contain raw (or near-raw) quantification intensities. The same matrix SHOULD also be present in `layers['raw']` so that the `transformations` chain (see [`quantification_provenance`](#quantification_provenance)) can anchor on a stable reference matrix without overwriting `X`.
* All matrix layers (`X` and every entry in `layers`) MUST have the same shape, the same row labels (`obs.assay_id`), and the same column labels (the modality's `var` index).
* Any observation filtering MUST be applied across `X` and every entry in `layers`. Features SHOULD NOT be filtered out of raw or intensity-bearing layers; downstream filters MAY be expressed by adding boolean flag columns to `var`.

The following table describes the canonical matrix shape:

| Attribute | Definition |
|---|---|
| `AnnData.X` | dense `float32` or `float64` matrix, shape `(n_observations, n_features)`. Missing = `NaN`. |
| `AnnData.obs.index` | `assay_id`, unique, string. |
| `AnnData.var.index` | The modality's canonical index column (`protein_group` for protein, `sequence` for peptide, `peptidoform` for peptidoform, `peptidoform_ion` for peptidoform ion, `site_id` for site), unique, string. |

## `layers`

`layers` is a [`dict`](https://docs.python.org/3/library/stdtypes.html#mapping-types-dict) of named matrices, each with the same shape and labels as `X`. The following layer names are **canonical**: when a producer stores a matrix with the semantic content described below, the producer MUST use the canonical name.

Producers MAY store additional, non-canonical layers under producer-chosen names. Non-canonical layers MUST NOT begin with `"__"` and SHOULD have names that accurately summarize what the matrix represents (e.g. `counts_per_million_protein`, `imputed_lod_minimum`). `DEFERRED`: a declaration mechanism for typed score layers (per-run q-values, posterior error probabilities) is part of the second pass; until then such matrices are non-canonical layers.

### raw

<table><tbody>
    <tr>
      <th>Key</th>
      <td>raw</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>The unmodified engine-output quantification matrix. The anchor matrix that the <code>transformations</code> chain in <a href="#quantification_provenance"><code>quantification_provenance</code></a> reads from at step 1.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>Dense <code>numpy.ndarray</code> of dtype <code>numpy.float32</code> or <code>numpy.float64</code>, same shape and labels as <code>X</code>. Missing values MUST be <code>NaN</code>. In many pipelines, <code>layers['raw']</code> and <code>X</code> hold identical content; this duplication is intentional, since <code>X</code> is the analyst-facing matrix while <code>layers['raw']</code> is the provenance anchor.</td>
    </tr>
</tbody></table>
<br>

### missingness_mask

<table><tbody>
    <tr>
      <th>Key</th>
      <td>missingness_mask</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Boolean matrix recording which positions in <code>layers['raw']</code> were missing or special values prior to any imputation. Polarity: <code>True</code> = missing, <code>False</code> = observed.<br><br><strong>Polarity ratification (WG):</strong> This specification fixes <code>True = missing, False = observed</code> to match the Python ecosystem convention (<a href="https://numpy.org/doc/stable/reference/maskedarray.generic.html"><code>numpy.ma.MaskedArray</code></a>, <code>pandas.isna()</code>, <code>alphapepttools.pp.detect_special_values()</code>). R / Bioconductor tooling more commonly uses the opposite convention (<code>True = observed</code>); cross-language consumers MUST translate at the language boundary. This polarity is fixed for ratification stability — changing it post-ratification would silently corrupt every downstream analysis written against the old convention.<br><br><code>DEFERRED</code>: the second pass states when the mask carries information not derivable from <code>layers['raw']</code> (engine special values converted to <code>NaN</code> on read; imputation flags such as QPX's <code>imputed_mask</code> and proteopy's <code>imputation_mask_X</code>). The conditional requirement below stands.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate when <code>layers['imputed']</code> is present; otherwise SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>Dense matrix, same shape as <code>X</code>. On-disk dtype SHOULD be <code>numpy.uint8</code> with <code>1</code> = missing and <code>0</code> = observed, since native HDF5 booleans serialize inconsistently across AnnData versions. Loaders MAY cast back to <code>bool</code> in memory. The polarity convention (<code>True</code> = missing) is fixed.</td>
    </tr>
</tbody></table>
<br>

### log2

<table><tbody>
    <tr>
      <th>Key</th>
      <td>log2</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Log<sub>2</sub>-transformed quantification values. Produced from <code>layers['raw']</code> by a <code>log2_transform</code> <code>TransformationStep</code> in <code>quantification_provenance.transformations</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>Dense <code>numpy.ndarray</code> of dtype <code>numpy.float32</code> or <code>numpy.float64</code>, same shape and labels as <code>X</code>. Missing values MUST be <code>NaN</code>. NaN-preserving log<sub>2</sub> (e.g. <code>numpy.log2</code> applied with NaN passthrough) MUST be used; substituting NaN with zero or any sentinel is non-compliant.</td>
    </tr>
</tbody></table>
<br>

### normalized

<table><tbody>
    <tr>
      <th>Key</th>
      <td>normalized</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Post-normalization quantification values. The normalization method MUST be recorded in <code>quantification_provenance.transformations</code> with its CV term and parameters. When the search engine itself produced the normalized quantity (for example DIA-NN's <code>PG.Normalised</code>), the recording step names the engine in <code>software</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate when a normalization step has been applied.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>Dense <code>numpy.ndarray</code> of dtype <code>numpy.float32</code> or <code>numpy.float64</code>, same shape and labels as <code>X</code>. Missing values MUST be <code>NaN</code>.</td>
    </tr>
</tbody></table>
<br>

### imputed

<table><tbody>
    <tr>
      <th>Key</th>
      <td>imputed</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Post-imputation quantification values. No <code>NaN</code> values are permitted in this layer. The imputation method MUST be recorded in <code>quantification_provenance.transformations</code> with its CV term and parameters.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>Dense <code>numpy.ndarray</code> of dtype <code>numpy.float32</code> or <code>numpy.float64</code>, same shape and labels as <code>X</code>. <code>layers['imputed']</code> MUST NOT contain <code>NaN</code>; <code>layers['missingness_mask']</code> MUST be present when <code>layers['imputed']</code> is present so that imputed positions remain identifiable.</td>
    </tr>
</tbody></table>
<br>

### batch_corrected_observed

<table><tbody>
    <tr>
      <th>Key</th>
      <td>batch_corrected_observed</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Batch-corrected quantification values applied only at positions where <code>layers['missingness_mask']</code> is <code>False</code> (observed positions). Imputed positions are preserved as <code>NaN</code> in this layer. The batch-correction method MUST be recorded in <code>quantification_provenance.transformations</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>Dense <code>numpy.ndarray</code> of dtype <code>numpy.float32</code> or <code>numpy.float64</code>, same shape and labels as <code>X</code>. Missing positions MUST be <code>NaN</code>.</td>
    </tr>
</tbody></table>
<br>

### batch_corrected_all

<table><tbody>
    <tr>
      <th>Key</th>
      <td>batch_corrected_all</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Batch-corrected quantification values applied across all positions, including imputed positions. When <code>layers['batch_corrected_all']</code> is present, <code>layers['imputed']</code> SHOULD also be present so that downstream consumers can identify which positions were imputed before batch correction.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>Dense <code>numpy.ndarray</code> of dtype <code>numpy.float32</code> or <code>numpy.float64</code>, same shape and labels as <code>X</code>. No <code>NaN</code> values are permitted.</td>
    </tr>
</tbody></table>
<br>

## `obs` (Sample Metadata)

`obs` is a [`pandas.DataFrame`](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.html). **One row per quantification channel: one `label` within one acquisition (one `.raw` file / one MS run), or within one declared aggregate of acquisitions (the fractions or replicate injections of one sample that were quantified together).** For label-free data the channel is the acquisition itself. This is the granularity of an SDRF-Proteomics row, whose identity is the pair `(comment[data file], comment[label])`. Sample-level metadata (`sample_id`, `donor_id`, `biological_replicate`, `disease`, `organism`, etc.) is **duplicated across the rows of one biological sample**, which lets each channel be independently discoverable. Sample-level aggregation is a downstream analytical step; when performed inside the MIAPE pipeline it MUST be recorded as a `TransformationStep` in `uns.quantification_provenance`.

Submitter columns carry CURIEs in `*_ontology_term_id` fields and are populated by the producer. System columns (the human-readable label fields paired with each CURIE column) are populated by validators after ontology lookup.

The following columns MUST or SHOULD be present in `obs`:

### Granularity: one row per quantification channel

This specification fixes the `obs` granularity at the **quantification channel**, not the biological sample. The identity of a row is the pair `(raw_file, label)`: no two rows of an artifact share the same `raw_file` and `label`, and no raw file appears in more than one row for the same `label`, whether on its own or as a member of an aggregate. The choice is grounded in the proteomics-AnnData ecosystem:

- **Every current producer emits per-channel or per-acquisition rows** (alphapepttools, msmu, apb2, proteopy, QPX, protdata), and the two that model multiplexing (msmu, QPX) key their rows by channel or by run-and-channel.
- **SDRF row semantics** are one row per `(data file, label)`. A channel-level `obs` round-trips to SDRF 1:1; a sample-level `obs` would require an aggregation layer in every consumer.
- **FAIR alignment.** Findable: `raw_file` is a searchable column on every row. Accessible: direct traceability to raw MS data. Interoperable: no adapter needed for existing tools. Reusable: sample-level aggregation is preserved as a downstream `TransformationStep` rather than baked into the schema, so re-analyzers can apply alternative aggregations.

**Consequences:**
- **Multiplexed acquisitions (TMT, iTRAQ, SILAC, plexDIA):** the N channels of one acquisition produce N obs rows sharing the same `raw_file` value with distinct `label` values.
- **Fractionated samples:** either N rows, one per fraction file, each with its own `raw_file` and `fraction` and the same `sample_id`; or one **aggregated** row whose `raw_file` lists the member files `;`-joined and whose `fraction` lists the aligned fraction identifiers, with the aggregation recorded as a `TransformationStep`. A file is never both a standalone row and a member of an aggregate for the same `label`. Engine protein tables are typically per experiment (fractions already combined) while PSM and precursor tables are per file; a producer picks the granularity its data has.
- **Technical replicate injections** follow the same two options as fractions.
- **Sample-level metadata duplicates** across the rows of one biological sample. Downstream tools that want sample-level views can `groupby(['sample_id'])` or use the per-sample aggregation `TransformationStep`.

The polarity of this choice (channel-level vs. sample-level) is **fixed for ratification stability** — changing it would require every existing tool to restructure its reader output and is therefore a breaking change for the entire ecosystem.

### index of pandas.DataFrame

<table><tbody>
    <tr>
      <th>Key</th>
      <td>index of <code>pandas.DataFrame</code></td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Unique identifier for the observation row. Equivalent to the value of <code>obs['assay_id']</code>; the index and the <code>assay_id</code> column MUST be the same. The grammar of the identifier is not fixed: identity is carried by the <code>raw_file</code> and <code>label</code> columns, and two artifacts join on that pair rather than on the index.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The index MUST contain unique identifiers; duplicate values are non-compliant. The index SHOULD equal <code>raw_file</code> for label-free artifacts and <code>raw_file|label</code> for multiplexed artifacts (the composite-key form, see <a href="#delimiters">Delimiters</a>); for an aggregated row the <code>raw_file</code> part is the <code>;</code>-joined member list. Producers that key observations differently (for example by channel alone) keep their index and fill <code>raw_file</code> and <code>label</code>.</td>
    </tr>
</tbody></table>
<br>

### assay_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>assay_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Unique identifier for the observation row. The values in this column MUST be identical to the index of <code>obs</code>. Multiple rows from the same biological sample (fractionation, multiplexing channels carrying the same sample, technical replicates) share the same <code>sample_id</code> but have distinct <code>assay_id</code> values. SDRF <code>assay name</code> is <em>not</em> the source of this column: in SDRF it identifies the data acquisition run and repeats across the channels of a multiplexed run.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Unique per row; same recommended form as the index.</td>
    </tr>
</tbody></table>
<br>

### sample_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>sample_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Identifier for the biological sample from which the observation was derived. Multiple rows from the same biological sample (fractionation, multiplexing channels carrying the same sample, technical replicates) share the same <code>sample_id</code>. This column is the natural <code>groupby</code> key for sample-level aggregation in downstream analysis. Distinct biological samples from the same donor are differentiated by <code>biological_replicate</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Not required to be unique per obs row (multiple observations of the same sample share the value). SDRF source: <code>source name</code> (SDRF Rule 1; the biological sample identifier).</td>
    </tr>
</tbody></table>
<br>

### condition

<table><tbody>
    <tr>
      <th>Key</th>
      <td>condition</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Experimental condition assigned to the sample. When multiple SDRF <code>factor value[*]</code> columns are present, the canonical condition value MUST be the pipe-joined factor values sorted alphabetically by factor term name before joining. Example: SDRF with <code>factor value[treatment]=Drug</code> and <code>factor value[time]=T0</code> yields <code>condition="T0|Drug"</code>. SDRF <code>factor value[*]</code> columns are recommended but not required in the base template stack, so this field is SHOULD-level: when the source SDRF lacks factor-value columns, this field MAY be absent or carry a producer-chosen default (e.g. <code>"unspecified"</code>) and validators emit a WARNING. Producers SHOULD populate this field whenever the experimental design declares factors.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. SDRF source: <code>factor value[*]</code> (SDRF Rule 4), pipe-joined alphabetically.</td>
    </tr>
</tbody></table>
<br>

### organism_ontology_term_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>organism_ontology_term_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Defines the organism from which the assayed biosample was derived.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be an <a href="https://www.ebi.ac.uk/ols4/ontologies/ncbitaxon">NCBI Organismal Classification</a> CURIE such as <code>"NCBITaxon:9606"</code> for <i>Homo sapiens</i>.</td>
    </tr>
</tbody></table>
<br>

### organism

<table><tbody>
    <tr>
      <th>Key</th>
      <td>organism</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Human-readable label for the organism. Populated by validator from <code>organism_ontology_term_id</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>System MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be the human-readable name assigned to the corresponding <code>organism_ontology_term_id</code>.</td>
    </tr>
</tbody></table>
<br>

### assay_ontology_term_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>assay_ontology_term_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Defines the proteomics assay used to acquire the data for this sample.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be an <a href="https://www.ebi.ac.uk/ols4/ontologies/efo/">Experimental Factor Ontology (EFO)</a> CURIE that is a descendant of <code>EFO:0002772</code> for <i>mass spectrometry assay</i>, or a PSI-MS CURIE that names a proteomics acquisition strategy.</td>
    </tr>
</tbody></table>
<br>

### assay

<table><tbody>
    <tr>
      <th>Key</th>
      <td>assay</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Human-readable label for the assay. Populated by validator from <code>assay_ontology_term_id</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>System MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be the human-readable name assigned to the corresponding <code>assay_ontology_term_id</code>.</td>
    </tr>
</tbody></table>
<br>

### tissue_type

<table><tbody>
    <tr>
      <th>Key</th>
      <td>tissue_type</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Categorical type of the biological source. Drives which ontology <code>tissue_ontology_term_id</code> MUST be drawn from.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate when <code>tissue_ontology_term_id</code> is present.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be one of:
        <ul>
          <li><code>"cell culture"</code></li>
          <li><code>"cell line"</code></li>
          <li><code>"organelle"</code></li>
          <li><code>"organoid"</code></li>
          <li><code>"tissue"</code></li>
          <li><code>"other"</code> — for biosamples the producer cannot classify into the above (validators emit WARNING)</li>
          <li><code>"unknown"</code> — for biosamples whose classification is unavailable (validators emit WARNING)</li>
        </ul>
      </td>
    </tr>
</tbody></table>
<br>

### tissue_ontology_term_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>tissue_ontology_term_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Defines the tissue, cell line, organelle, or anatomical entity from which the assayed biosample was derived. The required ontology depends on the value of <code>tissue_type</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>
        <code>str</code>.<br><br>
        If <code>tissue_type</code> is <code>"cell line"</code>, the value MUST be a <a href="https://www.cellosaurus.org/">Cellosaurus</a> identifier such as <code>"CVCL_1P02"</code>.<br><br>
        If <code>tissue_type</code> is <code>"organelle"</code>, the value MUST be a descendant of <a href="https://www.ebi.ac.uk/ols4/ontologies/go/classes?obo_id=GO%3A0005575"><code>GO:0005575</code></a> for <i>cellular_component</i>.<br><br>
        If <code>tissue_type</code> is <code>"tissue"</code> or <code>"organoid"</code>, the value MUST be the most accurate descendant of <a href="https://www.ebi.ac.uk/ols4/ontologies/uberon/classes?obo_id=UBERON%3A0001062"><code>UBERON:0001062</code></a> for <i>anatomical entity</i>.<br><br>
        If <code>tissue_type</code> is <code>"cell culture"</code>, the value MUST be a <a href="https://www.ebi.ac.uk/ols4/ontologies/cl/">Cell Ontology (CL)</a> term.
      </td>
    </tr>
</tbody></table>
<br>

### tissue

<table><tbody>
    <tr>
      <th>Key</th>
      <td>tissue</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Human-readable label for the tissue, cell line, organelle, or anatomical entity. Populated by validator from <code>tissue_ontology_term_id</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>System SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be the human-readable name assigned to the corresponding <code>tissue_ontology_term_id</code>.</td>
    </tr>
</tbody></table>
<br>

### disease_ontology_term_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>disease_ontology_term_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Defines the disease state of the patient or organism from which the assayed biosample was derived. Required for human samples to match the <code>human/1.1.0</code> SDRF template (<code>characteristics[disease]</code>) and the <code>clinical-metadata/1.0.0</code> SDRF template; recommended for other organisms.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate when <code>organism_ontology_term_id</code> is <a href="https://www.ebi.ac.uk/ols4/ontologies/ncbitaxon/classes?obo_id=NCBITaxon%3A9606"><code>"NCBITaxon:9606"</code></a> for <i>Homo sapiens</i>; SHOULD annotate otherwise.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>
        <code>str</code>. The value MUST be one of:
        <ul>
          <li><a href="https://www.ebi.ac.uk/ols4/ontologies/pato/classes?obo_id=PATO%3A0000461"><code>"PATO:0000461"</code></a> for <i>normal</i> or <i>healthy</i></li>
          <li>the most accurate descendant of <a href="https://www.ebi.ac.uk/ols4/ontologies/mondo/classes?obo_id=MONDO%3A0000001"><code>"MONDO:0000001"</code></a> for <i>disease</i></li>
          <li><a href="https://www.ebi.ac.uk/ols4/ontologies/mondo/classes?obo_id=MONDO%3A0021178"><code>"MONDO:0021178"</code></a> for <i>injury</i> or, preferably, its most accurate descendant</li>
        </ul>
      </td>
    </tr>
</tbody></table>
<br>

### disease

<table><tbody>
    <tr>
      <th>Key</th>
      <td>disease</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Human-readable label for the disease state. Populated by validator from <code>disease_ontology_term_id</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>System MUST annotate when <code>disease_ontology_term_id</code> is present (i.e. when <code>organism_ontology_term_id</code> is <code>"NCBITaxon:9606"</code> for <i>Homo sapiens</i>, or whenever a submitter has supplied the CURIE for other organisms); otherwise this field SHOULD be absent.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be the human-readable name assigned to the corresponding <code>disease_ontology_term_id</code>.</td>
    </tr>
</tbody></table>
<br>

### development_stage_ontology_term_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>development_stage_ontology_term_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Defines the developmental stage of the patient or organism from which the assayed biosample was derived. The required ontology depends on the value of <code>organism_ontology_term_id</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>
        <code>str</code>.<br><br>
        If <code>tissue_type</code> is <code>"cell line"</code>, the value MUST be <code>"na"</code>.<br><br>
        If the developmental stage is unknown, the value MUST be <code>"unknown"</code>.<br><br>
        Otherwise:
        <table>
          <thead>
            <tr>
              <th>For the corresponding<br><code>organism_ontology_term_id</code></th>
              <th>Value</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td><a href="https://www.ebi.ac.uk/ols4/ontologies/ncbitaxon/classes?obo_id=NCBITaxon%3A9606"><code>"NCBITaxon:9606"</code></a><br>for <i>Homo sapiens</i></td>
              <td>The most accurate descendant of <a href="https://www.ebi.ac.uk/ols4/ontologies/hsapdv/classes?obo_id=HsapDv%3A0000001"><code>HsapDv:0000001</code></a> for <i>life cycle</i></td>
            </tr>
            <tr>
              <td><a href="https://www.ebi.ac.uk/ols4/ontologies/ncbitaxon/classes?obo_id=NCBITaxon%3A10090"><code>"NCBITaxon:10090"</code></a><br>for <i>Mus musculus</i> or one of its descendants</td>
              <td>The most accurate descendant of <a href="https://www.ebi.ac.uk/ols4/ontologies/mmusdv/classes?obo_id=MmusDv%3A0000001"><code>MmusDv:0000001</code></a> for <i>life cycle</i></td>
            </tr>
            <tr>
              <td>For all other organisms</td>
              <td>The most accurate descendant of <a href="https://www.ebi.ac.uk/ols4/ontologies/uberon/classes?obo_id=UBERON%3A0000105"><code>UBERON:0000105</code></a> for <i>life cycle stage</i>, excluding <a href="https://www.ebi.ac.uk/ols4/ontologies/uberon/classes?obo_id=UBERON%3A0000071"><code>UBERON:0000071</code></a> for <i>death stage</i></td>
            </tr>
          </tbody>
        </table>
      </td>
    </tr>
</tbody></table>
<br>

### development_stage

<table><tbody>
    <tr>
      <th>Key</th>
      <td>development_stage</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Human-readable label for the developmental stage. Populated by validator from <code>development_stage_ontology_term_id</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>System SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td>
        <code>str</code>.<br><br>
        The value MUST be <code>"na"</code> if <code>development_stage_ontology_term_id</code> is <code>"na"</code>.<br><br>
        The value MUST be <code>"unknown"</code> if <code>development_stage_ontology_term_id</code> is <code>"unknown"</code>.<br><br>
        Otherwise, the value MUST be the human-readable name assigned to the corresponding <code>development_stage_ontology_term_id</code>.
      </td>
    </tr>
</tbody></table>
<br>

### donor_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>donor_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Pseudonymous identifier for the subject or individual from which the biosample was derived. Used to link biological replicates from the same individual. MUST NOT contain personally identifiable information.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Producer-chosen pseudonymous identifier. SDRF source: <code>characteristics[individual]</code>.</td>
    </tr>
</tbody></table>
<br>

### biological_replicate

<table><tbody>
    <tr>
      <th>Key</th>
      <td>biological_replicate</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Biological-replicate identifier. Samples sharing a <code>donor_id</code> and a <code>biological_replicate</code> value are the same biological replicate; samples with the same <code>donor_id</code> but different <code>biological_replicate</code> values are distinct biological replicates drawn from the same individual (separate cultures, animals or tissue pieces). Repeated measurements of the same biological replicate are technical replicates, see <code>technical_replicate</code>. Required in the universally-inherited <code>sample-metadata</code> SDRF template (<code>characteristics[biological replicate]</code>), so MUST per the Derivation Principle.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Pattern accepts integer values as strings (<code>"1"</code>, <code>"2"</code>, <code>"3"</code>, ...) or the literal <code>"pooled"</code> for pooled biological replicates. SDRF source: <code>characteristics[biological replicate]</code>.</td>
    </tr>
</tbody></table>
<br>

### sex

<table><tbody>
    <tr>
      <th>Key</th>
      <td>sex</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Biological sex of the donor.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. RECOMMENDED values are <code>"female"</code>, <code>"male"</code>, <code>"unknown"</code>, or <code>"na"</code> (for cell lines or non-applicable contexts). SDRF source: <code>characteristics[sex]</code>.</td>
    </tr>
</tbody></table>
<br>

### age

<table><tbody>
    <tr>
      <th>Key</th>
      <td>age</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Age of the donor at the time of biosample collection. Free-text; not validated. For ontology-bound developmental staging, use <code>development_stage_ontology_term_id</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Free-text. SDRF source: <code>characteristics[age]</code>.</td>
    </tr>
</tbody></table>
<br>

### raw_file

<table><tbody>
    <tr>
      <th>Key</th>
      <td>raw_file</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>The mass-spectrometry raw-data file (or files) this row was quantified from. One file name for a single acquisition; for an aggregated row (fractions or replicate injections quantified together) the member file names <code>;</code>-joined in a stable order. Together with <code>label</code> this column is the identity of the row: no two rows MUST share the same <code>(raw_file, label)</code> pair, and no file name MUST appear, alone or as a member of an aggregate, in more than one row for the same <code>label</code>. When any value of this column holds more than one member, a <code>TransformationStep</code> recording the aggregation MUST be present in <code>quantification_provenance</code> (its <code>software</code> is the engine when the engine combined the fractions). Required in the <code>base</code> SDRF template (<code>comment[data file]</code>), so MUST per the Derivation Principle.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. File name as written in the SDRF (<code>comment[data file]</code>), with or without extension but consistently within an artifact; members of an aggregate <code>;</code>-joined, no whitespace. SDRF source: <code>comment[data file]</code>.</td>
    </tr>
</tbody></table>
<br>

### batch

<table><tbody>
    <tr>
      <th>Key</th>
      <td>batch</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Producer's canonical single batch identifier for downstream batch-correction methods (e.g. ComBat, LOWESS, removeBatchEffect). Samples sharing a batch value were acquired or processed together in a way that may introduce systematic bias. Designed as a drop-in covariate for downstream tools that expect a single categorical batch variable per sample. When the source SDRF carries both <code>comment[sample preparation batch]</code> and <code>comment[lc batch]</code>, the producer combines them into a single <code>batch</code> label (e.g. <code>"prepA_lcA"</code>); the granular SDRF columns are preserved separately in <code>prep_batch</code> and <code>lc_batch</code>. When only one SDRF batch column applies, <code>batch</code> mirrors its value.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Producer-chosen batch label. SDRF source: producer-supplied combination of <code>comment[sample preparation batch]</code> and / or <code>comment[lc batch]</code>; both source columns are themselves preserved as <code>prep_batch</code> and <code>lc_batch</code> respectively.</td>
    </tr>
</tbody></table>
<br>

### prep_batch

<table><tbody>
    <tr>
      <th>Key</th>
      <td>prep_batch</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Sample-preparation batch identifier. Verbatim of the SDRF <code>comment[sample preparation batch]</code> column when present. Preserved separately from <code>batch</code> so that re-analyses can isolate sample-prep batch effects from LC batch effects. For multiplexed designs this is also where a labelled mixture (plex) identifier lands when the SDRF records it as the sample-preparation batch.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate. Present when the source SDRF provides <code>comment[sample preparation batch]</code>; absent otherwise.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Free-text identifier. SDRF source: <code>comment[sample preparation batch]</code>. When both <code>prep_batch</code> and <code>batch</code> are present, <code>batch</code> SHOULD encode <code>prep_batch</code> (possibly combined with <code>lc_batch</code>); validators emit WARNING when they disagree.</td>
    </tr>
</tbody></table>
<br>

### lc_batch

<table><tbody>
    <tr>
      <th>Key</th>
      <td>lc_batch</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>LC-batch identifier. Verbatim of the SDRF <code>comment[lc batch]</code> column when present. Preserved separately from <code>batch</code> so that re-analyses can isolate LC batch effects (column changes, instrument drift) from sample-prep batch effects.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate. Present when the source SDRF provides <code>comment[lc batch]</code>; absent otherwise.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Free-text identifier. SDRF source: <code>comment[lc batch]</code>. When both <code>lc_batch</code> and <code>batch</code> are present, <code>batch</code> SHOULD encode <code>lc_batch</code> (possibly combined with <code>prep_batch</code>); validators emit WARNING when they disagree.</td>
    </tr>
</tbody></table>
<br>

### technical_replicate

<table><tbody>
    <tr>
      <th>Key</th>
      <td>technical_replicate</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Technical-replicate identifier: a repeated measurement (injection or preparation) of the same biological replicate. Rows sharing <code>sample_id</code> and <code>biological_replicate</code> and differing only in <code>technical_replicate</code> are technical replicates. Distinct biological samples from the same individual are encoded with <code>biological_replicate</code>, not here. Renamed from <code>replicate</code> in v0.5.0 to match the SDRF column and to pair with <code>biological_replicate</code>. Required in the <code>base</code> SDRF template (<code>comment[technical replicate]</code>), so MUST per the Derivation Principle. For an aggregated row whose member acquisitions were different technical replicates, the value is the <code>;</code>-joined list of their identifiers.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Integer values as strings starting from <code>"1"</code>. SDRF source: <code>comment[technical replicate]</code>.</td>
    </tr>
</tbody></table>
<br>

### is_pool

<table><tbody>
    <tr>
      <th>Key</th>
      <td>is_pool</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Derived convenience boolean: <code>True</code> if the sample is a QC pool (a mixture of multiple biological samples used for technical normalization or batch correction). Pool samples are used to compute pool-injection CVs and SHOULD NOT be included in biological analyses without explicit handling. <strong>Validator-derived from <code>sample_type</code>:</strong> <code>is_pool = (sample_type == "pooled")</code>. The source of truth is <code>sample_type</code>; this column is the derived convenience boolean for downstream filtering like <code>adata[~adata.obs.is_pool]</code>. Producers MUST NOT populate this field directly; validators overwrite any producer-supplied value to enforce consistency with <code>sample_type</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>System MUST annotate. The validator computes <code>is_pool = (sample_type == "pooled")</code> from the submitter-provided <code>sample_type</code> during artifact validation. The producer's responsibility is to populate <code>sample_type</code> (which mirrors SDRF <code>comment[sample type]</code>); <code>is_pool</code> is derived.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>bool</code>. Derived from <code>sample_type</code>; the SDRF source for the underlying classification is <code>comment[sample type]</code> with the value <code>"pooled"</code>.</td>
    </tr>
</tbody></table>
<br>

### sample_type

<table><tbody>
    <tr>
      <th>Key</th>
      <td>sample_type</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Categorical classification of the sample, mirroring the SDRF <code>comment[sample type]</code> convention. Distinguishes various non-biological sample classes (pools, blanks, references, controls) from straightforward biological experimental samples, and for multiplexed designs the role of a channel (reference, bridge, carrier, empty). When this field is absent or unset, the sample is assumed to be a biological experimental sample. Samples whose <code>sample_type</code> is set to any of the QC / control classes below SHOULD be excluded from biological analyses by default and reserved for QC, normalization, or instrument-tracking workflows.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate when the sample is anything other than a straightforward biological experimental sample; otherwise this field MAY be absent. Validators MUST NOT require this field for biological samples.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. RECOMMENDED values, aligned exactly with the SDRF <code>sample-metadata.yaml</code> <code>comment[sample type]</code> PRIDE-CV enumeration (canonical CURIEs in the PSI-MS <code>MS:1003027</code> family; pending exact CURIE registration, the spec lists string values):
        <ul>
          <li><code>"single cell"</code></li>
          <li><code>"reference"</code></li>
          <li><code>"bridge"</code></li>
          <li><code>"carrier"</code></li>
          <li><code>"pooled"</code></li>
          <li><code>"empty"</code></li>
          <li><code>"quality control sample"</code></li>
          <li><code>"negative control"</code></li>
          <li><code>"positive control"</code></li>
          <li><code>"bulk control"</code></li>
          <li><code>"standard"</code></li>
          <li><code>"calibrator"</code></li>
          <li><code>"plate control"</code></li>
          <li><code>"buffer control"</code></li>
        </ul>
        SDRF source: <code>comment[sample type]</code>. Producer-chosen values are acceptable for sample types not yet covered by the recommended list, but validators emit a WARNING because producer-chosen values do not interoperate with downstream tools that filter on this column. Note: the SDRF base template has since added <code>characteristics[sample type]</code> with a PRIDE enumeration; reconciling the source column and the value list with it is a pending editorial item.</td>
    </tr>
</tbody></table>
<br>

### passed_qc

<table><tbody>
    <tr>
      <th>Key</th>
      <td>passed_qc</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>True if the sample passed the producer's quality-control criteria. Producers MUST document their QC criteria in <code>uns['quantification_provenance']</code> (e.g. as a <code>TransformationStep</code> with method-specific parameters).</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>System SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>bool</code>.</td>
    </tr>
</tbody></table>
<br>

### qc_fail_reason

<table><tbody>
    <tr>
      <th>Key</th>
      <td>qc_fail_reason</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>If <code>passed_qc</code> is <code>False</code>, free-text reason for failure (e.g. "low MS2 spectrum count", "outlier in median intensity"). MUST NOT be present when <code>passed_qc</code> is <code>True</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>System MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Free-text.</td>
    </tr>
</tbody></table>
<br>

### injection_order

<table><tbody>
    <tr>
      <th>Key</th>
      <td>injection_order</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Index of the sample in the LC-MS injection sequence. Used for time-drift detection and run-order batch effects.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. 1-indexed.</td>
    </tr>
</tbody></table>
<br>

### label

<table><tbody>
    <tr>
      <th>Key</th>
      <td>label</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>The quantification channel of this row. For multiplexed acquisitions (TMT, iTRAQ, SILAC, mTRAQ, dimethyl) the channel label; for label-free acquisitions the constant <code>"LFQ"</code>. Together with <code>raw_file</code> this column is the identity of the row. Values are drawn from the canonical channel vocabulary of the sdrf-pipelines channel map, reproduced in <a href="#appendix-e-label-vocabulary">Appendix E</a>; the SDRF ontology name <code>label free sample</code> and key-value spellings such as <code>NT=TMT126;AC=PRIDE:0000114</code> are normalised to the canonical label on ingest. Aligns with SDRF <code>comment[label]</code>, which is REQUIRED in every <code>ms-proteomics/1.1.0</code> row including LFQ.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate in ms-proteomics-derived artifacts. The field MAY be absent in non-MS-proteomics artifacts (e.g. affinity-proteomics) where labeling is not applicable; the row identity is then <code>raw_file</code> alone.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be a bare label with no <code>NT=</code> / <code>AC=</code> wrapper that matches one of the six family patterns of Appendix E (<code>LFQ</code>; <code>TMT</code> reporter; <code>ITRAQ</code> reporter; <code>SILAC light|medium|heavy</code>; <code>MTRAQ</code>; <code>DIMETHYL</code>), and it SHOULD be a member of the canonical list in Appendix E (a non-member of the right family is a WARNING, so a newly released plex does not invalidate an artifact). Every value MUST belong to the family declared by <code>uns['labeling_type']</code>; when <code>labeling_type</code> is <code>"other"</code> the pattern requirement does not apply. SDRF source: <code>comment[label]</code>.</td>
    </tr>
</tbody></table>
<br>

### fraction

<table><tbody>
    <tr>
      <th>Key</th>
      <td>fraction</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Fraction identifier of the acquisition this row was quantified from. For an aggregated row the fraction identifiers of the member files, <code>;</code>-joined in the same order as <code>raw_file</code>. Required in the <code>ms-proteomics/1.1.0</code> SDRF template (<code>comment[fraction identifier]</code>, an integer, <code>1</code> for unfractionated data); MIAPE-AnnData specifies SHOULD here so that producers without the SDRF-side requirement (e.g. affinity-proteomics) remain compliant.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. One identifier per member of <code>raw_file</code>, <code>;</code>-joined and aligned with it; validators warn when the member counts differ. SDRF source: <code>comment[fraction identifier]</code>.</td>
    </tr>
</tbody></table>
<br>

### instrument

<table><tbody>
    <tr>
      <th>Key</th>
      <td>instrument</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Mass spectrometer or affinity-platform identifier used to acquire the sample. RECOMMENDED to use a <a href="https://www.psidev.info/groups/controlled-vocabularies">PSI-MS</a> CURIE for the instrument model. Required in the <code>ms-proteomics/1.1.0</code> SDRF template (<code>comment[instrument]</code>) and in the <code>affinity-proteomics/1.0.0</code> SDRF template (<code>comment[platform]</code>).</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Free-text or PSI-MS CURIE. SDRF source: <code>comment[instrument]</code>.</td>
    </tr>
</tbody></table>
<br>

### acquisition_method

<table><tbody>
    <tr>
      <th>Key</th>
      <td>acquisition_method</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Acquisition mode used to sample the mass spectrometer. Distinguishes data-dependent (DDA), data-independent (DIA), and targeted (PRM, SRM) acquisitions, which materially affect how downstream tools should handle the quantification. Required in the <code>ms-proteomics/1.1.0</code> SDRF template (<code>comment[proteomics data acquisition method]</code>); MIAPE-AnnData specifies SHOULD per the Derivation Principle since affinity-proteomics consumers do not need this column.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. RECOMMENDED values, aligned with the SDRF <code>ms-proteomics/1.1.0</code> PRIDE-CV enumeration:
        <ul>
          <li><code>"data-dependent acquisition"</code> (DDA)</li>
          <li><code>"data-independent acquisition"</code> (DIA)</li>
          <li><code>"parallel reaction monitoring"</code> (PRM)</li>
          <li><code>"selected reaction monitoring"</code> (SRM)</li>
        </ul>
        Producer-chosen values are acceptable for emerging acquisition strategies (e.g. <code>"plexDIA"</code>, <code>"BoxCar"</code>) not yet in the CV. SDRF source: <code>comment[proteomics data acquisition method]</code>.</td>
    </tr>
</tbody></table>
<br>

### enzyme

<table><tbody>
    <tr>
      <th>Key</th>
      <td>enzyme</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Proteolytic enzyme (cleavage agent) used to digest the sample. RECOMMENDED to use a <a href="https://www.psidev.info/groups/controlled-vocabularies">PSI-MS</a> CURIE for the cleavage agent. Two PSI-MS terms cover preparations without a specific enzyme and MUST be used for them rather than a free-text paraphrase: <code>unspecific cleavage</code> (<code>MS:1001956</code>) for non-enzymatic or immunopeptidomic preparations, and <code>no cleavage</code> (<code>MS:1001955</code>) for intact or top-down measurements. An SDRF value of <code>not applicable</code> passes through unchanged. Required in the <code>ms-proteomics/1.1.0</code> SDRF template (<code>comment[cleavage agent details]</code>).</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Free-text or PSI-MS CURIE. SDRF source: <code>comment[cleavage agent details]</code>.</td>
    </tr>
</tbody></table>
<br>

## `var` (Feature Metadata)

`var` carries per-feature metadata. For single-modality `.h5ad` artifacts, `var` is a single [`pandas.DataFrame`](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.html). For multi-modality `.h5mu` artifacts, each modality has its own `var` DataFrame accessed via `mdata.mod['<level>'].var`. The following subsections specify `var` columns per modality: protein (v0.1.0), peptide (v0.2.0), peptidoform (v0.3.0), site (v0.4.0) and peptidoform ion (v0.5.0).

Three rules hold for every modality:

- **Indices are grammar-free; identity lives in columns.** Every `var` index MUST be unique and MUST equal the modality's key column (`protein_group`, `sequence`, `peptidoform`, `peptidoform_ion`, `site_id`). The schema does not fix the grammar of the index string; the information a consumer needs to join two artifacts is carried in normalized columns that are specified well enough to validate (`protein_ids`, `leading_protein`; `protein_id`, `residue_positions`, `modification`). Producers keep the key they already write.
- **Every non-protein modality links upward.** A peptide, peptidoform, peptidoform-ion or site modality MUST carry at least one foreign-key column that resolves into a coarser modality present in the artifact (`protein_group`, `sequence` or `peptidoform`), so that a `.h5mu` with any subset of modalities is navigable. Foreign keys to a modality that is absent from the artifact are MAY and are not validated.
- **No decoys.** Decoy features MUST NOT be present (see General Requirements).

**Per-feature confidence.** Every modality's `var` SHOULD carry `q_value`: a float in `[0.0, 1.0]`, the **experiment-wide** q-value of the feature at its own level, as reported by the engine or recomputed by the producer by target-decoy competition. It is the evidence that the FDR threshold recorded as a `TransformationStep` was applied to. Per-run q-values and posterior error probabilities are matrices, not columns, and belong to the `DEFERRED` typed score layers. The PSI-MS term that types the value per modality:

| Modality | `q_value` is the | PSI-MS term |
|---|---|---|
| protein | protein group q-value at the experiment level (DIA-NN `Global.PG.Q.Value`, MaxQuant protein `Q-value`) | `protein group-level q-value` |
| peptide | q-value of the distinct peptide sequence | `distinct peptide-level q-value` |
| peptidoform | q-value of the peptidoform, where the engine computes one | `distinct peptide-level q-value` (no peptidoform-specific term exists) |
| peptidoform_ion | experiment-level precursor q-value (DIA-NN `Global.Q.Value`) | `PSM-level q-value` when the engine reports the PSM-level estimate; otherwise untyped |
| site | q-value over site-modification claims, counted per site row | none in PSI-MS; QPX's `site_qvalue` is the reference definition |

### Protein modality

The protein modality's `var` carries one row per protein group. Protein groups are the output of the search engine's protein-inference step; a protein group is a set of indistinguishable or subsumed proteins represented by one or more accessions. The group's identity for joining purposes is its member list, `protein_ids`; the index string itself is whatever the producer writes.

### index of pandas.DataFrame

<table><tbody>
    <tr>
      <th>Key</th>
      <td>index of <code>pandas.DataFrame</code></td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Unique identifier for the protein group. Equivalent to the value of <code>var['protein_group']</code>; the index and the <code>protein_group</code> column MUST be the same. The grammar is not fixed: a producer MAY use the engine's group string verbatim, its own member-list form, or any other unique string. Joins between artifacts are made on <code>protein_ids</code>, not on the index.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Unique per row.</td>
    </tr>
</tbody></table>
<br>

### protein_group

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_group</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Identifier for the protein group. The values in this column MUST be identical to the index of <code>var</code>, and it is the value that the <code>protein_group</code> foreign key of every other modality points at. Engine-native column names (<code>Protein IDs</code>, <code>Protein.Group</code>, <code>PG.ProteinGroups</code>, <code>Protein ID</code>) MAY be preserved as additional non-canonical columns, but the canonical column name <code>protein_group</code> MUST be present.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Unique per row; grammar not fixed.</td>
    </tr>
</tbody></table>
<br>

### protein_ids

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_ids</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>The member accessions of the protein group, as written in the sequence database recorded in <code>uns['search_database']</code>. This is the normalized identity of the group and the column on which two artifacts join. Tool-added prefixes that mark decoys or contaminants (<code>rev_</code>, <code>REV__</code>, <code>CON__</code>, <code>Cont_</code>) are not part of an accession and MUST NOT appear in a token; FASTA header fields other than the accession (<code>sp|P12345|NAME_HUMAN</code>) MUST be reduced to the accession. An accession MAY carry an isoform suffix (<code>P12345-2</code>); the schema neither requires nor forbids isoform resolution, and two artifacts join on tokens only when both resolve isoforms the same way (the representative rule is a working-group item).</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. One or more accessions, <code>;</code>-joined, no whitespace, every token non-empty (e.g. <code>"P12345"</code>, <code>"P12345;P67890"</code>). Each token SHOULD be a <a href="https://www.uniprot.org/">UniProtKB</a> accession, optionally with an isoform suffix; validators check the UniProtKB pattern and WARN on tokens that do not match (custom databases in immunopeptidomics or metaproteomics have no UniProt accessions).</td>
    </tr>
</tbody></table>
<br>

### leading_protein

<table><tbody>
    <tr>
      <th>Key</th>
      <td>leading_protein</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>One representative accession of the group, for consumers that need a single protein per row (gene annotation, sequence retrieval, PTM site positions). The engine's leading or razor protein where the engine names one; otherwise producer-chosen.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. When present, MUST be one of the tokens of <code>protein_ids</code>.</td>
    </tr>
</tbody></table>
<br>

### gene_name

<table><tbody>
    <tr>
      <th>Key</th>
      <td>gene_name</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Gene symbol associated with the protein group. For multi-accession protein groups, the value SHOULD be the symbol of <code>leading_protein</code> or a semicolon-separated list aligned with <code>protein_ids</code>. Not validated when <code>is_contaminant</code> is <code>True</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. For human samples, SHOULD be an HGNC-approved symbol; for other organisms, SHOULD be the species-canonical gene symbol.</td>
    </tr>
</tbody></table>
<br>

### organism

<table><tbody>
    <tr>
      <th>Key</th>
      <td>organism</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Species to which the protein group belongs. For non-contaminant protein groups, the value SHOULD match the sample's <code>obs.organism</code>; mismatches indicate cross-species contamination or hybrid datasets.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value SHOULD be the human-readable species name corresponding to an NCBITaxon CURIE.</td>
    </tr>
</tbody></table>
<br>

### is_contaminant

<table><tbody>
    <tr>
      <th>Key</th>
      <td>is_contaminant</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>True if the protein group originates from a contamination database (e.g. cRAP, iRT peptides, MaxQuant common contaminants). Contaminant protein groups SHOULD be excluded from biological analyses but are preserved in the artifact for QC and provenance.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>bool</code>.</td>
    </tr>
</tbody></table>
<br>

### contaminant_source

<table><tbody>
    <tr>
      <th>Key</th>
      <td>contaminant_source</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Identifier of the contaminant database that flagged the row as a contaminant. Disambiguates which contaminant list was used; different lists (cRAP, MaxQuant common contaminants, FragPipe <code>contaminants_2022.fas</code>) overlap but disagree on membership, so the same UniProt accession may be flagged by one list and not by another. MUST be empty or absent when <code>is_contaminant</code> is <code>False</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate when <code>is_contaminant</code> is <code>True</code>.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. RECOMMENDED values are well-known list names: <code>"cRAP"</code>, <code>"MaxQuant common"</code>, <code>"FragPipe contaminants_2022"</code>, or a producer-chosen identifier matching the side database recorded in <code>uns['search_database']</code>.</td>
    </tr>
</tbody></table>
<br>

### protein_description

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_description</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Free-text description of the protein group, typically the UniProt protein name or FASTA header description of <code>leading_protein</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Free-text.</td>
    </tr>
</tbody></table>
<br>

### n_peptides_used

<table><tbody>
    <tr>
      <th>Key</th>
      <td>n_peptides_used</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Number of distinct peptide sequences aggregated into this protein group during the engine's peptide-to-protein-group aggregation step. Used for confidence weighting and for evaluating protein-level quantification quality.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. Non-negative.</td>
    </tr>
</tbody></table>
<br>

### detection_rate

<table><tbody>
    <tr>
      <th>Key</th>
      <td>detection_rate</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Fraction of observations in which the protein group was detected (i.e. had a non-NaN value in <code>layers['raw']</code>). Useful for filtering protein groups with low coverage prior to downstream analysis.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>.</td>
    </tr>
</tbody></table>
<br>

### q_value

<table><tbody>
    <tr>
      <th>Key</th>
      <td>q_value</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Experiment-wide q-value of the protein group (PSI-MS <code>protein group-level q-value</code>), as reported by the engine (DIA-NN <code>Global.PG.Q.Value</code>, MaxQuant <code>Q-value</code>) or recomputed by the producer. See <a href="#var-feature-metadata">Per-feature confidence</a>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>; <code>NaN</code> where no estimate exists for the row.</td>
    </tr>
</tbody></table>
<br>

### cv_pools

<table><tbody>
    <tr>
      <th>Key</th>
      <td>cv_pools</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Coefficient of variation of the protein group's intensities across QC-pool injections (observations with <code>obs.is_pool == True</code>). Used as a per-protein technical reproducibility metric.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Non-negative.</td>
    </tr>
</tbody></table>
<br>

### median_intensity

<table><tbody>
    <tr>
      <th>Key</th>
      <td>median_intensity</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Median of the protein group's non-NaN intensities across all biological (non-pool, non-blank) observations in <code>layers['raw']</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>.</td>
    </tr>
</tbody></table>
<br>

### Peptide modality

The peptide modality's `var` carries one row per **bare amino acid sequence** — a peptide without modification or charge state. In the QuantifiedEntity ontology this corresponds to `Peptide` (as distinct from `Peptidoform`, which retains modification state). Bare-sequence peptide quantifications are produced by tools that collapse modification states either implicitly (during peptide-to-protein-group aggregation) or explicitly (via a `ModificationStateToBasePeptide` aggregation step recorded in `quantification_provenance.transformations`).

Two protein columns are kept apart on every peptide-level row (peptide, peptidoform and peptidoform ion): `protein_ids`, the accessions the peptide **maps to**, and `protein_group`, the group the engine **assigned** it to for quantification. Engines that infer no groups at the level they emit (FragPipe ion tables, PEAKS, Sage, MaxQuant's peptidoform table) and endpoint-peptide experiments with no inference step (immunopeptidomics, peptidomics) carry `protein_ids` without `protein_group`.

### index of pandas.DataFrame

<table><tbody>
    <tr>
      <th>Key</th>
      <td>index of <code>pandas.DataFrame</code></td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Unique identifier for the peptide (the bare amino acid sequence). Equivalent to the value of <code>var['sequence']</code>; the index and the <code>sequence</code> column MUST be the same.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Unique per row.</td>
    </tr>
</tbody></table>
<br>

### sequence

<table><tbody>
    <tr>
      <th>Key</th>
      <td>sequence</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Bare amino acid sequence (no modifications, no charge state) in uppercase one-letter notation. The values in this column MUST be identical to the index of <code>var</code>. Engine-native column names (<code>Peptide</code>, <code>Sequence</code>, <code>Stripped.Sequence</code>) MAY be preserved as additional non-canonical columns, but the canonical column name <code>sequence</code> MUST be present.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Uppercase one-letter amino acid sequence (e.g. <code>"PEPTIDEK"</code>). No spaces, no modifications, no charge state, no flanking residue notation.</td>
    </tr>
</tbody></table>
<br>

### protein_group

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_group</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Foreign key into the protein-modality <code>var</code> index: the protein group to which the engine assigned this peptide for quantification. For shared peptides the value reflects the engine's protein-inference decision — see <code>is_unique</code> and the protein-inference <code>TransformationStep</code> recorded in <code>quantification_provenance</code> for the inference method used. Null when the engine assigned the peptide to no group.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate when the artifact carries a protein modality (nullable); MAY annotate otherwise.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code> or null. Every non-null value MUST exactly match a value present in the protein modality's <code>var.protein_group</code> when the artifact carries a protein modality.</td>
    </tr>
</tbody></table>
<br>

### protein_ids

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_ids</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>All protein accessions this peptide sequence maps to in the sequence database recorded in <code>uns['search_database']</code>, independent of any protein-inference decision (DIA-NN <code>Protein.Ids</code>; FragPipe <code>Protein</code> joined with <code>Mapped Proteins</code>; MaxQuant <code>Proteins</code>). Same token rules as the protein modality's <code>protein_ids</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Accessions <code>;</code>-joined, no whitespace, every token non-empty.</td>
    </tr>
</tbody></table>
<br>

### is_unique

<table><tbody>
    <tr>
      <th>Key</th>
      <td>is_unique</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>True if the peptide is specific to its assigned protein group: every token of <code>protein_ids</code> is a member of the <code>protein_ids</code> of the group named in <code>protein_group</code>. False if the peptide also maps to proteins outside that group (a shared peptide). A proteotypic peptide, one that maps to a single protein, needs no column: it is <code>protein_ids</code> having one token. Validators check the definition when <code>protein_ids</code>, <code>protein_group</code> and the protein modality are all present. Tools that emit only group-specific peptides set this to <code>True</code> for every row.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>bool</code>.</td>
    </tr>
</tbody></table>
<br>

### missed_cleavages

<table><tbody>
    <tr>
      <th>Key</th>
      <td>missed_cleavages</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Number of missed enzymatic cleavages in the peptide. For tryptic digests, computed as the count of K/R residues internal to the sequence (excluding the C-terminus). Used as a quality / digestion-completeness metric. Not meaningful when <code>obs.enzyme</code> is <code>unspecific cleavage</code> or <code>no cleavage</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. Non-negative.</td>
    </tr>
</tbody></table>
<br>

### length

<table><tbody>
    <tr>
      <th>Key</th>
      <td>length</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Length of the peptide sequence in residues. Equal to <code>len(sequence)</code>; recorded as a column for efficient filtering.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. Positive.</td>
    </tr>
</tbody></table>
<br>

### n_psms

<table><tbody>
    <tr>
      <th>Key</th>
      <td>n_psms</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Number of peptide-spectrum matches (PSMs) aggregated into this peptide row across all observations in the artifact. Used for confidence weighting and for evaluating peptide-level quantification quality.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. Non-negative.</td>
    </tr>
</tbody></table>
<br>

### detection_rate

<table><tbody>
    <tr>
      <th>Key</th>
      <td>detection_rate</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Fraction of observations in which the peptide was detected (had a non-NaN value in <code>layers['raw']</code>). Useful for filtering peptides with low coverage prior to downstream analysis.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>.</td>
    </tr>
</tbody></table>
<br>

### q_value

<table><tbody>
    <tr>
      <th>Key</th>
      <td>q_value</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Experiment-wide q-value of the peptide (PSI-MS <code>distinct peptide-level q-value</code>). See <a href="#var-feature-metadata">Per-feature confidence</a>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>; <code>NaN</code> where no estimate exists for the row.</td>
    </tr>
</tbody></table>
<br>

### Peptidoform modality

The peptidoform modality's `var` carries one row per **peptidoform** — a peptide together with a fully specified modification state, charge-collapsed. In the QuantifiedEntity ontology this corresponds to `Peptidoform`. The peptidoform is what most search engines actually emit when their output refers to "peptide-level": the same bare sequence with different modification states (e.g. unmodified vs. phospho-S5 vs. phospho-S5+S7) appears as distinct peptidoform rows.

Peptidoforms are encoded in [PSI ProForma 2.1](https://www.psidev.info/proforma) notation (Final, 2026-06-09), which represents the sequence with bracketed modifications at residue positions and at the termini. ProForma 2.1 is organised in compliance levels; the index requires **Level 1** (names, full accessions, delta masses, terminal modifications), MAY use the Level 2 unknown-position form, and does not use the Level 3 global fixed-modification form. Examples:

* `EMEVEESPEK` — unmodified bare sequence (also valid as a Peptide row)
* `EM[UNIMOD:35]EVEES[UNIMOD:21]PEK` — methionine oxidation (UNIMOD:35) and phospho-serine (UNIMOD:21), full accession form
* `[UNIMOD:1]-EMEVEESPEK` — N-terminal acetylation; terminal modifications are represented here, at the peptidoform level
* `EM[+15.9949]EVEES[+79.9663]PEK` — the same peptidoform in mass-delta form, valid Level 1 but not joinable to the accession form
* `[UNIMOD:35]?EMEVEESPEK` — oxidation at an unknown position (Level 2)

The index MUST be a valid Level 1-ProForma 2.1 string in any modification form that level allows; vendor strings that are not ProForma (`_EM(Oxidation (M))EVEESPEK_`, `EM(UniMod:35)EVEESPEK`) MUST NOT be the index and MAY be preserved as non-canonical columns. The full accession form (`UNIMOD:N` / `MOD:N`) SHOULD be used, because it is the only form that joins across engines and is ProForma's own recommendation (§6.2.3: delta masses only when the modification cannot be named); validators WARN on other forms. The ontology-abbreviation shorthand (`U:35`, `M:00719`) is not valid ProForma 2.1 (§6.2.2) and MUST NOT appear. Fixed modifications are written per residue. One modification ontology (UNIMOD or PSI-MOD) is used per artifact.

### index of pandas.DataFrame

<table><tbody>
    <tr>
      <th>Key</th>
      <td>index of <code>pandas.DataFrame</code></td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Unique identifier for the peptidoform, a Level 1-ProForma 2.1 string. Equivalent to the value of <code>var['peptidoform']</code>; the index and the <code>peptidoform</code> column MUST be the same.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Unique per row. Valid Level 1-ProForma 2.1; full accession form SHOULD.</td>
    </tr>
</tbody></table>
<br>

### peptidoform

<table><tbody>
    <tr>
      <th>Key</th>
      <td>peptidoform</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>ProForma 2.1 string identifying the peptidoform. The values in this column MUST be identical to the index of <code>var</code>. Engine-native columns that encode the modification state differently (msmu's <code>peptide</code> string-encoded mods; MaxQuant's <code>Modified sequence</code> with bracketed mod abbreviations; FragPipe's <code>Modified Peptide</code>) MAY be preserved as non-canonical columns, but the canonical column name <code>peptidoform</code> MUST be present and MUST carry the ProForma form.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Level 1-ProForma 2.1 (e.g. <code>"EM[UNIMOD:35]EVEES[UNIMOD:21]PEK"</code>).</td>
    </tr>
</tbody></table>
<br>

### sequence

<table><tbody>
    <tr>
      <th>Key</th>
      <td>sequence</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Foreign key into the peptide-modality <code>var</code> index. The bare amino acid sequence obtained by stripping all modifications from <code>peptidoform</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Uppercase one-letter amino acid sequence; MUST exactly match a value present in the peptide modality's <code>var.sequence</code> when the artifact is multi-modality and includes the peptide modality.</td>
    </tr>
</tbody></table>
<br>

### protein_group

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_group</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Foreign key into the protein-modality <code>var</code> index: the protein group to which the engine assigned this peptidoform for quantification. Same semantics and conditions as the peptide modality's <code>protein_group</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate when the artifact carries a protein modality (nullable); MAY annotate otherwise.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code> or null. Every non-null value MUST exactly match a value present in the protein modality's <code>var.protein_group</code> when the artifact carries a protein modality.</td>
    </tr>
</tbody></table>
<br>

### protein_ids

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_ids</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>All protein accessions this peptidoform's sequence maps to. Same semantics and token rules as the peptide modality's <code>protein_ids</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Accessions <code>;</code>-joined, no whitespace, every token non-empty.</td>
    </tr>
</tbody></table>
<br>

### modifications

<table><tbody>
    <tr>
      <th>Key</th>
      <td>modifications</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Structured list of the modifications present on this peptidoform. Redundant with the encoded modifications in <code>peptidoform</code>; provided as a separate column for queryability and to allow downstream tools to filter peptidoforms by modification type without parsing ProForma strings.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>List[Tuple[int, str]]</code> serialized as a string (e.g. JSON-encoded list of <code>[position, curie]</code> pairs). Each tuple is <code>(position, curie)</code> where <code>position</code> is the 1-indexed residue position (<code>0</code> for N-terminal, <code>length + 1</code> for C-terminal) and <code>curie</code> is the UNIMOD or PSI-MOD CURIE for the modification (e.g. <code>"UNIMOD:21"</code> for phosphorylation). Example: <code>"[[2, \"UNIMOD:35\"], [6, \"UNIMOD:21\"]]"</code> for the example peptidoform <code>EM[UNIMOD:35]EVEES[UNIMOD:21]PEK</code>.</td>
    </tr>
</tbody></table>
<br>

### is_unique

<table><tbody>
    <tr>
      <th>Key</th>
      <td>is_unique</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>True if the peptidoform is specific to its assigned protein group, with the same checkable definition as the peptide modality's <code>is_unique</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>bool</code>.</td>
    </tr>
</tbody></table>
<br>

### n_psms

<table><tbody>
    <tr>
      <th>Key</th>
      <td>n_psms</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Number of peptide-spectrum matches (PSMs) aggregated into this peptidoform row across all observations in the artifact.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. Non-negative.</td>
    </tr>
</tbody></table>
<br>

### detection_rate

<table><tbody>
    <tr>
      <th>Key</th>
      <td>detection_rate</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Fraction of observations in which the peptidoform was detected (had a non-NaN value in <code>layers['raw']</code>).</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>.</td>
    </tr>
</tbody></table>
<br>

### q_value

<table><tbody>
    <tr>
      <th>Key</th>
      <td>q_value</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Experiment-wide q-value of the peptidoform where the engine computes one. See <a href="#var-feature-metadata">Per-feature confidence</a>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>; <code>NaN</code> where no estimate exists for the row.</td>
    </tr>
</tbody></table>
<br>

### Peptidoform ion modality

The peptidoform ion modality's `var` carries one row per **peptidoform ion** — a peptidoform in a specific charge state, the entity commonly called a precursor. PSI-MS names it `peptidoform ion` (`MS:1003051`, "commonly called a 'precursor' or 'precursor ion'") and defines its string form as ProForma including the charge state (`MS:1003270`). In the QuantifiedEntity ontology this corresponds to `Precursor`. Most DIA engines and several DDA pipelines quantify at this level, and immunopeptidomics needs this level and nothing coarser. The modality is **MAY**: a producer that quantifies at precursor level MAY emit it instead of, or alongside, the peptidoform modality. There is no PSM modality: one spectrum in one run makes a cross-run PSM matrix near-empty.

### index of pandas.DataFrame

<table><tbody>
    <tr>
      <th>Key</th>
      <td>index of <code>pandas.DataFrame</code></td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Unique identifier for the peptidoform ion: a Level 1-ProForma 2.1 peptidoform string followed by the charge suffix <code>/z</code> (ProForma 2.1 §11.5, a Level 3 feature; the spec names it so that readers know a Level 1 parser alone does not cover the suffix). Equivalent to the value of <code>var['peptidoform_ion']</code>; the index and the <code>peptidoform_ion</code> column MUST be the same.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Unique per row, e.g. <code>"EM[UNIMOD:35]EVEES[UNIMOD:21]PEK/2"</code>. The peptidoform part follows the same rules as the peptidoform modality's index (full accession form SHOULD).</td>
    </tr>
</tbody></table>
<br>

### peptidoform_ion

<table><tbody>
    <tr>
      <th>Key</th>
      <td>peptidoform_ion</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>The ProForma-with-charge string identifying the peptidoform ion. The values in this column MUST be identical to the index of <code>var</code>. Engine-native precursor identifiers (DIA-NN <code>Precursor.Id</code>, Spectronaut <code>EG.PrecursorId</code>) MAY be preserved as non-canonical columns.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. ProForma 2.1 peptidoform plus <code>/z</code>.</td>
    </tr>
</tbody></table>
<br>

### charge

<table><tbody>
    <tr>
      <th>Key</th>
      <td>charge</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Precursor charge state. Redundant with the <code>/z</code> suffix of the index; provided as a typed column for filtering.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. Positive; MUST equal the <code>/z</code> suffix of <code>peptidoform_ion</code>.</td>
    </tr>
</tbody></table>
<br>

### peptidoform

<table><tbody>
    <tr>
      <th>Key</th>
      <td>peptidoform</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Foreign key into the peptidoform-modality <code>var</code> index: the charge-collapsed peptidoform of this ion (the index string without the <code>/z</code> suffix).</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate when the artifact carries a peptidoform modality; SHOULD annotate otherwise.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. MUST exactly match a value present in the peptidoform modality's <code>var.peptidoform</code> when that modality is present.</td>
    </tr>
</tbody></table>
<br>

### sequence

<table><tbody>
    <tr>
      <th>Key</th>
      <td>sequence</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Foreign key into the peptide-modality <code>var</code> index: the bare amino acid sequence of this ion.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate when the artifact carries a peptide modality and no peptidoform modality; SHOULD annotate otherwise.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. MUST exactly match a value present in the peptide modality's <code>var.sequence</code> when that modality is present.</td>
    </tr>
</tbody></table>
<br>

### protein_group

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_group</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Foreign key into the protein-modality <code>var</code> index: the protein group to which the engine assigned this ion for quantification. Same semantics and conditions as the peptide modality's <code>protein_group</code>. At least one of <code>peptidoform</code>, <code>sequence</code> and <code>protein_group</code> MUST be present and resolve into a coarser modality present in the artifact.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate when the artifact carries a protein modality (nullable); MAY annotate otherwise.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code> or null. Every non-null value MUST exactly match a value present in the protein modality's <code>var.protein_group</code> when the artifact carries a protein modality.</td>
    </tr>
</tbody></table>
<br>

### protein_ids

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_ids</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>All protein accessions this ion's sequence maps to. Same semantics and token rules as the peptide modality's <code>protein_ids</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Accessions <code>;</code>-joined, no whitespace, every token non-empty.</td>
    </tr>
</tbody></table>
<br>

### is_unique

<table><tbody>
    <tr>
      <th>Key</th>
      <td>is_unique</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>True if the ion is specific to its assigned protein group, with the same checkable definition as the peptide modality's <code>is_unique</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>bool</code>.</td>
    </tr>
</tbody></table>
<br>

### n_psms

<table><tbody>
    <tr>
      <th>Key</th>
      <td>n_psms</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Number of peptide-spectrum matches aggregated into this ion row across all observations in the artifact.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. Non-negative.</td>
    </tr>
</tbody></table>
<br>

### detection_rate

<table><tbody>
    <tr>
      <th>Key</th>
      <td>detection_rate</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Fraction of observations in which the ion was detected (had a non-NaN value in <code>layers['raw']</code>).</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>.</td>
    </tr>
</tbody></table>
<br>

### q_value

<table><tbody>
    <tr>
      <th>Key</th>
      <td>q_value</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Experiment-wide q-value of the precursor (DIA-NN <code>Global.Q.Value</code>; Spectronaut <code>EG.Qvalue</code> at the experiment level). Per-run precursor q-values are matrices and belong to the <code>DEFERRED</code> typed score layers. See <a href="#var-feature-metadata">Per-feature confidence</a>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>; <code>NaN</code> where no estimate exists for the row.</td>
    </tr>
</tbody></table>
<br>

### Site modality

The site modality's `var` carries one row per **PTM-localized site claim** — one modification type at one residue of one protein, or, for a **composite** row, at a set of co-modified residues of one protein that were quantified together because the engine measured them on one multiply modified peptidoform. In the QuantifiedEntity ontology a single-residue row corresponds to `Site`; a co-modified site set has no CV entity yet (working-group item). Site rows are the unit of PTM-localized quantification referenced by phosphoproteomics, ubiquitinomics, glycoproteomics and acetylation workflows.

Two phenomena are kept apart. **Localization ambiguity** (one modification that is at S12 *or* S25) is represented by candidate rows, one per candidate position, each with its `localization_prob`, and filtered by the class I convention. **Co-modification** (two modifications, at S12 *and* S25, both present) is one composite row with both positions in `residue_positions`. A composite MUST NOT be used to express ambiguity, or a consumer cannot tell "either" from "both". Splitting a doubly modified peptidoform into two single-site rows would assert two independent events and double-count the intensity, so a producer that does not want composites keeps such peptidoforms at the peptidoform level instead.

The `site_id` index is grammar-free; identity lives in `protein_id`, `residue_positions` and `modification`. The index SHOULD take the form `{protein_accession_CURIE}|{residue_position}|{modification_CURIE}` for a single-residue site, e.g. `UniProtKB:P11802|794|UNIMOD:21`, which adapts the PSI [PEFF](https://www.psidev.info/peff) `\ModResPsi` syntax into a row key. The `;` list delimiter MUST NOT appear inside a `site_id` field, so that the one list delimiter keeps one meaning; the identifier format for composites is a working-group item, and a producer's own composite key (for example msmu's `P1|S12_S25`) is a valid index. Legacy string forms MAY appear as a secondary non-canonical column.

`DEFERRED`: **terminal modifications** (protein N-terminal acetylation, C-terminal amidation) have no residue position and cannot be placed in `residue_positions` in this draft; they are represented at the peptidoform level (`[UNIMOD:1]-PEPTIDE`). The second pass chooses between sentinel positions (`0` and `length + 1`, QPX's convention) and a `terminus` column after checking how the engine site reports encode them.

The site modality MUST be present when PTM-site analysis is part of the artifact's intended use. When a site modality is present alongside a peptidoform modality, the `peptidoform` foreign key SHOULD be populated so downstream tools can trace which peptidoform localized each site.

### index of pandas.DataFrame

<table><tbody>
    <tr>
      <th>Key</th>
      <td>index of <code>pandas.DataFrame</code></td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Unique identifier for the site row. Equivalent to the value of <code>var['site_id']</code>; the index and the <code>site_id</code> column MUST be the same. Grammar not fixed; the recommended single-site form is above.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Unique per row.</td>
    </tr>
</tbody></table>
<br>

### site_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>site_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Site identifier. The values in this column MUST be identical to the index of <code>var</code>. SHOULD follow <code>{protein_accession_CURIE}|{residue_position}|{modification_CURIE}</code> for single-residue sites (e.g. <code>"UniProtKB:P11802|794|UNIMOD:21"</code>), whose three fields are self-describing under <code>split('|')</code>. The <code>;</code> character MUST NOT appear in the identifier. The modification ontology (UNIMOD or PSI-MOD) MUST be consistent across all <code>site_id</code> values within a single artifact.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Unique per row.</td>
    </tr>
</tbody></table>
<br>

### protein_id

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_id</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>The one protein accession whose sequence the positions in <code>residue_positions</code> are numbered against. This is the accession the site was localized on (msmu: the peptide's own accession; MaxQuant: the leading protein of the site table), and it answers the residue-position provenance question: positions are meaningful only against a named sequence. When the artifact carries a protein modality, the value MUST be a token of the <code>protein_ids</code> of the group named in <code>protein_group</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. One accession, same token rules as <code>protein_ids</code> (no tool prefixes, optional isoform suffix).</td>
    </tr>
</tbody></table>
<br>

### protein_group

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_group</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Foreign key into the protein-modality <code>var</code> index: the protein group on which the site is localized. Same semantics and conditions as the peptide modality's <code>protein_group</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate when the artifact carries a protein modality (nullable); MAY annotate otherwise.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code> or null. Every non-null value MUST exactly match a value present in the protein modality's <code>var.protein_group</code> when the artifact carries a protein modality.</td>
    </tr>
</tbody></table>
<br>

### peptidoform

<table><tbody>
    <tr>
      <th>Key</th>
      <td>peptidoform</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Foreign key into the peptidoform-modality <code>var</code> index identifying the peptidoform that localized this site. MAY be populated; the linkage is ambiguous when multiple peptidoforms support the same site (different missed-cleavage variants, different co-occurring modifications). When ambiguous, the value MAY be a <code>;</code>-separated list of peptidoform identifiers, or MAY be omitted entirely with the multi-peptidoform support recorded in <code>n_localizing_peptidoforms</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Single ProForma 2.1 peptidoform identifier, or <code>;</code>-separated list of identifiers; each MUST exactly match a value present in the peptidoform modality's <code>var.peptidoform</code> when the artifact is multi-modality and includes the peptidoform modality.</td>
    </tr>
</tbody></table>
<br>

### residue_positions

<table><tbody>
    <tr>
      <th>Key</th>
      <td>residue_positions</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>The 1-indexed position, or positions, in the sequence of <code>protein_id</code> at which the modification is localized: one position for an ordinary site, two or more for a composite row of co-modified residues. The position is on the protein sequence (not on the peptidoform sequence) so downstream tools can map directly to the database sequence. A list column replaces v0.4.0's integer <code>residue_position</code> so that every position of a composite is first-class and no null is needed; a null position would read as "unlocalized", which is a different thing. Validators check that the tokens are distinct positive integers and that their count equals <code>n_sites</code> when present.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. One or more positive integers, <code>;</code>-joined, ascending, e.g. <code>"794"</code> or <code>"12;25"</code>.</td>
    </tr>
</tbody></table>
<br>

### n_sites

<table><tbody>
    <tr>
      <th>Key</th>
      <td>n_sites</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Number of residues the row covers: <code>1</code> for an ordinary site, the token count of <code>residue_positions</code> for a composite. Lets single-site consumers select <code>n_sites == 1</code> and cast one token to an integer.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. Positive; MUST equal the token count of <code>residue_positions</code> when present.</td>
    </tr>
</tbody></table>
<br>

### residue

<table><tbody>
    <tr>
      <th>Key</th>
      <td>residue</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Single-letter amino acid code(s) at <code>residue_positions</code>, aligned with it: <code>"S"</code> for an ordinary site, <code>"S;S"</code> for a composite. Redundant with the protein sequence; provided for efficient filtering by residue type.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Uppercase one-letter codes, <code>;</code>-joined, same count as <code>residue_positions</code>.</td>
    </tr>
</tbody></table>
<br>

### modification

<table><tbody>
    <tr>
      <th>Key</th>
      <td>modification</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>CURIE identifying the modification type localized at this site. RECOMMENDED to use a <a href="https://www.unimod.org/">UNIMOD</a> CURIE; the <a href="https://www.psidev.info/MOD">PSI-MOD</a> CURIE is also acceptable. A composite row carries one modification type that applies to every position in <code>residue_positions</code>; mixed modification types in one row are out of scope.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. CURIE of the form <code>UNIMOD:N</code> or <code>MOD:N</code>.</td>
    </tr>
</tbody></table>
<br>

### protein_positions

<table><tbody>
    <tr>
      <th>Key</th>
      <td>protein_positions</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Other proteins that carry the same site through a shared peptide, with the position in each of their sequences, for consumers that need every carrier rather than the one in <code>protein_id</code>. The pairing character <code>:</code> keeps <code>;</code> a plain list delimiter.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. <code>accession:position</code> pairs, <code>;</code>-joined, e.g. <code>"P1:12;P2:47"</code>; for a composite each pair lists the positions of that carrier in the same order as <code>residue_positions</code>, joined with <code>_</code> (e.g. <code>"P2:47_60"</code>).</td>
    </tr>
</tbody></table>
<br>

### localization_prob

<table><tbody>
    <tr>
      <th>Key</th>
      <td>localization_prob</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Engine-reported PTM site localization probability — the probability that the modification is localized at the stated position (as opposed to a different position on the same peptidoform). Sources include FragPipe PTM-Prophet, OpenMS PhosphoScoring / LuciPhor2 / AScore, MaxQuant localization probability. For a composite row the value is the smallest per-site probability. Producers SHOULD propagate the engine's value rather than discard it; v0.4.0's "MUST when the engine emits it" is not mechanically checkable and is replaced by this guidance.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate (with <code>NaN</code> when no probability is available).</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>. <code>NaN</code> when the engine did not emit a probability for this site.</td>
    </tr>
</tbody></table>
<br>

### is_class_I

<table><tbody>
    <tr>
      <th>Key</th>
      <td>is_class_I</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Convention flag for high-confidence PTM site localization. Computed by validator from <code>localization_prob</code>: <code>True</code> when <code>localization_prob &gt; 0.75</code>, <code>False</code> otherwise. The 0.75 threshold matches the <a href="https://doi.org/10.1038/nmeth.2557">Olsen / Mann class-I PTM convention</a>; tools that use a different threshold MAY override this column with the engine-specific value.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>System MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>bool</code>. <code>NaN</code> permitted when <code>localization_prob</code> is <code>NaN</code>.</td>
    </tr>
</tbody></table>
<br>

### n_localizing_peptidoforms

<table><tbody>
    <tr>
      <th>Key</th>
      <td>n_localizing_peptidoforms</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Number of distinct peptidoforms in the source data that supported localization of this site. Used as a per-site evidence-strength metric. When the site modality is present without the peptidoform modality, the value is informational; when both are present, the value SHOULD equal the count of peptidoform foreign keys in <code>peptidoform</code> for this site (or the count of <code>varp['feature_mapping']</code> edges from this site row to peptidoform rows).</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MAY annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>int</code>. Positive.</td>
    </tr>
</tbody></table>
<br>

### q_value

<table><tbody>
    <tr>
      <th>Key</th>
      <td>q_value</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>q-value over the site-modification claims of the artifact, counted per site row: the estimated proportion of false claims among the rows at least as well supported. A claim is false when the residue does not carry this modification, whatever the reason. Distinct from <code>localization_prob</code>, which is the position probability of a claim taken as true. PSI-MS has no site-level q-value term; QPX's <code>site_qvalue</code> is the reference definition. See <a href="#var-feature-metadata">Per-feature confidence</a>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>float</code>. Range <code>[0.0, 1.0]</code>; <code>NaN</code> where no estimate exists for the row.</td>
    </tr>
</tbody></table>
<br>

## MuData Composition

When an artifact carries quantifications at more than one modality level, it MUST be packaged as a single `.h5mu` file using the [MuData specification](https://mudata.readthedocs.io/). MuData stores multiple AnnData objects under named modality keys and provides shared observation-axis alignment via the MuData-level `obs`.

### Modality keys

Modality keys in a MIAPE-AnnData v0.5.0 `.h5mu` MUST be drawn from the following enumerated set; producer-chosen keys are non-compliant.

| Modality key | Modality |
|---|---|
| `protein` | Protein-group level (as specified in §`var`/Protein modality) |
| `peptide` | Bare-sequence peptide level (as specified in §`var`/Peptide modality) |
| `peptidoform` | Modification-aware peptidoform level (as specified in §`var`/Peptidoform modality) |
| `peptidoform_ion` | Precursor level, peptidoform plus charge (as specified in §`var`/Peptidoform ion modality) |
| `site` | PTM-localized site level (as specified in §`var`/Site modality) |

Any subset of two or more keys is permitted provided every non-protein modality present links upward to a coarser modality present (§`var`). A `.h5mu` carrying every modality records the chain of adjacencies in `varp['feature_mapping']`: peptide ↔ protein, peptidoform ↔ peptide, peptidoform_ion ↔ peptidoform, site ↔ peptidoform, plus any transitive edges the producer chooses to materialise (peptidoform ↔ protein, site ↔ protein). Tools that consume the artifact via `mdata.link` can navigate multi-hop relationships (e.g. site → peptidoform → protein) using mulink's transitive-closure queries.

### Sample alignment

Per-modality `obs` DataFrames within a `.h5mu` MUST share the same index — `mdata.mod['protein'].obs.index` and `mdata.mod['peptide'].obs.index` MUST be element-wise equal, and so for every pair of modalities present. This fixes one observation granularity per artifact: a producer whose precursor table is per acquisition file and whose protein table is per aggregated sample either aggregates the finer level to the coarser rows (recording the step) or ships two artifacts. MuData's `obs` join policy MUST be `"inner"` so the MuData-level `obs` reflects the shared observation set without ambiguity. Producers MAY additionally populate the MuData-level `obs` with the same metadata fields specified in [`obs` (Sample Metadata)](#obs-sample-metadata); when MuData-level `obs` is present, it MUST agree with each per-modality `obs` on shared keys.

### `uns` placement

The MuData-level `mdata.uns` MUST carry the dataset-wide provenance and version fields:

* `mdata.uns['miape_anndata_spec']` — `"0.5.0"`.
* `mdata.uns['level']` — `"multi"`.
* `mdata.uns['linkage_convention']` — see [`uns.linkage_convention`](#linkage_convention).
* `mdata.uns['quantification_provenance']` — cross-modality operations (e.g. parsimony protein inference that produced both modalities from a shared PSM source) MUST be recorded here, as a JSON string per [`uns.quantification_provenance`](#quantification_provenance). Per-modality operations (e.g. log2-transforming the protein matrix specifically) are recorded in `mdata.mod['<level>'].uns['quantification_provenance']`.

All other `uns` fields specified in §[`uns` (Dataset Metadata)](#uns-dataset-metadata) MAY be populated at either the MuData level, the per-modality level, or both; when both are present they MUST agree.

## `varp` (Cross-modality feature linkage)

When an artifact carries multiple modalities, it MUST record the relationships between features at different modality levels so downstream tools can navigate from a finer row to its parent feature and vice versa.

### feature_mapping

<table><tbody>
    <tr>
      <th>Key</th>
      <td>feature_mapping</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Sparse boolean adjacency matrix over the concatenated <code>mdata.var_names</code> of all modalities present, encoding which finer features belong to which coarser features. Rows and columns follow MuData's concatenation order of the modalities, which SHOULD run coarse to fine (<code>protein</code>, <code>peptide</code>, <code>peptidoform</code>, <code>peptidoform_ion</code>, <code>site</code>); a <code>True</code> entry at position <code>(i, j)</code> means feature <code>i</code> and feature <code>j</code> are related by the engine's assignment (peptide-to-protein-group, peptidoform-to-peptide, ion-to-peptidoform, site-to-peptidoform). Producers MAY materialise the transitive closure (e.g. site ↔ protein) so that multi-hop queries are single lookups; consumers MUST NOT assume closure is present.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate on multi-modality <code>.h5mu</code> artifacts (conventions 1 and 2 below); MUST NOT be present on single-modality <code>.h5ad</code> artifacts.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>scipy.sparse.csr_matrix</code> of <code>bool</code> dtype. Square, shape <code>(Σ n_features, Σ n_features)</code> over the modalities present. Symmetric (the adjacency is undirected). Producers MAY also store equivalent finer → coarser adjacency directly under <code>mdata.mod['<level>'].varm[...]</code> as a producer-side convenience, but the canonical storage MUST be the <code>varp['feature_mapping']</code> matrix.</td>
    </tr>
</tbody></table>
<br>

### Storage convention

The cross-modality adjacency MAY be stored via any of the following three conventions, listed in **decreasing order of preference**. The chosen convention MUST be recorded in `mdata.uns['linkage_convention']`.

1. **`mulink.link` accessor (RECOMMENDED for new artifacts).** Producers attach the adjacency via `mdata.link.add_link(...)`, which writes a sparse boolean matrix to `mdata.varp['feature_mapping']` and exposes transitive-closure queries through the `.link` accessor API. This convention composes natively with scverse multi-omics tooling (scanpy, scvi-tools). Set `mdata.uns['linkage_convention'] = "mulink"`.
2. **Direct `varp['feature_mapping']` (ALLOWED for backwards compatibility).** Producers write the sparse boolean adjacency directly to `mdata.varp['feature_mapping']` without the mulink accessor. On-disk shape is equivalent to convention 1. Set `mdata.uns['linkage_convention'] = "varp"`.
3. **`uns['peptide_map']` / `uns['protein_map']` dict storage (DISCOURAGED for new artifacts; accepted for `msmu` interop).** Producers encode the relationships as Python dicts in `mdata.uns`. This convention does not compose with mulink's `.link` accessor; downstream tools MUST be informed via `mdata.uns['linkage_convention'] = "uns_dict"` so they know to query the dicts rather than the `varp` matrix.

A compliant `.h5mu` artifact SHOULD use convention 1 or 2. A declared key join alone (the foreign-key columns of §`var` without any of the three storage forms) is not a convention: the foreign keys are required in any case, and the adjacency MUST additionally be materialised in one of the forms above.

## `uns` (Dataset Metadata)

`uns` is a [`dict`](https://docs.python.org/3/library/stdtypes.html#mapping-types-dict) of dataset-level metadata. A compliant artifact MUST carry the fields below in `uns`.

### miape_anndata_spec

<table><tbody>
    <tr>
      <th>Key</th>
      <td>miape_anndata_spec</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Semantic version of the MIAPE-AnnData specification that this artifact claims compliance with. A v0.5.0-compliant artifact MUST set this to <code>"0.5.0"</code>. Profiles of this specification (subsets that relax named sections for a particular consumer) do not change this value; they carry their own version in their own documentation.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Semver string. For artifacts produced under this version of the specification, the value MUST be <code>"0.5.0"</code>.</td>
    </tr>
</tbody></table>
<br>

### search_engine

<table><tbody>
    <tr>
      <th>Key</th>
      <td>search_engine</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Identifier for the search engine or analysis tool that produced the quantifications in this artifact.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. RECOMMENDED values are lowercase tool short names: <code>"fragpipe"</code>, <code>"maxquant"</code>, <code>"dia-nn"</code>, <code>"sage"</code>, <code>"openms"</code>, <code>"spectronaut"</code>, <code>"msfragger"</code>, <code>"comet"</code>, <code>"msgfplus"</code>, <code>"alphadia"</code>, <code>"peaks"</code>.</td>
    </tr>
</tbody></table>
<br>

### search_engine_version

<table><tbody>
    <tr>
      <th>Key</th>
      <td>search_engine_version</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Version string of the search engine identified in <code>search_engine</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. As reported by the tool (e.g. <code>"22.0"</code>, <code>"1.8.1"</code>).</td>
    </tr>
</tbody></table>
<br>

### search_database

<table><tbody>
    <tr>
      <th>Key</th>
      <td>search_database</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Identifier of the protein sequence database (FASTA) against which the engine searched the spectra. The accessions in every <code>protein_ids</code>, <code>protein_id</code> and <code>leading_protein</code> value are written as they appear in this database. The FASTA is a post-search artifact known at write time; recording it here makes the artifact reproducible against the same database in the future. Required for PRIDE / ProteomeXchange-style submission. The value SHOULD identify the database uniquely enough to retrieve it (e.g. <code>"UP000005640_9606_2024_03.fasta"</code>, <code>"swissprot_human_2024_03"</code>, <code>"contaminants_2022.fas"</code> for a contamination-only side database).</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Producer-chosen identifier — filename, UniProt proteome ID, or local database name.</td>
    </tr>
</tbody></table>
<br>

### loader

<table><tbody>
    <tr>
      <th>Key</th>
      <td>loader</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Identifier for the loader package or library that read the engine's output and serialized the artifact.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. Lowercase package name (e.g. <code>"alphapepttools"</code>, <code>"msmu"</code>, <code>"apb2"</code>, <code>"protdata"</code>, <code>"proteopy"</code>, <code>"qpx"</code>).</td>
    </tr>
</tbody></table>
<br>

### loader_version

<table><tbody>
    <tr>
      <th>Key</th>
      <td>loader_version</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Version string of the loader package identified in <code>loader</code>.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. As reported by the loader's package metadata.</td>
    </tr>
</tbody></table>
<br>

### sdrf_file

<table><tbody>
    <tr>
      <th>Key</th>
      <td>sdrf_file</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Filename of the SDRF-Proteomics file used to populate <code>obs</code>. Informational; the SDRF file itself is not embedded in the artifact.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>.</td>
    </tr>
</tbody></table>
<br>

### loaded_at

<table><tbody>
    <tr>
      <th>Key</th>
      <td>loaded_at</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>ISO 8601 date on which the artifact was first serialized.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter SHOULD annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. ISO 8601 date format <code>"YYYY-MM-DD"</code>.</td>
    </tr>
</tbody></table>
<br>

### labeling_type

<table><tbody>
    <tr>
      <th>Key</th>
      <td>labeling_type</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Labeling chemistry used in the quantification. Captures only the labeling chemistry; the acquisition mode (DDA / DIA / PRM / SRM) is recorded separately in <code>obs.acquisition_method</code>. Composite acquisition × labeling methods are encoded as the orthogonal combination of <code>labeling_type</code> and <code>obs.acquisition_method</code>. Examples: <strong>plexDIA</strong> = <code>labeling_type=mTRAQ</code> (or <code>dimethyl</code>) + <code>acquisition_method=data-independent acquisition</code>; <strong>DIA-SILAC</strong> = <code>labeling_type=SILAC</code> + <code>acquisition_method=data-independent acquisition</code>; <strong>mTRAQ-PRM</strong> = <code>labeling_type=mTRAQ</code> + <code>acquisition_method=parallel reaction monitoring</code>. Every <code>obs.label</code> value MUST belong to the family this value declares (<a href="#appendix-e-label-vocabulary">Appendix E</a>); <code>"other"</code> exempts <code>label</code> from the pattern check.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be one of:
        <ul>
          <li><code>"LFQ"</code> — label-free quantification</li>
          <li><code>"TMT"</code> — Tandem Mass Tag isobaric labeling (TMT and TMTpro)</li>
          <li><code>"iTRAQ"</code> — iTRAQ isobaric labeling</li>
          <li><code>"SILAC"</code> — Stable Isotope Labeling by Amino acids in cell Culture</li>
          <li><code>"mTRAQ"</code> — mTRAQ non-isobaric labeling</li>
          <li><code>"dimethyl"</code> — dimethyl labeling</li>
          <li><code>"other"</code> — labeling chemistry not covered by the above</li>
        </ul>
      </td>
    </tr>
</tbody></table>
<br>

### level

<table><tbody>
    <tr>
      <th>Key</th>
      <td>level</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Modality level of the artifact. For a single-modality <code>.h5ad</code>, the value identifies the one modality present; for a multi-modality <code>.h5mu</code>, the MuData-level <code>uns['level']</code> is <code>"multi"</code> and each per-modality <code>mdata.mod['&lt;k&gt;'].uns['level']</code> identifies its own modality.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be one of:
        <ul>
          <li><code>"protein"</code> — single-modality protein-group artifact</li>
          <li><code>"peptide"</code> — single-modality bare-sequence peptide artifact</li>
          <li><code>"peptidoform"</code> — single-modality peptidoform artifact</li>
          <li><code>"peptidoform_ion"</code> — single-modality precursor artifact</li>
          <li><code>"site"</code> — single-modality PTM-localized site artifact</li>
          <li><code>"multi"</code> — multi-modality MuData artifact (only at <code>mdata.uns['level']</code>; per-modality <code>uns['level']</code> MUST be one of the modality-specific values above)</li>
        </ul>
      </td>
    </tr>
</tbody></table>
<br>

### linkage_convention

<table><tbody>
    <tr>
      <th>Key</th>
      <td>linkage_convention</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Identifier of the cross-modality feature-linkage storage convention used by this artifact. Required on multi-modality <code>.h5mu</code> artifacts (set at the MuData level); MUST NOT be present on single-modality <code>.h5ad</code> artifacts.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate on multi-modality <code>.h5mu</code> artifacts.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>. The value MUST be one of:
        <ul>
          <li><code>"mulink"</code> — adjacency stored via <code>mdata.link.add_link(...)</code> (RECOMMENDED)</li>
          <li><code>"varp"</code> — adjacency stored directly in <code>mdata.varp['feature_mapping']</code> without the mulink accessor</li>
          <li><code>"uns_dict"</code> — adjacency stored as <code>uns['peptide_map']</code> / <code>uns['protein_map']</code> dicts (DISCOURAGED; accepted for <code>msmu</code> interop)</li>
        </ul>
      </td>
    </tr>
</tbody></table>
<br>

### quantification_provenance

<table><tbody>
    <tr>
      <th>Key</th>
      <td>quantification_provenance</td>
    </tr>
    <tr>
      <th>Description</th>
      <td>Ordered, structured record of the transformations applied to the quantification data from raw engine output through the layers present in this artifact. Each transformation step is bound to a Quantification CV term, names its software, and records its parameters. <strong>Container.</strong> The value is a single JSON-encoded string, always, under this fixed key: AnnData and MuData cannot write a list of dicts to HDF5, and a JSON document is what the structural schema validates directly. Readers decode with <code>json.loads</code>; writers encode with <code>json.dumps</code>. No marker that the value is JSON is needed, because the schema says so for this key.</td>
    </tr>
    <tr>
      <th>Annotator</th>
      <td>Submitter MUST annotate.</td>
    </tr>
    <tr>
      <th>Value</th>
      <td><code>str</code>: UTF-8 JSON text decoding to a JSON object. The object MUST contain a key <code>"transformations"</code> whose value is a non-empty JSON array; each element MUST conform to the <a href="#transformationstep-schema"><code>TransformationStep</code> schema</a> below. Other top-level keys MAY be present and are ignored by validators. Example: <code>'{"transformations": [{"step": 1, "accession": "QCMS:0000005", ...}]}'</code>.</td>
    </tr>
</tbody></table>
<br>

#### TransformationStep schema

Each element of `transformations` is a JSON object recording one processing step. The fields below MUST or SHOULD be present in each element. **Additional keys on a step are permitted and ignored by the validator**, so producers with richer provenance (input and output hashes, environment records, event identifiers) can carry it alongside the canonical fields.

<table>
  <thead>
    <tr>
      <th>Field</th>
      <th>Description</th>
      <th>Annotator</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>step</code></td>
      <td>1-based step index. Steps MUST be contiguous and ordered, and MUST agree with the element's position in the array. <code>DEFERRED</code>: the second pass decides whether the field stays, given that array position already carries the order, and whether MAY <code>id</code> / <code>inputs</code> fields express a DAG of steps.</td>
      <td>Submitter MUST annotate.</td>
      <td><code>int</code></td>
    </tr>
    <tr>
      <td><code>accession</code></td>
      <td>Quantification CV term CURIE identifying the transformation method. Until the Quantification CV has a release with accessions, the placeholder form <code>PLACEHOLDER:&lt;name&gt;</code> is accepted with a validator WARNING. <code>DEFERRED</code>: resolution against a pinned CV release and the binding of <code>params</code> keys to the term's published parameters.</td>
      <td>Submitter MUST annotate.</td>
      <td><code>str</code> CURIE of the form <code>"QCMS:NNNNNNN"</code>, or <code>"PLACEHOLDER:&lt;name&gt;"</code>.</td>
    </tr>
    <tr>
      <td><code>name</code></td>
      <td>Human-readable label of the CV term, or of the operation when the accession is a placeholder.</td>
      <td>Submitter SHOULD annotate; MUST annotate when <code>accession</code> is a placeholder.</td>
      <td><code>str</code>.</td>
    </tr>
    <tr>
      <td><code>term_definition</code></td>
      <td>Definition of the CV term, as recorded in the CV.</td>
      <td>System MAY annotate.</td>
      <td><code>str</code>.</td>
    </tr>
    <tr>
      <td><code>input_layer</code></td>
      <td>Layer name read by this step. For the step that produces <code>layers['raw']</code> directly from engine output, MUST be <code>null</code>.</td>
      <td>Submitter SHOULD annotate.</td>
      <td><code>str</code> or <code>null</code>.</td>
    </tr>
    <tr>
      <td><code>output_layer</code></td>
      <td>Layer name written by this step. MUST be a canonical layer name from <a href="#layers"><code>layers</code></a>, a producer-chosen non-canonical layer name, or <code>null</code> for a step that produces no matrix (cross-modality inference that produces <code>var</code> indices).</td>
      <td>Submitter MUST annotate.</td>
      <td><code>str</code> or <code>null</code>.</td>
    </tr>
    <tr>
      <td><code>software</code></td>
      <td>Tool name that applied the step. The search engine when the engine performed the operation (engine-side FDR filtering, fraction aggregation into protein groups, engine normalisation).</td>
      <td>Submitter SHOULD annotate.</td>
      <td><code>str</code>.</td>
    </tr>
    <tr>
      <td><code>software_version</code></td>
      <td>Version of the tool that applied the step.</td>
      <td>Submitter SHOULD annotate.</td>
      <td><code>str</code>.</td>
    </tr>
    <tr>
      <td><code>date_applied</code></td>
      <td>ISO 8601 date the step was applied.</td>
      <td>Submitter SHOULD annotate.</td>
      <td><code>str</code> in <code>"YYYY-MM-DD"</code> format.</td>
    </tr>
    <tr>
      <td><code>params</code></td>
      <td>Method-specific parameter object. Keys and values are CV-term-dependent; values that name a CV class SHOULD be given as CURIEs.</td>
      <td>Submitter MAY annotate.</td>
      <td>JSON object.</td>
    </tr>
  </tbody>
</table>
<br>

The `transformations` array MUST begin with a step whose `output_layer` is `"raw"` and whose `input_layer` is `null`. This step records the engine-output read that anchored the artifact, including the identification filter the engine applied to produce the population in `layers['raw']` (the worked examples record it as the FDR filter). Subsequent steps record `log2`, `normalize`, `impute`, `batch_correct` and aggregation operations in order. `DEFERRED`: the second pass separates the read from the pre-`raw` operations (engine-side FDR, match-between-runs, engine normalisation, engine-side fraction aggregation) so that each can be its own step, by replacing this sentence with the rule that every layer present in `layers` is the `output_layer` of at least one step and `raw` appears; the one-step form above remains valid under that rule. Field-level provenance (which source column became which canonical column) is likewise `DEFERRED` and will be a MAY block in this container.

#### PTM-stoichiometry-adjustment provenance

When PTM site intensities have been normalized against parent-protein abundance — a transformation that converts raw site intensity into site occupancy fraction or log site/protein ratio — the producer MUST record a `TransformationStep` whose CV term identifies the operation as a PTM-stoichiometry adjustment and whose `params` object contains the following fields. This pattern is only meaningful when the artifact carries a `site` modality and a `protein` modality together, or when the parent-protein abundances come from a second, global artifact (in which case `params` SHOULD name that artifact). `DEFERRED`: the membership-match rule between a site's `protein_id` and the quantified protein groups, a per-site `adjustment_status` column, and an `obs` column for the SDRF enrichment process are part of the second pass.

<table>
  <thead>
    <tr>
      <th>Field</th>
      <th>Description</th>
      <th>Required</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>method</code></td>
      <td>Adjustment method.</td>
      <td>MUST</td>
      <td><code>str</code>. One of <code>"ratio"</code> (simple site / parent-protein ratio), <code>"ridge"</code> (ridge-regression-based site occupancy estimate per Wu et al.), <code>"custom"</code> (producer-defined; cite source in <code>custom_reference</code> sibling field).</td>
    </tr>
    <tr>
      <td><code>parent_protein_layer</code></td>
      <td>Name of the protein-modality layer used as the denominator. Determines whether the adjustment is computed on raw intensities, log-transformed values, or normalized values.</td>
      <td>MUST</td>
      <td><code>str</code>. One of the canonical layer names from <a href="#layers"><code>layers</code></a>, e.g. <code>"raw"</code>, <code>"log2"</code>, or <code>"normalized"</code>.</td>
    </tr>
    <tr>
      <td><code>rescale_factor</code></td>
      <td>Multiplicative factor applied to the site intensities after adjustment, e.g. to rescale ridge estimates back to the original-data scale. <code>1.0</code> indicates no rescaling.</td>
      <td>MUST</td>
      <td><code>float</code>.</td>
    </tr>
    <tr>
      <td><code>parent_protein_aggregation</code></td>
      <td>How the parent-protein intensity is obtained for sites whose <code>protein_group</code> contains multiple accessions (multi-accession protein groups). Determines reproducibility for shared peptides.</td>
      <td>SHOULD</td>
      <td><code>str</code>. One of <code>"first_accession"</code>, <code>"mean"</code>, <code>"median"</code>, <code>"max"</code>, <code>"sum"</code>.</td>
    </tr>
    <tr>
      <td><code>custom_reference</code></td>
      <td>Free-text citation (DOI, GitHub URL, or paper reference) for the adjustment method when <code>method == "custom"</code>. MUST NOT be present otherwise.</td>
      <td>MUST when <code>method == "custom"</code></td>
      <td><code>str</code>.</td>
    </tr>
  </tbody>
</table>
<br>

Reference implementations of the `ratio` and `ridge` methods exist in the proteomics tooling ecosystem (msmu's `adjust_ptm_by_protein` is the behavioural reference). The PSI Quantification CV will assign a single accession to the PTM-stoichiometry-adjustment operation; until that accession is registered, producers SHOULD use the placeholder `accession: "PLACEHOLDER:PTM_stoichiometry_adjustment"` and update on CV ratification. `PLACEHOLDER:` accessions trigger validator WARNING (not ERROR) and MUST be replaced with the registered QCMS CURIE upon CV ratification.

## SDRF Integration

This section is **informational**. It describes the canonical mapping from an SDRF-Proteomics file to the `obs` columns defined above. Validators MAY enforce this mapping; producers MAY populate `obs` by any mechanism (reader-side join, adapter-side join, engine-side emission) provided the resulting `obs` columns conform to the schema.

The SDRF-Proteomics specification is maintained at [https://github.com/bigbio/proteomics-sample-metadata](https://github.com/bigbio/proteomics-sample-metadata), with the template stack at [https://github.com/bigbio/sdrf-templates](https://github.com/bigbio/sdrf-templates) (`templates.yaml` declares base / sample / technology / experiment / organism / context layers, e.g. `base/1.1.0`, `sample-metadata/1.0.0`, `ms-proteomics/1.1.0`, `human/1.1.0`, `clinical-metadata/1.0.0`, `affinity-proteomics/1.0.0`). The canonical channel-label vocabulary is maintained in [sdrf-pipelines](https://github.com/bigbio/sdrf-pipelines) (`converters/channel_map.yaml`).

### Derivation principle

The `obs` MUST / SHOULD / MAY classifications in this specification are **derived from the SDRF template stack** rather than chosen ad-hoc:

- A column **required in the `base` or `sample-metadata` template** (inherited by every usable template) maps to `obs` **MUST**.
- A column **required in a technology-layer template** such as `ms-proteomics/1.1.0` or `affinity-proteomics/1.0.0` maps to `obs` **SHOULD**, since not every MIAPE-AnnData consumer uses that technology. Field descriptions explicitly cite the requiring template.
- A column **required only in an organism- or context-layer template** (e.g. `human/1.1.0`, `clinical-metadata/1.0.0`) maps to a **conditional MUST** clause on the `obs` field's Annotator row — the field is MUST when the relevant organism / context applies, SHOULD otherwise. Example: `disease_ontology_term_id` is MUST when `organism_ontology_term_id == "NCBITaxon:9606"`.
- A column **optional in all templates** maps to `obs` **MAY**.
- An `obs` field **not present in any SDRF template** is **MIAPE-introduced** (e.g. QC outputs like `passed_qc`, validator-derived fields, instrument-log-derived fields like `injection_order`). These fields are explicitly labelled as schema-introduced in their descriptions.

### Mapping rules

1. **Identity columns.** `source name` → `obs.sample_id`, unconditional. The identity of an SDRF row is the pair `(comment[data file], comment[label])` → `obs.raw_file`, `obs.label`; `obs.index` and `obs.assay_id` are producer-chosen, SHOULD `raw_file` for label-free and `raw_file|label` for multiplexed data. `assay name` identifies the data acquisition run and repeats across the channels of a multiplexed run; it is **not** a row identifier and falls through as a non-canonical column. When several SDRF rows are quantified together as one aggregated observation (fractions), `raw_file` is the `;`-joined list of their data files.
2. **`characteristics[known term]` → canonical `obs` column** per the `CHARACTERISTICS_MAP` listed below.
3. **`comment[known term]` → canonical `obs` column** per the `COMMENT_MAP` listed below.
4. **`factor value[*]` → `obs.condition`,** pipe-joined alphabetically by factor term name.
5. **Unknown columns** fall through to non-canonical `char_*` / `comment_*` prefixed columns and are not part of the canonical MIAPE-AnnData surface.

### `CHARACTERISTICS_MAP` (informational)

| SDRF column | `obs` column |
|---|---|
| `characteristics[individual]` | `donor_id` |
| `characteristics[sex]` | `sex` |
| `characteristics[age]` | `age` |
| `characteristics[organism]` | `organism` (via `organism_ontology_term_id`) |
| `characteristics[organism part]` | `tissue` (via `tissue_ontology_term_id`) |
| `characteristics[cell type]` | `tissue` (via `tissue_ontology_term_id`, when `tissue_type == "cell culture"`) |
| `characteristics[disease]` | `disease` (via `disease_ontology_term_id`) |
| `characteristics[biological replicate]` | `biological_replicate` |
| `characteristics[material type]` | `tissue_type` |
| `characteristics[pooled sample]` | `is_pool` |
| `characteristics[treatment]` | (factor-value pattern — joined into `condition` per SDRF Rule 4) |
| `characteristics[sampling time]` | (factor-value pattern — joined into `condition` per SDRF Rule 4) |
| `characteristics[enrichment process]` | (no canonical obs mapping in this draft; falls through to `char_*` prefix; `DEFERRED` with the PTM-stoichiometry item) |
| `characteristics[biosample accession number]` | (no canonical obs mapping; falls through to `char_*` prefix) |

### `COMMENT_MAP` (informational)

| SDRF column | `obs` column |
|---|---|
| `comment[data file]` | `raw_file` |
| `comment[sample preparation batch]` | `prep_batch` (and may contribute to `batch`) |
| `comment[lc batch]` | `lc_batch` (and may contribute to `batch`) |
| `comment[technical replicate]` | `technical_replicate` |
| `comment[label]` | `label` (normalised to the canonical channel label of Appendix E; `label free sample` → `LFQ`) |
| `comment[fraction identifier]` | `fraction` |
| `comment[instrument]` | `instrument` |
| `comment[cleavage agent details]` | `enzyme` |
| `comment[proteomics data acquisition method]` | `acquisition_method` |
| `comment[sample type]` | `sample_type` |
| `comment[acquisition date]` | (no canonical obs mapping; falls through to `comment_*` prefix) |
| `comment[dissociation method]` | (no canonical obs mapping; falls through to `comment_*` prefix) |
| `comment[modification parameters]` | (no canonical obs mapping; encoded modifications appear in `var.modifications` for the peptidoform modality) |

### CURIE columns

Some downstream SDRF dialects carry CURIE values in `factor value[*_ontology_term_id]` columns alongside the human-readable `factor value[X]` column. When present, validators read CURIEs directly from those columns and populate the paired human-readable label column by ontology lookup. The base SDRF-Proteomics 1.1.0 spec does not declare `factor value[*_ontology_term_id]` columns; this convention is supported for interop with dialects that do.

### Multi-factor condition composition

When the SDRF declares multiple `factor value[*]` columns, the canonical `condition` value MUST be the pipe-joined factor values, sorted alphabetically by factor term name before joining. This prevents non-matching condition strings across tools that load the same SDRF in different column orders.

Example: an SDRF declaring `factor value[treatment]=Drug` and `factor value[time]=T0` yields `obs.condition = "T0|Drug"` (because `time` sorts before `treatment` alphabetically).


## Appendix A. Changelog

> **How to read this changelog.** Each entry records what that version introduced *relative to its predecessor* at the time it was written. Because every version of this schema has remained `Document Status: _Draft_`, patches have been applied retroactively to each version's file rather than as patch-version bumps (the 2026-05-27 review-response patches; the v0.5.0 second pass will patch this file in place). Entries below therefore describe the version's original scope; where a later patch changed that content — for example the ProForma 2.1 → 2.0 rebinding recorded under v0.3.0, since reversed by v0.5.0 — the normative body of this document, not this changelog, is authoritative.

### schema v0.5.0

First cut, 2026-11-17. Written against six producers (alphapepttools, msmu, apb2, proteopy, QPX, protdata); design record RFC-002 and ADR-005 … ADR-016 in the maintaining repository. Versions 0.1.0–0.4.0 were internal; the renames below would have been major-version changes had they been released.

* Added: **peptidoform ion modality** (`peptidoform_ion`; precursor level, PSI-MS `peptidoform ion`). MAY. Index is Level 1-ProForma 2.1 plus `/charge`; `charge` MUST; at least one upward foreign key MUST. No PSM modality.
* Changed: **feature indices are grammar-free.** Every `var` index MUST be unique and equal to its key column; the string grammar is no longer fixed. Identity moves to normalized columns: protein `protein_ids` (MUST) and `leading_protein` (SHOULD); site `protein_id` (MUST), `residue_positions` (MUST, list column replacing `residue_position`) and `modification`.
* Changed: **peptide-level protein columns.** `protein_group` is MUST only when a protein modality is present (nullable), otherwise MAY; `protein_ids` SHOULD (what the peptide maps to); `is_unique` SHOULD with a checkable definition (every `protein_ids` token is a member of the assigned group). Every non-protein modality MUST link upward to a coarser modality present.
* Changed: **peptidoform index is Level 1-ProForma 2.1** in any modification form (MUST); full accession form SHOULD; the `U:N` / `M:N` shorthand is removed; the Level 2 unknown-position form MAY appear.
* Changed: **composite site rows** for homogeneous co-modification, never for localization ambiguity; `n_sites`, aligned `residue`, `protein_positions` added; `localization_prob` becomes SHOULD. Terminal modifications stay at the peptidoform level in this draft.
* Changed: **`obs` rows are quantification channels.** The index is grammar-free (MUST unique, equal to `assay_id`); identity is `(raw_file, label)`, unique per row and at the member-file level; `raw_file` becomes MUST and lists the member files `;`-joined for an aggregated row, with the aggregation recorded as a `TransformationStep`; SDRF `assay name` no longer feeds the index. `fraction` aligns with `raw_file`.
* Changed: **`label` vocabulary.** Bare canonical channel labels of the sdrf-pipelines channel map (Appendix E): family pattern MUST, list membership SHOULD, cross-checked against `labeling_type`; the label-free value is `LFQ` (was `"label free sample"`).
* Changed: **`replicate` renamed `technical_replicate`** and tiered MUST per the Derivation Principle (base-required SDRF column); the sentence requiring biological replication to be encoded by `donor_id` is removed; `biological_replicate` unchanged.
* Changed: `enzyme` names `unspecific cleavage` (MS:1001956) and `no cleavage` (MS:1001955) as the values for preparations without a specific enzyme.
* Changed: **`uns['quantification_provenance']` is one JSON string** under the fixed key (a list of dicts does not write to HDF5); additional keys on a step are permitted and ignored. `output_layer` MAY be null for steps that produce no matrix.
* Added: **no decoy features** in `var` or any matrix; decoy evidence MAY live in `uns` under a producer key.
* Added: **`var['q_value']`** SHOULD on every modality: the experiment-wide q-value at the feature's level, typed per level by PSI-MS terms.
* Changed: `varp['feature_mapping']` generalised from two levels to N, coarse-to-fine concatenation order, transitive closure permitted; a declared key join alone is not a storage convention.
* Changed: Delimiter rule stated once: `;` for list members, `|` for composite-key fields.
* Changed: SDRF mapping rule 1 rewritten around `(comment[data file], comment[label])`; `COMMENT_MAP` updated for `technical_replicate` and label normalisation; `characteristics[enrichment process]` listed as a fall-through.
* Deferred (second pass; see Status of this draft): step order and DAG edges, the step-1 anchor and pre-`raw` operations, field-level provenance, `params`–CV binding, typed score layers, the `missingness_mask` clarification, the PTM-stoichiometry adjustment details and terminal positions, result slots, gene-level matrices, a multiplexed fractionated worked example.

### schema v0.4.0

* **Patched 2026-05-27 (obs reconciliation against SDRF templates):** Same obs patches as v0.1.0 (fraction/enzyme/instrument MAY → SHOULD; disease conditional MUST when human; is_blank → sample_type; condition MUST → SHOULD; SDRF Derivation principle; `COMMENT_MAP` extended). Applied to the same `obs` fields across versions 0.1.0 through 0.4.0. Note that a subsequent 2026-05-27 patch made `obs` assay-level (`assay_id` as the MUST index, `sample_id` demoted to SHOULD), so v0.4.0's `obs` section is no longer identical to v0.1.0's earlier text.
* Added: site modality. `### Site modality` subsection under `## var (Feature Metadata)` specifies `site_id` (index, CURIE-typed pipe-delimited `{protein_accession_CURIE}|{residue_position}|{modification_CURIE}` format), `protein_group` (foreign key to protein modality), `peptidoform` (foreign key to peptidoform modality, MAY), `residue_position`, `residue`, `modification` (UNIMOD/PSI-MOD CURIE), `localization_prob`, `is_class_I` (system-derived), `n_localizing_peptidoforms`.
* Added: `localization_prob` propagation requirement — producers MUST propagate engine-reported PTM site localization probabilities. Tool-specific migration guidance, including implementations that currently drop the value, belongs in the implementation-notes companion document (TBD) rather than in this specification.
* Added: `#### PTM-stoichiometry-adjustment provenance` sub-subsection under the `TransformationStep` schema. When site intensities have been normalized against parent-protein abundance, the producer MUST record a `TransformationStep` whose `params` dict contains `method` (`ratio` / `ridge` / `custom`), `parent_protein_layer`, `rescale_factor`, and `parent_protein_aggregation`. Named reference implementations are listed in the implementation-notes companion document (TBD) rather than in this specification.
* Changed: `## MuData Composition` modality-key enumerated set expanded to include `site`. Note added on the four-modality chain of `varp['feature_mapping']` adjacencies (site ↔ peptidoform ↔ peptide ↔ protein).
* Changed: `uns.level` enum extended with `"site"`.
* Changed: Background and Overview sections updated to reflect site addition and PTM-stoichiometry-adjustment formalization.

### schema v0.3.0

* Added: peptidoform modality. `### Peptidoform modality` subsection under `## var (Feature Metadata)` specifies `peptidoform` (index, ProForma 2.1 string with UNIMOD/PSI-MOD CURIEs), `sequence` (foreign key to peptide modality), `protein_group` (foreign key to protein modality), `modifications` (structured list of (position, CURIE) tuples), `is_unique`, `n_psms`, `detection_rate`.
* Added: ProForma 2.1 specified as the canonical peptidoform notation. Modifications MUST be CURIE-form (`UNIMOD:N` / `MOD:N` or shorthand `U:N` / `M:N`); mass-delta-only ProForma forms MAY appear as secondary engine-native columns but MUST NOT be used in the canonical index.
* Changed: `## MuData Composition` modality-key enumerated set expanded to include `peptidoform`. Note added on the multi-hop peptidoform → peptide → protein adjacency that becomes possible with all three modalities present.
* Changed: `uns.level` enum extended with `"peptidoform"`.
* Changed: Background and Overview sections updated to reflect peptidoform addition and ProForma 2.1 reference.

### schema v0.2.0

* Added: bare-sequence peptide modality. `### Peptide modality` subsection under `## var (Feature Metadata)` specifies `sequence` (index), `protein_group` (foreign key to protein modality), `is_unique`, `missed_cleavages`, `length`, `n_psms`, `detection_rate`.
* Added: `## MuData Composition` section formalizing the `.h5mu` artifact shape for multi-modality datasets (protein + peptide). Modality keys MUST be drawn from `{"protein", "peptide"}`; per-modality `obs` MUST share the same `sample_id` index. (Historical: the shared index became `assay_id` with the 2026-05-27 assay-level `obs` change — see the normative `### Sample alignment` section, which is authoritative.)
* Added: `## varp (Cross-modality feature linkage)` section specifying `varp['feature_mapping']` as a sparse boolean adjacency matrix between modalities, with three storage conventions (mulink, varp, uns_dict) in decreasing order of preference.
* Added: `uns.linkage_convention` field — MUST be set on `.h5mu` artifacts; values `"mulink"` / `"varp"` / `"uns_dict"`.
* Changed: `uns.level` enum expanded — values are now `"protein"`, `"peptide"`, `"multi"`. The MuData-level `uns.level` MUST be `"multi"`; per-modality `uns.level` MUST be a modality-specific value.
* Changed: `## var (Protein Feature Metadata)` heading renamed to `## var (Feature Metadata)` with new intro describing single-modality vs. multi-modality var DataFrames. Protein modality content moved under `### Protein modality` divider; per-field subsections (`### protein_group`, `### gene_name`, etc.) unchanged.
* Changed: Background and Overview sections updated to reflect the multi-modality scope.

### schema v0.1.0

* Initial release.
* Specifies the protein-modality MIAPE-AnnData artifact: `X` shape, `layers` (raw, missingness_mask, log2, normalized, imputed, batch_corrected_observed, batch_corrected_all), `obs` (25 sample-level fields), `var` (9 protein-group fields), `uns` (10 dataset-level fields including `quantification_provenance` with nested `TransformationStep` schema).
* Establishes the CURIE-pair convention for ontology-bound metadata (`*_ontology_term_id` submitter-annotated; `*` human-readable system-annotated).
* Establishes the dense-float-with-NaN missing-value convention; sparse encodings non-compliant; zero-imputed missing values non-compliant.
* Establishes the `transformations` provenance chain anchored on `layers['raw']` and bound to Quantification CV terms.
* Informational SDRF-Proteomics → `obs` mapping documented but not enforced as normative on the wire format.


## Appendix B. Worked example — minimal compliant protein artifact

The following describes a minimal v0.5.0-compliant single-modality protein `.h5ad` artifact for a label-free FragPipe protein-group dataset, one acquisition per sample.

```
adata.obs:
  index = assay_id            # here equal to raw_file (label-free)
  required columns:
    assay_id, raw_file, label (= "LFQ"), biological_replicate, technical_replicate,
    organism_ontology_term_id, organism (system),
    assay_ontology_term_id, assay (system),
    tissue_type
  recommended columns:
    sample_id, condition,
    tissue_ontology_term_id, tissue (system),
    disease_ontology_term_id, disease (system),
    development_stage_ontology_term_id, development_stage (system),
    donor_id, sex, age, batch, fraction,
    is_pool, sample_type, passed_qc

adata.var:
  index = protein_group       # grammar-free; here the engine's group string
  required columns:
    protein_group, protein_ids
  recommended columns:
    leading_protein, gene_name, organism, is_contaminant,
    n_peptides_used, detection_rate, q_value
  Example var row values:
    protein_group:    "P12345;P67890"
    protein_ids:      "P12345;P67890"
    leading_protein:  "P12345"
    q_value:          0.0021            # Global.PG.Q.Value-style, experiment-wide

adata.X:
  dense numpy.ndarray, dtype = float32, shape = (n_observations, n_protein_groups)
  missing values = NaN

adata.layers:
  raw                 (required, identical content to X)
  log2                (recommended)
  normalized          (recommended; method recorded in transformations)
  missingness_mask    (recommended; required if imputed is present)
  imputed             (optional)

adata.uns:
  miape_anndata_spec      = "0.5.0"
  search_engine           = "fragpipe"
  search_engine_version   = "22.0"
  loader                  = "alphapepttools"
  loader_version          = "0.4.1"
  sdrf_file               = "experiment_design.sdrf.tsv"
  loaded_at               = "2026-11-17"
  labeling_type           = "LFQ"
  level                   = "protein"
  search_database         = "UP000005640_9606_2024_03.fasta"
  quantification_provenance = <one JSON string>; decoded:
    {"transformations": [
      {"step": 1,
       "accession": "QCMS:0000005",          # placeholder, pending CV registration
       "name": "protein-level FDR filter",
       "input_layer": null,
       "output_layer": "raw",
       "software": "fragpipe",
       "software_version": "22.0",
       "date_applied": "2026-11-10",
       "params": {"threshold": 0.01, "criterion": "q_value",
                  "decoy_strategy": "target_decoy"}},
      {"step": 2,
       "accession": "QCMS:0000010",          # placeholder, pending CV registration
       "name": "log2 transform",
       "input_layer": "raw",
       "output_layer": "log2",
       "software": "alphapepttools",
       "software_version": "0.4.1",
       "date_applied": "2026-11-10",
       "params": {"offset": 1.0}},
      {"step": 3,
       "accession": "QCMS:0000020",          # placeholder, pending CV registration
       "name": "median normalization",
       "input_layer": "log2",
       "output_layer": "normalized",
       "software": "alphapepttools",
       "software_version": "0.4.1",
       "date_applied": "2026-11-10",
       "params": {"target": "sample_median"}}
    ]}
```

Note: the `QCMS:NNNNNNN` accessions above are illustrative placeholders. The PSI Quantification CV is under active development; producers MUST consult the registered CV (see [Required Ontologies](#required-ontologies)) for the canonical accession of each transformation method, and MAY use `PLACEHOLDER:<name>` accessions with a `name` until then.

### Note on MIAPE-MSI compliance via `transformations`

Throughout this specification **MIAPE-MSI** denotes the HUPO-PSI **Mass Spectrometry Informatics** module — Binz et al., *Nature Biotechnology* **26**, 862 (2008), "Guidelines for reporting the use of mass spectrometry informatics in proteomics" — which covers search engine, version, parameters, and identification thresholds. It does **not** denote mass-spectrometry *imaging*. The module list at [psidev.info/miape](https://www.psidev.info/miape) names MIAPE-MS, MIAPE-MSI, MIAPE-Quant, MIAPE-GE, MIAPE-GI, MIAPE-CC, and MIAPE-CE, none of which covers imaging; any future imaging profile MUST therefore be given a distinct name to avoid collision.

Step 1 in the example above shows how MIAPE-MSI's identification-threshold reporting requirement is satisfied within MIAPE-AnnData. The protein-level FDR cutoff is recorded as a `TransformationStep` whose CV term identifies the operation (an FDR filter), whose `params` carry the threshold and decoy strategy, and whose `output_layer` is `"raw"` — meaning the protein groups present in `layers['raw']` are exactly those that passed the recorded FDR filter, and `var['q_value']` carries the per-group evidence that threshold was applied to. There is no separate `uns['fdr_threshold_*']` field; the `transformations` chain is the single source of truth for which operations produced the artifact's protein-group population.

The same pattern handles peptide-, precursor- and site-localization-threshold reporting in the other modalities.


## Appendix C. Worked example — protein + peptide `.h5mu` artifact

The following describes a minimal v0.5.0-compliant `.h5mu` artifact for a label-free FragPipe dataset carrying both protein-group and bare-sequence peptide quantifications.

```
mdata.mod['protein']:
  obs.index = assay_id (= raw_file)
  var.index = protein_group
  var columns: protein_group, protein_ids, leading_protein, gene_name,
               is_contaminant, n_peptides_used, detection_rate, q_value
  X: dense float32, shape (n_observations, n_protein_groups), NaN for missing
  layers: raw, log2, missingness_mask (per §`layers`)
  uns:
    miape_anndata_spec = "0.5.0"
    level              = "protein"
    search_engine      = "fragpipe"
    search_engine_version = "22.0"
    search_database    = "UP000005640_9606_2024_03.fasta"
    labeling_type      = "LFQ"
    loader             = "alphapepttools"
    loader_version     = "0.4.1"
    quantification_provenance = <JSON string>; decoded:
      {"transformations": [
        {"step": 1, "accession": "QCMS:0000005", "name": "protein-level FDR filter",
         "input_layer": null, "output_layer": "raw",
         "software": "fragpipe", "software_version": "22.0",
         "params": {"threshold": 0.01, "criterion": "q_value",
                    "decoy_strategy": "target_decoy"}}
      ]}

mdata.mod['peptide']:
  obs.index = assay_id      (element-wise equal to the protein modality's)
  var.index = sequence       (bare amino acid sequences)
  var columns:
    sequence, protein_group, protein_ids, is_unique, missed_cleavages,
    length, n_psms, detection_rate, q_value
  Example var row values:
    sequence:       "EMEVEESPEK"
    protein_group:  "P12345;P67890"     # the group the engine assigned (FK to protein var)
    protein_ids:    "P12345"            # what the sequence maps to
    is_unique:      true                # "P12345" is a member of the assigned group
  X: dense float32, shape (n_observations, n_peptides), NaN for missing
  layers: raw, log2, missingness_mask
  uns:
    miape_anndata_spec = "0.5.0"
    level              = "peptide"
    search_engine      = "fragpipe"
    search_engine_version = "22.0"
    labeling_type      = "LFQ"
    quantification_provenance = <JSON string>; decoded:
      {"transformations": [
        {"step": 1, "accession": "QCMS:0000006", "name": "peptide-level FDR filter",
         "input_layer": null, "output_layer": "raw",
         "software": "fragpipe", "software_version": "22.0",
         "params": {"threshold": 0.01, "criterion": "q_value"}}
      ]}

mdata.obs:
  index = assay_id (shared across both modalities — MuData inner-join)
  required columns:
    assay_id, raw_file, label, biological_replicate, technical_replicate,
    organism_ontology_term_id, organism (system),
    assay_ontology_term_id, assay (system), tissue_type

mdata.uns:
  miape_anndata_spec   = "0.5.0"
  level                = "multi"
  linkage_convention   = "mulink"
  loaded_at            = "2026-11-17"
  quantification_provenance = <JSON string>; decoded:
    {"transformations": [
      {"step": 1, "accession": "QCMS:0000050", "name": "parsimony protein inference",
       "input_layer": null, "output_layer": null,   # produces both modalities' var indices
       "software": "fragpipe", "software_version": "22.0",
       "params": {"method": "parsimony", "shared_peptide_handling": "razor"}}
    ]}

mdata.varp['feature_mapping']:
  scipy.sparse.csr_matrix of bool
  shape (n_protein_groups + n_peptides,
         n_protein_groups + n_peptides)
  True at position (i, j) iff feature i and feature j are linked
  by the parsimony protein-inference step recorded in the MuData-level
  quantification_provenance.
  Constructed via mdata.link.add_link(...) (linkage_convention = "mulink").
```

The cross-modality `parsimony protein inference` step is recorded in the MuData-level `quantification_provenance` because it produced **both** the protein modality's var index AND the peptide modality's `protein_group` foreign-key column from a shared PSM source. Per-modality `quantification_provenance` records modality-specific operations (e.g. the per-modality FDR filters in this example). This separation makes cross-modality vs. modality-local provenance distinguishable when querying the artifact.


## Appendix D. Worked example — four-modality `.h5mu` artifact with site + PTM-stoichiometry

The following describes a minimal v0.5.0-compliant `.h5mu` artifact for a label-free phosphoproteomics dataset carrying protein, peptide, peptidoform and site modalities with PTM site localization and PTM-stoichiometry-adjusted site intensities. A `peptidoform_ion` modality could be added beside `peptidoform` with the same `obs` rows; it is omitted here.

```
mdata.mod['protein']:
  obs.index = assay_id (= raw_file)
  var.index = protein_group
  var columns: protein_group, protein_ids, leading_protein, gene_name, organism,
               is_contaminant, contaminant_source, n_peptides_used, detection_rate, q_value
  X: dense float32, shape (n_observations, n_protein_groups), NaN for missing
  layers: raw, log2, missingness_mask
  uns:
    miape_anndata_spec = "0.5.0"
    level              = "protein"
    search_engine      = "fragpipe"
    search_engine_version = "22.0"
    search_database    = "UP000005640_9606_2024_03.fasta"
    labeling_type      = "LFQ"
    loader             = "alphapepttools"
    loader_version     = "0.4.1"
    quantification_provenance = <JSON string>; decoded:
      {"transformations": [
        {"step": 1, "accession": "QCMS:0000005", "name": "protein-level FDR filter",
         "input_layer": null, "output_layer": "raw",
         "software": "fragpipe", "software_version": "22.0",
         "params": {"threshold": 0.01, "criterion": "q_value",
                    "decoy_strategy": "target_decoy"}}
      ]}

mdata.mod['peptide']:
  obs.index = assay_id
  var.index = sequence
  var columns: sequence, protein_group, protein_ids, is_unique, missed_cleavages,
               length, n_psms, detection_rate, q_value
  X: dense float32, NaN for missing
  layers: raw, log2, missingness_mask

mdata.mod['peptidoform']:
  obs.index = assay_id
  var.index = peptidoform   # Level 1-ProForma 2.1, full accession form
  var columns: peptidoform, sequence, protein_group, protein_ids, modifications,
               is_unique, n_psms, detection_rate, q_value
  Example var.index values:
    "EM[UNIMOD:35]EVEESPEK"             # methionine oxidation
    "EM[UNIMOD:35]EVEES[UNIMOD:21]PEK"  # + phospho-S7
    "EMEVEES[UNIMOD:21]PEK"             # phospho-S7 only
    "EMEVEES[UNIMOD:21]PEKS[UNIMOD:21]R" # two phospho sites on one peptidoform
  X: dense float32, NaN for missing
  layers: raw, log2, missingness_mask

mdata.mod['site']:
  obs.index = assay_id
  var.index = site_id      # grammar-free; recommended single-site form shown
  var columns: site_id, protein_id, protein_group, peptidoform, residue_positions,
               n_sites, residue, modification, localization_prob, is_class_I,
               n_localizing_peptidoforms, q_value
  Example var row values (single site):
    site_id:                   "UniProtKB:P11802|794|UNIMOD:21"
    protein_id:                "P11802"
    protein_group:             "P11802"
    peptidoform:               "EMEVEES[UNIMOD:21]PEK"
    residue_positions:         "794"
    n_sites:                   1
    residue:                   "S"
    modification:              "UNIMOD:21"
    localization_prob:         0.98
    is_class_I:                true
    n_localizing_peptidoforms: 2
    q_value:                   0.004
  Example var row values (composite, co-modified):
    site_id:                   "P11802|794_797|UNIMOD:21"   # producer's composite key
    protein_id:                "P11802"
    protein_group:             "P11802"
    peptidoform:               "EMEVEES[UNIMOD:21]PEKS[UNIMOD:21]R"
    residue_positions:         "794;797"
    n_sites:                   2
    residue:                   "S;S"
    modification:              "UNIMOD:21"
    localization_prob:         0.91      # the smaller of the two per-site probabilities
    is_class_I:                true
  X: dense float32, NaN for missing
  layers:
    raw                   # raw site intensities (engine output)
    log_ratio             # log2(site / parent-protein) after PTM-stoichiometry adjustment
    missingness_mask
  uns:
    miape_anndata_spec = "0.5.0"
    level              = "site"
    quantification_provenance = <JSON string>; decoded:
      {"transformations": [
        {"step": 1, "accession": "QCMS:0000005", "name": "site-localization FDR filter",
         "input_layer": null, "output_layer": "raw",
         "software": "fragpipe", "software_version": "22.0",
         "params": {"threshold": 0.01, "criterion": "PTM-Prophet localization"}},
        {"step": 2, "accession": "PLACEHOLDER:PTM_stoichiometry_adjustment",
         "name": "PTM-stoichiometry adjustment (log site / parent protein)",
         "input_layer": "raw", "output_layer": "log_ratio",
         "software": "msmu", "software_version": "0.4.5",
         "params": {"method": "ridge", "parent_protein_layer": "log2",
                    "rescale_factor": 1.0,
                    "parent_protein_aggregation": "first_accession"}}
      ]}

mdata.obs:
  index = assay_id (shared across all four modalities, MuData inner-join)
  required columns:
    assay_id, raw_file, label, biological_replicate, technical_replicate,
    organism_ontology_term_id, organism (system),
    assay_ontology_term_id, assay (system), tissue_type

mdata.uns:
  miape_anndata_spec   = "0.5.0"
  level                = "multi"
  linkage_convention   = "mulink"
  loaded_at            = "2026-11-17"
  quantification_provenance = <JSON string>; decoded:
    {"transformations": [
      {"step": 1, "accession": "QCMS:0000050", "name": "parsimony protein inference",
       "input_layer": null, "output_layer": null,   # cross-modality; produces var indices
       "software": "fragpipe", "software_version": "22.0",
       "params": {"method": "parsimony", "shared_peptide_handling": "razor"}}
    ]}

mdata.varp['feature_mapping']:
  scipy.sparse.csr_matrix of bool
  shape (n_protein + n_peptide + n_peptidoform + n_site,
         n_protein + n_peptide + n_peptidoform + n_site)
  Records the chain of cross-modality adjacencies:
    - peptide ↔ protein           (engine peptide-to-protein-group assignment)
    - peptidoform ↔ peptide       (modification-state-to-bare-sequence collapse)
    - peptidoform ↔ protein       (transitive; materialised for queryability)
    - site ↔ peptidoform          (localized site to supporting peptidoforms)
    - site ↔ protein              (transitive)
  Constructed via mdata.link.add_link(...) (linkage_convention = "mulink").
  Tools that consume via mdata.link can navigate multi-hop queries
  (e.g. site → peptidoform → protein) using transitive-closure operations.
```

Key features illustrated:
- **Site modality** with a grammar-free `site_id`, identity in `protein_id` + `residue_positions` + `modification`, and a **composite row** for a doubly phosphorylated peptidoform whose two sites are quantified together rather than double-counted.
- **`localization_prob` propagation** — engine-reported PTM-Prophet probability preserved per site; the composite carries the smaller of its per-site probabilities.
- **`is_class_I` system-derived flag** computed from the 0.75 threshold convention.
- **PTM-stoichiometry-adjustment provenance** — the `log_ratio` layer is produced by step 2 in the site modality's `transformations`, bound to a `PLACEHOLDER:` accession with `method` / `parent_protein_layer` / `rescale_factor` / `parent_protein_aggregation` params per §`uns` → PTM-stoichiometry-adjustment provenance. The placeholder will be replaced with the registered QCMS CURIE upon CV ratification; validators emit WARNING (not ERROR) for `PLACEHOLDER:` accessions.
- **Provenance as one JSON string** per modality and at the MuData level, decoded for display above.
- **Four-way cross-modality linkage** in `varp['feature_mapping']`, enabling multi-hop transitive-closure queries via the mulink accessor.


## Appendix E. Label vocabulary

The values of `obs.label` are the canonical channel labels of the sdrf-pipelines channel map (`src/sdrf_pipelines/converters/channel_map.yaml`, schema version 2, as of sdrf-pipelines commit `8250b0f`, 2026-11-07), which is the vocabulary the SDRF converters normalise `comment[label]` to and the vocabulary QPX 1.3 made normative for its own `label` field. The family patterns are MUST; membership of the list is SHOULD, so that a plex released after this cut yields a WARNING rather than an ERROR. The spec reproduces the list so the artifact does not depend on a network lookup; when the upstream map changes, this appendix is refreshed by in-place patch.

| Family (`uns.labeling_type`) | Pattern (MUST) | Canonical labels (SHOULD) | Plexes |
|---|---|---|---|
| `LFQ` | `^LFQ$` | `LFQ` | label-free |
| `TMT` | `^TMT1(2[6-9]\|3[0-5])(N\|C\|D\|ND\|CD)?$` | `TMT126`, `TMT127`, `TMT127N`, `TMT127C`, `TMT127D`, `TMT128`, `TMT128N`, `TMT128C`, `TMT128ND`, `TMT128CD`, `TMT129`, `TMT129N`, `TMT129C`, `TMT129ND`, `TMT129CD`, `TMT130`, `TMT130N`, `TMT130C`, `TMT130ND`, `TMT130CD`, `TMT131`, `TMT131N`, `TMT131C`, `TMT131ND`, `TMT131CD`, `TMT132N`, `TMT132C`, `TMT132ND`, `TMT132CD`, `TMT133N`, `TMT133C`, `TMT133ND`, `TMT133CD`, `TMT134N`, `TMT134C`, `TMT134ND`, `TMT134CD`, `TMT135N`, `TMT135ND`, `TMT135CD` | 6, 10, 11, 16, 18, 32, 35 |
| `iTRAQ` | `^ITRAQ1(1[3-9]\|21)$` | `ITRAQ113`, `ITRAQ114`, `ITRAQ115`, `ITRAQ116`, `ITRAQ117`, `ITRAQ118`, `ITRAQ119`, `ITRAQ121` | 4, 8 |
| `SILAC` | `^SILAC (light\|medium\|heavy)$` | `SILAC light`, `SILAC medium`, `SILAC heavy` | 2, 3 |
| `mTRAQ` | `^MTRAQ[048]$` | `MTRAQ0`, `MTRAQ4`, `MTRAQ8` | 3 |
| `dimethyl` | `^DIMETHYL[02468]$` | `DIMETHYL0`, `DIMETHYL2`, `DIMETHYL4`, `DIMETHYL6`, `DIMETHYL8` | 2, 3, 5 |
| `other` | none | the SDRF `comment[label]` value as written (NeuCode, 15N and other schemes have no canonical name yet) | — |

Labels are compared exactly, case included. Producers that read SDRF normalise the common spellings to these labels on ingest, as the sdrf-pipelines converters do: the ontology name `label free sample` and its key-value forms (`NT=label free sample;AC=MS:1002038`) become `LFQ`; plex-qualified or ontology-form channel spellings (`TMT10plex_126`, `TMTpro-134C`, `iTRAQ4plex-114`, `NT=TMT126;AC=PRIDE:0000114`) become the bare canonical label; a bare reporter mass (`126`, `114`) resolves by range (iTRAQ reporters stop at 121, TMT starts at 126). PSI-MS cannot serve as this vocabulary today: its `TMT reagent` terms stop at 131 and it has no TMTpro, mTRAQ or dimethyl reagent terms; a term request is a working-group item.

