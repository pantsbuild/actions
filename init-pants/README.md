# init-pants-action

GitHub Action that installs pants and prepares the pants caches.

This action manages three GHA caches:
1. The pants `setup` cache.
2. The pants `named_caches` cache.
3. The pants `lmdb_store` cache (optional).

Several input parameters are used to generate the GHA cache keys for each of these.

1. The `setup` cache key uses:
   `pants-python-version` and the `pants_version` extracted from `pants.toml`.
2. The `named_caches` cache key uses:
   `gha-cache-key` and `named-caches-hash`.
3. The `lmdb_store` cache key uses (if enabled):
   `gha-cache-key` and the SHA of the current commit or the latest commit from the `base-branch`.

We use the latest base commit for the `lmdb_store` so that for pull requests,
the local pants cache does not have any hits from commits in the pull request.

If you need to ignore old caches, please change `gha-cache-key`.

## Output

The following outputs are available for passing to the
[`save-pants-cache`](../save-pants-cache) companion action:

| Output | Description |
|--------|-------------|
| `setup-cache-path` | Path to the Pants setup cache directory |
| `setup-cache-key` | GHA cache key for the Pants setup cache |
| `named-caches-path` | Path to the Pants named caches directory |
| `named-caches-key` | GHA cache key for the Pants named caches |
| `lmdb-store-path` | Path to the Pants LMDB store directory |
| `lmdb-store-key` | GHA cache key for the Pants LMDB store |
| `cache-lmdb-store` | Whether the LMDB store cache is enabled |
| `named-caches-hash` | The named caches hash value |

## Input

### Required input arguments

`named-caches-hash`: This is used to create the GHA cache key for pants' `named_caches`.
Pants keeps pip and pex caches in the named caches, so they need to be invalidated
when transitive dependencies change. The `named-caches-hash` should use the
`hashFiles()` function to create a hash of the relevant files. We recommend using
lockfiles for this, because named caches are primarily used today for the results of
resolving lockfiles. If you aren't using lockfiles, you can also hash
`requirements.txt` and any `BUILD` files that define other dependencies.
For example: `${{ hashFiles('lockfiles/*.json') }}`

### Optional input arguments (alphabetical order)

`base-branch`: This action calculates the merge base for pull requests from this branch.
Looking up the merge commit allows us to use the cache from the latest commit on the base branch.

`cache-lmdb-store`: Pass the string `'true'` to enable caching pants' lmdb store in
a GHA cache. By default, this action does NOT cache the `lmdb_store`.
This is a very coarse cache that can grow unbounded. So, it is very likely to hit
GitHub's 10GB per repo max for action caches. If you enable this, you need another
process or workflow to manage discarding older GHA caches, or minimizing the cache size
as described in the [docs](https://www.pantsbuild.org/docs/using-pants-in-ci).
Use the default if using [remote caching](https://www.pantsbuild.org/docs/remote-caching).

`experimental-remote-cache-via-gha`: This is used to configure the remote caching address
and oauth token so that pants can use GHA as a fine-grained remote cache. You must also
configure the other remote caching options in `pants.ci.toml` or similar as described in
[remote caching](https://www.pantsbuild.org/2.20/docs/using-pants/remote-caching-and-execution/remote-caching#github-actions-cache).

`get-pants-version`: This is used to override the version of scie-pants
downloaded by `get-pants.sh`. The default is the latest version. To specify a
version set this to a release, e.g. `0.12.0`.

`gh-host`: This is used to add an enterprise GitHub host for API calls instead of `github.com`.

`gha-cache-key`: This is used to create the GHA cache keys for pants' `lmdb_store`
and `named_caches`. When pulling the pants files from the GHA cache,
this can be used to include relevant metadata (such as the python version your app is using),
or to bust the cache, discarding old versions of the cached pants metadata.

`lmdb-store-location`: This is used to override the lmdb store cache location when the location
has been customized in `pants.toml`.

`named-caches-location`: This is used to override the named cache location when the location
has been customized in `pants.toml`.

`pants-ci-config`: The value for the `PANTS_CONFIG_FILES` environment var.
Set to empty to skip adding it to the environment for the rest of the workflow.
Defaults to `pants.ci.toml`.
For more about this var and the file naming convention, see:
https://www.pantsbuild.org/docs/using-pants-in-ci#configuring-pants-for-ci-pantscitoml-optional

`setup-commit`: Which version/commit of get-pants.sh script to use when installing pants.

`setup-python-for-plugins`: Ensure python is installed for linting/testing pants-plugins.

## Secrets

This action does not use any secrets at this point. It might need some once it supports projects that use remote caching.

## Environment variables

This action sets the `PANTS_CONFIG_FILES` environment var using the value of the `pants-ci-config` input parameter.
The environment variable should be available for the remainder of the workflow that uses this action.

## Usage Example

This action only restores caches. You must add `save-pants-cache` with
`if: always()` at the end of your job to persist them. This ensures cache
saves survive workflow cancellation (e.g., `cancel-in-progress: true`).

```yaml
      - name: Initialize Pants
        id: pants-init
        uses: pantsbuild/actions/init-pants@v11
        with:
          # cache0 makes it easy to bust the cache if needed
          gha-cache-key: cache0-py${{ matrix.python_version }}
          named-caches-hash: ${{ hashFiles('lockfiles/*.json', '**/something-else.lock') }}

      # ... your build/test steps ...

      - name: Save Pants caches
        if: always()
        uses: pantsbuild/actions/save-pants-cache@v11
        with:
          setup-cache-path: ${{ steps.pants-init.outputs.setup-cache-path }}
          setup-cache-key: ${{ steps.pants-init.outputs.setup-cache-key }}
          named-caches-path: ${{ steps.pants-init.outputs.named-caches-path }}
          named-caches-key: ${{ steps.pants-init.outputs.named-caches-key }}
          named-caches-hash: ${{ steps.pants-init.outputs.named-caches-hash }}
          lmdb-store-path: ${{ steps.pants-init.outputs.lmdb-store-path }}
          lmdb-store-key: ${{ steps.pants-init.outputs.lmdb-store-key }}
          cache-lmdb-store: ${{ steps.pants-init.outputs.cache-lmdb-store }}
```

### Migration from previous versions

Previously, `init-pants` used `actions/cache@v5` which saves caches via a
post-step hook. Post-step hooks do not run on workflow cancellation, so
workflows using `cancel-in-progress: true` would lose cache saves on every
cancelled run — creating a cold-cache cycle.

To migrate: add an `id` to your `init-pants` step and add the `save-pants-cache`
step shown above at the end of your job. No other changes are needed.
