# psi_ai_quant_resources

Working repository of the HUPO-PSI AI Readiness Working Group, PSI-AI Quant Subgroup, for a
PSI Recommendation on representing quantitative proteomics results in AnnData and MuData
containers.

**Status:** working draft. Nothing here has been ratified through the HUPO-PSI Document
Process; content, schemas and vocabulary terms may change. Feedback is welcome as issues.

## What is here

| Path | What it is |
|---|---|
| [`ROADMAP.md`](./ROADMAP.md) | The direction: what is being built, why, the three work tracks, milestones, roles, and the decisions that are made and still open. Start here. |
| [`schemas/miape-anndata/0.5.0/schema.md`](./schemas/miape-anndata/0.5.0/schema.md) | The MIAPE-AnnData schema, v0.5.0 Draft. The normative field-level core of the specification: matrix and layers, `obs`, `var` per modality, MuData composition, `varp` linkage, `uns` provenance, SDRF integration. |
| `docs/` | The specification site: introduction, data model, schema reference, conformance, profiles. Not yet started. |
| `schema/`, `examples/` | The LinkML structural schema with generated JSON Schema, and the example artifacts. Not yet started. |

## How the work is organised

Three parent issues, one per track (Document, Artifacts, Process), each with a checklist of
child issues. Take a child issue, not a track. The roadmap states direction and changes only
when the direction changes; the issues are the assignments.

## Contributing

- Review comments and questions: open an issue in this repository.
- Term requests for the Quantification controlled vocabulary: the
  [PSI-Quant-CV](https://github.com/HUPO-PSI/PSI-Quant-CV) repository.
- Changes to the schema text are editorial patches in place while the document is a Draft;
  anything that changes a requirement is raised as an issue first.

## Related specifications

[mzPeak](https://www.mzpeak.org/spec/) (the layout this specification follows),
[SDRF-Proteomics](https://github.com/bigbio/proteomics-sample-metadata),
[ProForma 2.1](https://github.com/HUPO-PSI/ProForma),
[PSI-MS controlled vocabulary](https://github.com/HUPO-PSI/psi-ms-CV),
[mzQC](https://github.com/HUPO-PSI/mzQC),
[AnnData](https://anndata.readthedocs.io/) and [MuData](https://mudata.readthedocs.io/).

## License

See [`LICENSE`](./LICENSE).
