# The CLA check after a signature

Why a signed contributor's `cla/ribose` check turns green by itself
within minutes, and what an operator does when it does not. The check
is produced by the Ribose CLA App from webhook deliveries — it is not
an Action, it never waits for a maintainer approval, and it is
fail-closed: a store, config or GitHub failure shows as a red check
with the reason, never a silent pass.

## What you do as the maintainer

Nothing, in the ordinary case. The contributor signs — the portal or
`@ribose-cla agree` in the pull request — and the check turns green on
its own. Do not ask them to reopen the pull request and do not re-run
the check: both flip it, and both hide the fact that the sweep never
ran.

When the check is still red after a signature:

1. **Read the bot's comment on the pull request.** It states the case
   in words: who must sign, which agreement covers, or which edition
   went stale.
2. **Give it one sweep tick.** The sweep runs every five minutes and
   processes one queued signer per tick, so a check red for less than
   about ten minutes is still on its way. The wait is the signer's own
   pull-request count, not the size of the estate.
3. **If it is still red, re-evaluate that one pull request** with the
   ops token:
   ```text
   POST /admin/evaluate   {"repo":"owner/name","pr":123}
   ```
   The App re-runs the kernel for that pull request and writes the
   check run. Asking the contributor to push works too — any push
   re-evaluates — if you would rather not use the ops token.
4. **If the check is green but no test ever started**, that is a
   different gate: GitHub's first-time-contributor hold on Actions
   runs. One approval click resolves it — see
   [The first-time-contributor CI hold](first-time-contributor-ci-hold.md).

The ops routes (`/admin/evaluate`, `/admin/sweep`) need the estate's
ops token and do not exist at all without it — an unset token closes
the surface rather than leaving it open.

## What the machinery does

```
signature (portal, or @ribose-cla agree in the pull request)
  → the record lands in the data repo, append-only
  → the signer joins the D1 sweep queue
  → the cron tick (every 5 minutes) runs one bounded step:
      claim the oldest queued signer
      → read that signer's open pull requests from the D1 open_prs
        index — no enumeration of installations or repositories
      → evaluate each pull request (kernel) and write the cla/ribose
        check run
```

The index is the `open_prs` table (D1 migration 0005: repo, pr, login,
installation_id, head_sha). The App fills it from the `pull_request`
webhooks it already receives — opened, synchronize and reopened
upsert a row, closed removes one — so "which pull requests does this
signer have open" is a query, not a walk of the estate.

The legacy walk stays as the fallback for pull requests opened before
the index existed, and it feeds the index as it discovers them, so the
estate converges on the fast path by itself. Repeated `POST
/admin/sweep` calls backfill it — one queued signer per call, the
same bound as a cron tick — and after that every sweep takes the fast
path. Index upkeep is best-effort everywhere: a failed upsert falls
back to the walk and never blocks an evaluation.

## How to read a stuck check

| Symptom | Meaning | Remedy |
|---|---|---|
| Red, comment names who must sign | No coverage yet | The contributor signs, then waits one tick |
| Red for a few minutes after a signature | The sweep has not reached it | Wait — one tick, then re-evaluate if needed |
| Red for more than ~10 minutes after a signature | The sweep did not reach it | `POST /admin/evaluate` for that pull request |
| Red with an honest reason in the comment | Store, config or GitHub failure | Fail-closed by design; the comment names the failure |
| Green, but no test ran | GitHub's first-time-contributor hold | Approve and run — a different gate |

## Why the sweep reads an index

The sweep used to walk the estate: for every queued signer it listed
every installation, every repository in it, and every open pull request
in that repository, at a budget of 35 API calls per five-minute tick.
With roughly 56 installations and hundreds of repositories, one
signer's traversal cost hundreds of calls — hours of ticks before the
walk reached a given pull request — and the identical walk re-ran for
every signer.

rnpgp/rnp#2587 (2026-11-19) measured it: the contributor signed the
ICLA at 16:48 and the check stayed red until 22:30, when a maintainer
reopened the pull request and the re-evaluation turned it green — five
hours and forty-two minutes. The walk predated the App's own knowledge
of which pull requests were open, and the `pull_request` webhooks the
App already receives deliver exactly that.

The fix ([riboseinc/cla#133](https://github.com/riboseinc/cla/pull/133),
merged 2026-11-20) added the index and made the sweep read it: the
signer's own pull requests, O(their count), zero enumeration.

Consequence for the operator: a check still red more than ten minutes
after a signature is an anomaly to investigate, not the expected wait.
Before that date it was the expected wait, and a maintainer reopening
the pull request was the only thing that moved it.
