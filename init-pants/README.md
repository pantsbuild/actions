# init-pants-action

GitHub Action that installs pants and prepares the pants caches.

Currently, this action only supports storing the pants caches in the GHA cache.
Eventually, this should become configurable to support projects that use remote caching.

This action manages three GHA caches:
1. The pants `setup` cache.
2. The pants `named_caches` cache.
3. The pants `lmdb_store` cache.

Several input parameters are used to generate the GHA cache keys for each of these.

1. The `setup` cache key uses:
   `pants-python-version` and the `pants_version` extracted from `pants.toml`.
2. The `named_caches` cache key uses:
   `gha-cache-key` and `named-caches-hash`.
3. The `lmdb_store` cache key uses:
   `gha-cache-key` and the SHA of the current commit or the latest commit from the `base-branch`.

We use the latest base commit for the `lmdb_store` so that for pull requests,
the local pants cache does not have any hits from commits in the pull request.

If you need to ignore old caches, please change `gha-cache-key`.

## Output

This action has no output.

## Input

### Required input arguments

`gha-cache-key`: This is used to create the GHA cache keys for pants' `lmdb_store`
and `named_caches`. When pulling the pants files from the GHA cache,
this can be used to include relevant metadata (such as the python version your app is using),
or to bust the cache, discarding old versions of the cached pants metadata.

`named-caches-hash`: This is used to create the GHA cache key for pants' `named_caches`.
Pants keeps pip and pex caches in the named caches, so they need to be invalidated
when transitive dependencies change. The `named-cachees-hash` should use the
`filesHash()` function to create a hash of the relevant files. We recommend using
lockfiles for this, but you can also hash `requirements.txt` and any `BUILD` files that
define other dependencies. For example: `${{ hashFiles('lockfiles/*.json') }}`

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
	  named-caches-hash: ${{ hashFiles('lockfiles/*.json', '**/something-else.lock') }}
```
