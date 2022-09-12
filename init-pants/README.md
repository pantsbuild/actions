# init-pants-action

GitHub Action that installs pants and prepares the pants caches.

Currently, this action only supports storing the pants caches in the GHA cache.
Eventually, this should become configurable to support projects that use remote caching.

The GHA cache key is generated using the `gha-cache-key` input parameter,
and the SHA of the merge-base commit from the `base-branch`.
We use the merge-base commit so that for pull requests, the pants cache does not have any hits
from commits in the pull request.

## Output

This action has no output.

## Input

### Required input arguments

`gha-cache-key`: This is used to create the GHA cache keys. When pulling the pants files from the GHA cache,
this can be used to include relevant metadata (such as the python version your app is using),
or to bust the cache, discarding old versions of the cached pants metadata.

### Optional input arguments

`pants-python-version`: Which version of python to install, specifically for pants. Defaults to `'3.9'`.

`base-branch`: This action calculates the merge base for pull requests from this branch.
Looking up the merge commit allows us to use the cache from the latest commit on the base branch.

`pants-ci-config`: The value for the `PANTS_CONFIG_FILES` environment var.
Set to empty to skip adding it to the environment for the rest of the workflow.
Defaults to `pants.ci.toml`.
For more about this var and the file naming convention, see:
https://www.pantsbuild.org/docs/using-pants-in-ci#configuring-pants-for-ci-pantscitoml-optional

## Secrets

This action does not use any secrets at this point. It might need some once it supports projects that use remote caching.

## Environment variables

This action sets the `PANTS_CONFIG_FILES` environment var using the value of the `pants-ci-config` input parameter.
The environment variable should be available for the remainder of the workflow that uses this action.

## Usage Example

An example of how to use this action in a workflow:

```yaml
      - name: Initialize Pants
        uses: pantsbuild/actions/init-pants@main
        with:
          # cache0 makes it easy to bust the cache if needed
          gha-cache-key: cache0-py${{ matrix.python_version }}
```
