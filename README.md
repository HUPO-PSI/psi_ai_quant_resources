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
| [`.github/`](./.github) | Issue templates (bug report, feature request, documentation issue, work plan) and the pull request template. |

## How the work is organised

Three parent issues, one per track, each with a checklist of child issues:
[#4 Document](https://github.com/lmboat/psi_ai_quant_resources/issues/4),
[#5 Artifacts](https://github.com/lmboat/psi_ai_quant_resources/issues/5),
[#6 Process](https://github.com/lmboat/psi_ai_quant_resources/issues/6).
Take a child issue, not a track. Every issue carries a `plan`, `documentation` or
`enhancement` label and sits in one of three milestones, Draft 1 freeze, EuBIC-MS hackathon,
or Document Process submission; the
[Milestones page](https://github.com/lmboat/psi_ai_quant_resources/milestones) is the
progress view. The roadmap states direction and changes only when the direction changes; the
issues are the assignments.

## Contributing

- Review comments and questions: open an issue in this repository. The "New issue" chooser
  offers the four templates.
- Term requests for the Quantification controlled vocabulary: the
  [PSI-Quant-CV](https://github.com/HUPO-PSI/PSI-Quant-CV) repository.
- Changes to the schema text are editorial patches in place while the document is a Draft;
  anything that changes a requirement is raised as an issue first.
- Pull requests start from the template and link the issue they close.

## Related specifications

[mzPeak](https://www.mzpeak.org/spec/) (the layout this specification follows),
[SDRF-Proteomics](https://github.com/bigbio/proteomics-sample-metadata),
[ProForma 2.1](https://github.com/HUPO-PSI/ProForma),
[PSI-MS controlled vocabulary](https://github.com/HUPO-PSI/psi-ms-CV),
[mzQC](https://github.com/HUPO-PSI/mzQC),
[AnnData](https://anndata.readthedocs.io/) and [MuData](https://mudata.readthedocs.io/).

## License

See [`LICENSE`](./LICENSE).
