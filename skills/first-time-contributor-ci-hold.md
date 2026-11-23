# The first-time-contributor CI hold

Why a first-time contributor's CI appears to "not run" on their pull
request, and what a maintainer does about it. This is GitHub platform
security, not a fault in the repository's workflows, and it interacts
with the `cla/ribose` check in a way worth understanding.

## What maintainers see

A first-time contributor opens a pull request. The `cla/ribose` check
appears and evaluates normally (it is App-driven, not Actions-driven).
But the Actions workflows — the test matrix — sit in the Actions tab
with a banner: the runs are **waiting for approval**, and the Actions
tab shows an **Approve and run** button next to the pending runs. Until
a maintainer clicks it, no test executes, and the contributor sees
silence from CI rather than results.

## Why it exists

The hold is GitHub's protection against untrusted pull-request code:
a first-time contributor's CI definition changes could execute
arbitrary code on the repository's runners. The approval is a human
attestation that the pull request is safe to build. It applies to
workflows only — checks produced by GitHub Apps (like `cla/ribose`)
run regardless, which is why the CLA check evaluates while the tests
wait.

## What the maintainer does

Open the Actions tab on the pull request and click **Approve and run**.
One approval covers all the runs of that pull request, and once the
contributor's first batch is approved, their later pull requests run
automatically — the hold applies to their *first* contribution only.
This is expected behavior and the security posture stays as-is; do not
disable the requirement on public repositories.

## The CLA side needs no approval

`cla/ribose` is produced by the Ribose CLA App from a signed webhook
delivery, so signature evaluation never waits for the approval — a
first-time contributor's CLA status is visible immediately, and the
sweep flips the check green within one sweep tick (five minutes) of a
signature landing. That wait is a different thing from this hold, and
it has its own guide: [The CLA check after a
signature](cla-check-after-signature.md).

## Where this bit for real

rnpgp/rnp#2587 (2026-11-19) showed both holds at once, and they read
as one broken thing. The contributor signed the ICLA at 16:48, but the
`cla/ribose` check stayed red until 22:30, when a maintainer reopened
the pull request and the re-evaluation turned it green — five hours
and forty-two minutes, for a sweep that should have taken one tick.
The test matrix sat in the Actions tab waiting for the approval click
throughout. Two independent holds, one incident: the sweep was too
slow (a defect in the App, fixed in
[riboseinc/cla#133](https://github.com/riboseinc/cla/pull/133)), and
the CI hold was GitHub's first-time-contributor gate doing its job.
One click resolved the tests; the reopen resolved the check.
