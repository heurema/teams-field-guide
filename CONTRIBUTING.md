# Contributing

This guide is maintained by the community. Contributions that improve accuracy, add verified patterns, or document newly confirmed bugs and workarounds are welcome.

## Reporting Bugs and Limitations

If you discover a new bug or limitation:

1. Open a GitHub issue in this repository describing:
   - Claude Code version (`claude --version`)
   - The behavior you observed
   - Steps to reproduce
   - Whether an issue exists in the [anthropics/claude-code](https://github.com/anthropics/claude-code/issues) tracker (include issue number if so)

2. If you have a workaround, include it in the issue.

Format for entries in the limitations table (README.md §8):

```
| Description of the limitation | [#NNNNN](https://github.com/anthropics/claude-code/issues/NNNNN) | Workaround description |
```

## Adding Orchestration Patterns

New patterns should meet these criteria:

- Tested in a real project, not just described theoretically
- Clearly distinct from the 9 existing patterns
- Include: name, one-line description, when to use, and a pseudocode example using `Task()` calls

Submit as a pull request editing the "Orchestration Patterns" section of README.md.

## Updating Fixed Limitations

When a bug in the limitations table is fixed in a new Claude Code release:

1. Update the table entry to note the fix and the version it was fixed in
2. Keep the row — fixed bugs are historical record and help users understand version requirements
3. Format: add `Fixed in vX.Y.Z` to the Workaround column

## Updating the Ecosystem Catalog

To add a project to `docs/ecosystem.md`:

- Include: repo link, star count (or `—` if unknown/low), one-line description
- Place in the correct category
- Projects do not need to be popular, but should be functional and relevant

## Style Guidelines

- No personal references or attribution in documentation
- Professional, direct tone — no marketing language
- All Claude Code issue links should be full URLs: `https://github.com/anthropics/claude-code/issues/NNNNN`
- Version numbers should be specific: `v2.1.47`, not "recent versions"
- Include the verification date for time-sensitive information (e.g., bug status, star counts)
- No emojis except ⚠️ for warnings about experimental or undocumented features

## What Not to Contribute

- Unverified speculation about future features
- Content that duplicates official Anthropic documentation without adding practical context
- Promotional content for commercial products
