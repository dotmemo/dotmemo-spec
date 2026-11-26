# Contributing

Thank you for taking the time to read, test, and question `dotmemo/1`. Your
contributions, whether spotting an edge case, challenging an assumption, or
refining the wording, are what will make this specification clear, complete, and
reliable across implementations.

The guidelines below are simple suggestions to make collaboration easier for
everyone, especially while `dotmemo/1` is in draft status. Feel free to share
feedback in whatever way works best for you.

## Reporting Issues and Proposing Changes

Please use the repository's issue tracker to report an ambiguity, suggest an
editorial improvement, or discuss a change to the format. Writing issues in
English helps keep discussions easy to follow for all contributors.

When reporting an ambiguity or a difference between implementations, it helps to
include:

- The grammar version and draft or release you read;
- The section and rule in question;
- A minimal, self-contained `.dotmemo` input showing the case;
- What you expected the parser or evaluator to do, and why;
- Any alternative reading allowed by the current wording.

When proposing a change, it is helpful to note what it would mean for existing
`.dotmemo` files. If an example or conformance case is affected, please note
which case might need updating and what result you expect.

The [changelog](CHANGELOG.md) summarizes revisions to the specification. Keep
in mind that the grammar identifier (`dotmemo/1`) is distinct from a draft or
release tag.

## Questions

Please use [GitHub Discussions](https://github.com/dotmemo/dotmemo-spec/discussions) for general questions.
Keeping questions in Discussions helps prevent issue threads from becoming crowded.

If a question reveals an ambiguity or a missing detail in the specification,
it may then give rise to an issue to update the text.

## Submitting Pull Requests

- **Typos, formatting, and clarifications:** direct pull requests are always
  welcome.
- **Grammar or evaluation changes:** opening an issue first to discuss the idea
  usually saves effort before writing a pull request, though you are welcome to
  share a draft or prototype if that makes the discussion clearer.
- **Conformance test cases:** pull requests adding tricky edge cases or test
  fixtures are greatly appreciated.

## License Agreement

By submitting an issue, pull request, or other contribution to this repository,
you agree that:

1. Any text contribution may be edited and incorporated by the author into the
   official specification under the **CC BY-SA 4.0** license.
2. Any `.dotmemo` examples, JSON snippets, or conformance test cases you submit
   may be freely copied, adapted, and integrated into software or test suites
   without restriction.
