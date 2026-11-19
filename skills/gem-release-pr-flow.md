# Gem release, PR-based flow

How a Ruby gem release travels from a workflow dispatch to rubygems.org
through a pull request. This flow replaced the direct-push release
(with its GH006 rejections under the required `cla/ribose` check) in
[metanorma/ci#515](https://github.com/metanorma/ci/issues/515),
implemented in
[metanorma/ci#516](https://github.com/metanorma/ci/pull/516).

## What you do as the maintainer

One action, same as always: dispatch the release workflow with a
version.

```text
Actions → release → Run workflow
  next_version: patch   (or minor, major, x.y.z, prerelease, skip)
```

Everything after the dispatch is machinery. You never push the version
bump to `main` yourself, and you never tag locally.

## What the machinery does

```
dispatch(next_version)
  → preflight: fresh bundle resolve, gem build, Trusted-Publishing check
  → bump on a branch: release/vX.Y.Z, single commit "Bump version to X.Y.Z"
  → release pull request opens (auto-merge enabled, rebase merge)
  → the release PR runs the repository's own gates:
      rake tests  +  cla/ribose  (green: release accounts are allowlisted)
  → auto-merge lands the bump on main
  → the publish leg runs on the merge event:
      check out the merge commit
      → build the gem → publish via Trusted Publishing (OIDC)
      → create the tag vX.Y.Z and the GitHub Release on the merge commit
      → verify the gem is live on rubygems.org
```

The tag is created by CI **on the merge commit** — never locally, never
on an unmerged commit. The release tag is always reachable from `main`.

## How to read a run

- **Green dispatch run** — the release PR is open and merge-gated. The
  run summary names the pull request. The gem is **not** published yet;
  publishing follows the merge, automatically.
- **Green merge-leg run** — the gem is on rubygems.org, the tag and the
  GitHub Release exist on the merge commit.
- **Skipped release jobs on an ordinary PR close** — correct and
  deliberate: the publish leg runs only for merged pull requests whose
  head branch starts with `release/`. Skipped, never red.

## Prerequisites the flow assumes

1. **The repository setting "Allow auto-merge" is enabled.** Without
   it, the release PR sits open with checks running; the maintainer
   clicks merge, and the publish leg fires identically. The workflow
   warns in the run summary when auto-merge cannot be enabled.
2. **The PAT secret in the workflow** (for example
   `METANORMA_CI_PAT_TOKEN`) opens the pull request. This is what makes
   the release PR's own checks run — pull requests opened by the
   built-in `GITHUB_TOKEN` do not trigger workflow runs.
3. **The PAT's account can merge.** Repositories on the free GitHub
   plan cannot use branch-protection bypass allowances; a covered
   release account therefore needs repository-admin on the repos it
   releases (with `enforce_admins` off, repository administrators
   bypass required checks). This is the interim posture until cimas
   syncs the PR-based flow everywhere; at that point the grants are
   revoked — the flow no longer pushes to `main` directly.
4. **The release account is allowlisted in the CLA configuration**, so
   the `cla/ribose` check on the release PR is green by design.

## Troubleshooting

| Symptom | Meaning | Remedy |
|---|---|---|
| Dispatch run ends with "Release PR opened" and nothing published | Correct: publishing follows the merge | Nothing; wait for the merge |
| "Auto-merge unavailable" warning in the run summary | The repo lacks the auto-merge setting, or rebase merging is disabled | Merge the release PR manually once checks are green; the publish leg fires on merge |
| Release PR checks red on `cla/ribose` | The release account lost coverage or allowlisting | Check the config repo's allowlist; the check comment states the case |
| Publish leg failed after merge | Gem build or rubygems.org rejected something | Read the failed step; re-dispatch with `next_version=skip` after fixing — it re-publishes the current default-branch head and the idempotent guard skips an already-pushed version |
| "version file already at X — skipping the bump" | A re-dispatch of an already-released version | Intentional idempotent path; the publish proceeds or the guard skips as appropriate |
| Gem on rubygems.org but no tag/Release | Legacy release from before the flow | Dispatch `next_version=skip`; the ensure-tag and release-floor steps backfill |

## Why the flow is shaped this way

A required status check can only be evaluated *before* code lands
through a pull request; GitHub delivers push events only after a
direct push is accepted, so a directly pushed commit can never carry a
green required check — the wall every direct-push release hit
(GH006). Gating releases through a release PR makes the version bump
an evaluated, test-gated commit like every other, and removes the need
for any actor exemption in normal operation. The only bypass that
remains is the documented break-glass: repository administrators, who
are agreement-covered people, may push directly in an emergency.

## Per-ecosystem note

The Ruby flow is the one that needed this change (Bundler's `rake
release` pushes the bump directly). The npm and crates publication
paths are already PR-publish cultures; they adopt the same
`actions-mn/provenance-release` shape when the SBOM-and-attestation
work lands, so the release fix and the provenance emission are one
implementation.
