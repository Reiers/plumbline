# Post-mortem: SP test target t0143103 stopped producing blocks, 2026-10-01

**Status:** Resolved · **Published:** 2026-10-06 · Live status entry: <https://status.reiers.io>

## Impact

- **Window:** about 13:00 to 17:03 UTC, roughly 4 hours without blocks from t0143103. The collector recorded the miner as not active from 13:22 to 16:46 UTC (41 probes, about 3h 25m).
- **WindowPoSt:** one deadline missed. 17 of 1,285 sectors were marked faulty. Power was otherwise unchanged.
- **Faucet and Calix:** not affected. Lotus stayed in sync throughout.
- **Error budget:** about 47% (collector view) to 56% (block-production view) of the SP test-target 30-day budget in one event. No SLO breach.

## Timeline (UTC)

- ~12:50 to 13:10: both Curio nodes lose their machine registration in the cluster database. The task scheduler can no longer claim work, so no WinningPoSt (blocks) and no WindowPoSt.
- ~13:00: last block before the gap.
- 16:40: cause identified. Curio restarted on the first node.
- 16:48: Curio restarted on the second node.
- ~17:00: backlog of stale election tasks drained.
- 17:03: block production resumes and the miner wins and lands blocks every round again.
- 17:10: resolved on the status page.
- 2026-10-02, ~13:05: the 17 faulted sectors recovered at their next proving window. Confirmed on 2026-10-06: 1,285 live, 1,285 active, 0 faulty.

## Root cause

Both Curio nodes lost their machine registration in the cluster database at about the same time, and the scheduler had no machine to assign tasks to. Restarting Curio re-registered both nodes. Why the registrations were dropped is still not known.

## What went wrong in our response

- Detection took about 3.5 hours. Nothing alerted on "no blocks from t0143103".

## Follow-ups

1. Alert when t0143103 has not won a block for more than N expected rounds, or when the Curio machine registry is empty. *(open)*
2. Investigate why both machine registrations were lost at the same time. *(open)*
3. Add this failure mode (symptom, check, fix) to RUNBOOK.md. *(open)*
4. Confirm faulted-sector recovery. *(done: 0 faulty on 2026-10-06)*
