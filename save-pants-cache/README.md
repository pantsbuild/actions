# save-pants-cache

Companion to [`init-pants`](../init-pants) that persists Pants caches.

`init-pants` restores caches at the start of a job. This action saves them at
the end. Using `if: always()` ensures saves survive workflow cancellation
(e.g., `cancel-in-progress: true`), which post-step hooks do not.

## Usage

```yaml
      - name: Initialize Pants
        id: pants-init
        uses: pantsbuild/actions/init-pants@v11
        with:
          named-caches-hash: ${{ hashFiles('lockfiles/*.json') }}

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

All inputs come from `init-pants` outputs. GHA cache keys are immutable, so
`actions/cache/save` is a safe no-op when the exact key already exists.
