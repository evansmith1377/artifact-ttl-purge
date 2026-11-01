# artifact-ttl-purge

A GitHub Action (Node 24 runtime) that deletes workflow artifacts older than a
human-readable expiry such as `7days`. It's a maintained fork of
`kolpav/purge-artifacts-action`, and every evansmith1377 repo's purge workflow
runs it, pinned by SHA. README.md documents the inputs, outputs and release.

## Commands

| What | Command |
|---|---|
| Install | `npm ci` |
| Typecheck | `npm run build` (`tsc --noEmit`) |
| Lint / format check | `npm run lint` · `npm run format-check` |
| Bundle `dist/` | `npm run pack` (esbuild) |
| Test | `npm test` (Jest, ESM) |

**Full check:** `npm run all` (typecheck, format check, lint, bundle, test),
the same command CI and the release workflow run.

## Layout

- `src/main.ts`: lists artifacts, applies the age and prefix filters, deletes,
  and sets the `deleted-artifacts` output. `src/utils.ts` holds the pure
  helpers; `src/index.ts` only calls `run()`.
- `__tests__/`: Jest tests for both.
- `dist/index.js`: the bundled, **committed** entrypoint that `action.yml` runs.
- `.github/workflows/test.yml` also runs the action against itself (`uses: ./`).

## Rules for this repo

- **Rebuild and commit `dist/` in the same change as `src/`** (`npm run pack`).
  Consumers run `dist/index.js`, not `src/`, and the release workflow fails
  when they differ.
- **Every consumer pins this action by SHA**, so a release only reaches them
  when Dependabot or a re-seed moves the pin. Keep inputs and outputs backward
  compatible within a major version.
- **Deleting is the action's job; be careful with what qualifies.** An
  `expire-in` of `0` deletes everything, which is the bug claude-seed's purge
  workflow exists to avoid. Don't change filter semantics without a test.

<!-- seed:claude-md-house-style -->
## Working agreements

### Done means verified

A change is done when `npm run all` passes and you have checked the
behavior you changed. That might mean running the test that covers it, the CLI,
or the page in a browser. "It compiles" is not done. If you couldn't verify
something, say exactly what and why. Don't imply you did.

### Scope

Do what was asked. If you notice something else worth fixing, mention it at the
end instead of fixing it in the same change. Unrequested refactors make diffs
harder to review and harder to revert. When the request is ambiguous, ask once,
concretely. When a sensible default exists, pick it and say so.

### Fit the codebase

Before writing something new, look for the existing helper, component, or
pattern, and use it. Match the surrounding naming, error handling, and test
style even where you'd choose differently. Consistency is cheaper than
cleverness. No new dependency without saying what it buys over the platform or
what's already installed.

### Comments

The test for a comment is: **would a reader get something wrong without it?**
Keep comments that record a non-obvious constraint, a rejected alternative, or
an invariant the code can't express. Delete ones that restate the code or
narrate the happy path. Keep them to one or two lines. A long comment guarding
a short check gets skipped, and the one sentence that mattered goes with it.

### Tests

Behavior changes come with tests in the repo's existing style. A bug fix starts
with a test that reproduces the bug. Before trusting a new test, make it fail
once on purpose. Never skip, delete, or loosen a failing test to get green
without saying so. Name tests for the behavior they pin (`a rejected save shows
the server error`), and before adding a test file, look for the one that
already covers the area. A second home for an area starts with one stray test.

### Git

Work on a branch, never directly on the default branch. Commit in coherent
steps, with messages that say what changed and why. Don't push, tag, force-push,
or rewrite published history unless asked. Never commit secrets, `.env` files,
or personal data.

### Raise risks before the PR, not in it

Before calling a branch done, tell the user about the risks you know of: gaps,
shortcuts, unhandled edge cases, anything a reviewer would flag. Ask which to
fix now, while changing course is still cheap. Only the ones they defer go
into the PR, as known follow-ups.

### When blocked

If something fails in a way you don't understand, stop and investigate the cause
before trying alternatives. Don't retry the same command hoping for a different
result, and don't work around a failing check by disabling it. The same goes for
security controls: never loosen a CSP, a permission, an auth check or a
container setting to make something work. Raise it instead. If a permission
or environment limit is the blocker, say so plainly and suggest what the user
can do.
<!-- /seed:claude-md-house-style -->

<!-- seed:github-hygiene -->
## GitHub workflows

- **Pin every action to a full commit SHA**, with the release in a trailing
  comment: `uses: actions/checkout@<sha> # v7.0.1`. A tag can be moved to
  different code; a SHA can't. Dependabot keeps the pins current.
- **Start from no token permissions.** Set `permissions: {}` (or `contents:
  read`) at the top, and grant each job only what it uses.
- **Every `upload-artifact` sets `retention-days`**, short for hand-offs
  between jobs. The scheduled purge is a backstop, not the cleanup.
- **From an agent session, release by dispatching the release workflow**
  (`gh workflow run`, which asks first), never by pushing a tag. Pushed with
  your credentials, a tag starts the release behind an ordinary `git push`
  prompt that doesn't say it releases; pushed with a workflow's
  `GITHUB_TOKEN`, it silently starts nothing.
<!-- /seed:github-hygiene -->
