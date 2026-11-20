# Ribose maintainer skills

The estate's playbooks for the people who release and steward its
software. Each skill describes one operation end to end: what happens,
what the operator does, what the machinery does, and how to tell which
of those just failed.

## Skills

- [Gem release, PR-based flow](skills/gem-release-pr-flow.md) — how a
  Ruby gem release travels through a release pull request, why it does,
  and what to do when a step misbehaves.
- [The first-time-contributor CI hold](skills/first-time-contributor-ci-hold.md) —
  why a first-time contributor's tests wait for an approval click, and
  what the maintainer does about it.

## Conventions these skills assume

Estate posture (from the workspace CLAUDE.md, binding everywhere):
feature branches and rebase-merge pull requests, never a direct commit
to `main`; never push tags without an owner instruction; stage by
explicit path; no AI attribution anywhere in commits or pull requests.

## Related estate machinery

- [metanorma/ci](https://github.com/metanorma/ci) — the shared release
  and CI workflows every gem consumes (`rubygems-release.yml`).
- [riboseinc/cla](https://github.com/riboseinc/cla) — the Contributor
  License Agreement App gating every pull request (`cla/ribose`).
- [cimas](https://github.com/metanorma/cimas) — syncs the workflow
  templates in `metanorma/ci` into every managed gem.
