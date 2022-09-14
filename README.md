# Pants Github Actions

This repository houses actions used by the pantsbuild projects. Other use is not discouraged, but
it's also not actively supported at this time.

When adding a new action, just claim a meaningful top-level directory name to house it and then
consume it as described [here][1].

No tagging protocol is established yet; so it's probably wise to depend on a specific sha.

[1]: https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#example-using-a-public-action-in-a-subdirectory
