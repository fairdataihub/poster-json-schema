# Contributing to poster-json-schema

First off, thank you for considering contributing to poster-json-schema! It's contributors like you that help make scientific poster metadata more accessible and FAIR.

## Why Read These Guidelines?

Following these guidelines helps communicate that you respect the time of the developers maintaining this open source project. In return, they should reciprocate that respect in addressing your issue, assessing changes, and helping you finalize your pull requests.

## What Contributions We're Looking For

poster-json-schema is an open source project and we welcome contributions from the community! There are many ways to contribute:

- **Schema improvements** - Suggestions for new fields, better descriptions, or DataCite alignment
- **Bug reports** - Found an issue with the schema? Let us know
- **Documentation improvements** - Clarify confusing sections, fix typos, add examples
- **Example JSON files** - Real-world poster examples that demonstrate the schema
- **Validation feedback** - Report edge cases where the schema doesn't fit your poster metadata

## What We're Not Looking For

- Please don't use the issue tracker for support questions about poster extraction. For that, see [poster2json](https://github.com/fairdataihub/poster2json)
- Large schema changes without prior discussion - please open an issue first

## Ground Rules

**Responsibilities:**

- Ensure backwards compatibility when proposing schema changes, or clearly document breaking changes
- Write clear commit messages describing what changed and why
- Create issues for major changes and get community feedback before submitting PRs
- Be welcoming and respectful to newcomers and contributors from all backgrounds
- Validate example JSON files against the schema before submitting

## Your First Contribution

Unsure where to begin? Look for issues labeled:

- `good first issue` - Issues suitable for newcomers
- `help wanted` - Issues where we'd appreciate community help
- `documentation` - Improvements to docs, often good for first-timers

**Never contributed to open source before?** Check out [How to Contribute to an Open Source Project on GitHub](https://egghead.io/series/how-to-contribute-to-an-open-source-project-on-github) - it's free!

## Getting Started

### For Schema Contributions

1. Fork the repository
2. Clone your fork: `git clone https://github.com/YOUR-USERNAME/poster-json-schema.git`
3. Create a branch: `git checkout -b my-feature`
4. Make your changes to `poster_schema.json`
5. Validate the schema is valid JSON Schema: `ajv compile -s poster_schema.json`
6. Update example files in `example/` if needed
7. Commit your changes with a clear message
8. Push to your fork and submit a Pull Request

### For Small/Obvious Fixes

Small contributions like fixing typos, improving comments, or correcting documentation can be submitted directly as a PR without creating an issue first.

Examples of obvious fixes:
- Spelling/grammar corrections
- Typo fixes in descriptions or documentation
- Formatting improvements
- Improved examples

## How to Report a Bug

### Security Issues

**If you find a security vulnerability, do NOT open a public issue.** Please email bpatel@calmi2.org directly.

### Bug Reports

When filing a bug report, please include:

1. **What field(s)** are affected?
2. **What did you expect** the schema to allow/disallow?
3. **What actually happened?** (include validation error if applicable)
4. **Example JSON** that demonstrates the issue

## Schema Style

- Follow [JSON Schema 2020-12](https://json-schema.org/specification) conventions
- Use descriptive field names in camelCase
- Include helpful `description` and `examples` for all fields
- Align with [DataCite Metadata Schema 4.6](https://schema.datacite.org/) where applicable

## Questions?

Open an issue with the `question` label and we'll do our best to help!

---

Thank you for contributing to poster-json-schema and helping make scientific poster 