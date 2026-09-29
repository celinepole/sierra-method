---
ontology: https://fireforce6.github.io/mission-control/operational-analysis/entities
---

# Traceability, at operational scope

This page invokes the same method-owned template as the main analysis notebook, against the
entities description and its import closure instead of the whole Mission Control bundle.

The template is unchanged. Only the context differs, and the results are identical - which
is the useful observation. The traceability chain lives entirely inside this closure:
`entities` extends `capabilities`, which extends `objectives`, which extends `concerns` and
`missions`. Nothing the analysis needs comes from the system-analysis or scenario
descriptions, so it can run in a scope a fifth the size.

That is worth knowing before scaling the analysis to a programme. It is also worth checking
rather than assuming: an absence query run in a scope that does not import the description
holding the relationship will report a gap that is not there.

```compose
template: https://www.modelware.io/sierra/operational-analysis/traceability
```
