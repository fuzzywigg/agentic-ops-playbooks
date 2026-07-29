# Exit Codes for Unattended Jobs: 0 / 1 / 2

*A tiny convention that fixes a large problem in scheduled agent fleets.*

## The problem

Audit and monitoring scripts have two completely different "bad" outcomes:

- **They worked, and found something.** (The entire point of their existence.)
- **They broke.** (Crash, missing dependency, auth failure, timeout.)

Most schedulers collapse both into "nonzero exit = failure." So the job that
*successfully found problems* and the job that *crashed before checking anything*
produce the same red row. Within a week, the operator learns that red rows are
usually just findings — and stops reading them. Now real crashes are invisible.
The alerting layer has trained its only consumer to ignore it.

We found this live: six scheduled audit jobs showing `last=error` daily. All six
were *working correctly* — exiting 1 by design on findings. The actual failures of
other jobs scrolled past in the same color.

## The convention

Every unattended script exits with exactly one of three codes:

| Exit | Meaning | Scheduler should treat as |
|---|---|---|
| **0** | Ran clean, nothing to report | success |
| **1** | Ran correctly, **found findings** | success **with a report to deliver** |
| **2** | Harness failure — the script itself broke | failure, alert |

Put the contract in the script header where the next maintainer will see it:

```
# USAGE: exit 0 = clean · 1 = findings-to-review · 2 = harness failure
```

## The scheduler-boundary shim

Your scheduler probably can't distinguish exit codes natively. Map at the job
definition, not in the script:

```sh
run-the-audit; rc=$?
# 1 = findings (success + report). Anything else propagates.
if [ $rc -eq 1 ]; then exit 0; else exit $rc; fi
```

The findings still reach the operator — through the job's *output/delivery*
channel, which is where reports belong — while the job status stays green.
Status answers "is the machinery working?"; the report answers "what did it find?"
Conflating them is the original sin.

## Corollaries

- **`set -e` is wrong for audit scripts.** A grep with no matches returns 1;
  under `set -e` your "nothing found" path becomes a crash. Handle codes
  explicitly.
- **A failed backup must not look like a good one.** If verification fails,
  delete the artifact *and* exit 2. An unverified backup that exists is more
  dangerous than no backup — it ends searches.
- **Timeouts are 2, not 1.** A probe that timed out learned nothing. Recording
  it as a finding poisons your data with verdicts you never reached. (Same rule
  as capability probing: a transport error is not a capability verdict.)
- **Test the mapping live.** Force-run the job once, confirm the scheduler shows
  success *and* the findings arrived in the delivery channel. A shim that has
  never fired is — say it with me — a story, not a control.

## Why so small a thing matters

Alert fatigue isn't caused by too many alerts. It's caused by alerts that don't
mean anything. Three exit codes and a four-line shim make every red row mean
"machinery broken" — and a red row that always means something is the cheapest
observability upgrade an unattended fleet can get.
