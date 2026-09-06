# Artifact TTL Purge

A GitHub Action that deletes workflow artifacts older than a human-readable
expiry duration (a "time to live"). Useful for keeping artifact storage under
control until GitHub offers this natively.

This is a maintained, modernized fork of
[`kolpav/purge-artifacts-action`](https://github.com/kolpav/purge-artifacts-action)
(original author: Pavel Kolář). It runs on the Node 24 action runtime.

## Permissions

Deleting artifacts requires the workflow token to have `actions: write`. Grant
it in the workflow (or job), otherwise the action fails with
`Resource not accessible by integration`:

```yaml
permissions:
  actions: write
```

## Inputs

| Input          | Required | Default               | Description                                                                                                       |
| -------------- | -------- | --------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `expire-in`    | yes      | —                     | Max artifact age; anything older is deleted. Human-readable, e.g. `30 minutes`, `1 week`. `0` deletes everything. |
| `token`        | no       | `${{ github.token }}` | Token used to list and delete artifacts. Must carry `actions: write` (see Permissions above).                     |
| `onlyPrefix`   | no       | `''`                  | If set, only artifacts whose name starts with this prefix are eligible for deletion.                              |
| `exceptPrefix` | no       | `''`                  | Artifacts whose name starts with this prefix are never deleted (takes precedence over `onlyPrefix`).              |

Durations are parsed by
[`parse-duration`](https://github.com/jkroso/parse-duration); see it for the
full list of supported formats (`10 minutes`, `1hr 20mins`, `1week`, ...).

## Outputs

| Output              | Description                                                                      |
| ------------------- | -------------------------------------------------------------------------------- |
| `deleted-artifacts` | Serialized JSON array of the deleted artifacts; `"[]"` when nothing was deleted. |

## Usage

Run it on a schedule. Artifacts of in-progress workflows are not affected, since
they are only persisted after the workflow completes.

```yaml
name: 'Purge old artifacts'
on:
  schedule:
    - cron: '0 * * * *' # every hour

permissions:
  actions: write

jobs:
  purge:
    runs-on: ubuntu-latest
    steps:
      - uses: evansmith1377/artifact-ttl-purge@v1
        with:
          expire-in: 7days # 0 deletes all artifacts
```

### Only delete certain artifacts

```yaml
with:
  expire-in: 7days
  onlyPrefix: tmp_ # only purge artifacts whose name starts with "tmp_"
```

### Exclude certain artifacts

```yaml
with:
  expire-in: 7days
  exceptPrefix: prod_ # never purge artifacts whose name starts with "prod_"
```

### Using the output

```yaml
- uses: evansmith1377/artifact-ttl-purge@v1
  id: purge
  with:
    expire-in: 7days
- run: echo "Deleted: ${{ steps.purge.outputs.deleted-artifacts }}"
```

## Notes

- If you hit the storage size limit, you can temporarily switch to `on: push`
  to run it immediately.
- Even after the action succeeds, it can take a few minutes for the artifacts to
  actually disappear from the UI.
- The action paginates through all repository artifacts and deletes expired ones
  with retry/throttling so it stays within GitHub's rate limits. A failure to
  delete an individual artifact is reported but does not abort the rest.

## Development

```bash
npm ci
npm run all   # typecheck, format-check, lint, bundle (dist/), test
```

`dist/index.js` is the bundled, committed entrypoint; rebuild it with
`npm run pack` and commit the result whenever `src/` changes (the release
workflow enforces this).

## Releasing

The [release workflow](./.github/workflows/release.yml) only runs when a
`vX.Y.Z` tag is pushed - it does **not** bump the version itself. Cutting a
release is a manual, two-step process:

1. **Bump the version** in `package.json` (and `package-lock.json`, via
   `npm ci`/`npm install`), commit it (e.g. `Bump version 1.1.1 -> 1.1.2`),
   and merge it to `main`.
2. **Tag `main`**, either:
   - locally:

     ```bash
     git tag vX.Y.Z
     git push origin vX.Y.Z
     ```

   - or by running the [tag release workflow](./.github/workflows/tag-release.yml)
     from the Actions tab (`workflow_dispatch`), which tags `main` at its
     current `package.json` version and pushes the tag for you - useful if
     your local credentials aren't allowed to push tags directly.

Pushing the tag triggers the release workflow, which:

- Runs `npm run all` (typecheck, format-check, lint, bundle, test).
- Fails if `dist/` doesn't match what's committed, so bump the version and
  rebuild `dist/` (`npm run pack`) in the same commit.
- Creates a GitHub release for the tag with auto-generated notes.
- Force-moves the major version tag (e.g. `v1`) to point at the new tag, so
  consumers pinned to `uses: .../artifact-ttl-purge@v1` pick up the change.

## License

MIT. See [LICENSE](./LICENSE).
