# dotmemo spec

Write naturally, as you do on paper, with no syntax to learn; let `dotmemo` make
sense of the rest.

A `.dotmemo` file is plain UTF-8 text bringing together notes, lists, amounts,
currencies, dates, times, contacts, links, and calculations just as you would
jot them down. Behind that everyday freedom sits a deterministic grammar,
lossless text preservation, and strict execution bounds built for reliable,
real-world tools.

<p align="center">
  <img src=".github/assets/logo.svg" alt="dotmemo" height="150" />
</p>

This repository holds the specification for the `.dotmemo` format (grammar
`dotmemo/1`), along with JSON examples, conformance test cases, and supporting
tools needed to build a conforming reader, parser, or evaluator.

## Read the spec

- **Specification:** [`dotmemo/1` format spec](spec.md)
- **Changelog:** [`CHANGELOG.md`](CHANGELOG.md)

This is a working copy of Draft 1 for implementers of readers, parsers, and
evaluators. Section 20 and the appendices still need editorial review before
the first public release.

## Feedback and Questions

- **Issues and proposals:** open an issue at
  <https://github.com/dotmemo/dotmemo-spec/issues/new> to report an ambiguity,
  suggest an editorial fix, or propose a change.
- **Questions:** use [GitHub Discussions](https://github.com/dotmemo/dotmemo-spec/discussions)
  for general questions so issue threads stay focused.
- **Contribution guide:** see [`CONTRIBUTING.md`](CONTRIBUTING.md).
