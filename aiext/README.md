# aiext

Planning home for this fork: everything we write that is not app code. Upstream's own docs stay in `docs/`.

Create each subfolder when its first file arrives:

- `GLOSSARY.md`: domain terms (domain-modeling skill).
- `agents/`: config for the engineering skills (issue tracker, triage labels, domain docs).
- `architecture/`: a dated, frozen baseline of upstream's architecture (HTML, open in a browser) and `changing-safely.md`, the living guide to changing or extending it without breaking saved data or upstream merges.
- `adr/`: architecture decisions, `NNNN-slug.md`.
- `research/`: findings from the research skill, one file per question.
- `spec/`: PRD and feature specs.

Where plugin code lives is still open: it gets an ADR once the plugin architecture is designed.
