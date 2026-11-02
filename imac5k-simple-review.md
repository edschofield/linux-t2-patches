# Simple V6 r1

This branch contains **r1**.

# Simple V6: three alternatives for review

These are alternative flattened replacements for 7003–7010, against Linux v7.3
(`c03c1a14507dfaf8b856c2608f9ac216021fc222`). Apply one, not all three.
Other T2 patches are unchanged from the published Simple V5 collection.
This whole T2 collection is not claimed to be a newly boot-tested kernel.

| Version | Timing shortcut | Added/deleted lines | Equivalent tested image |
| --- | --- | --- | --- |
| r1 | Removed | +178/−11 | NOSYNC |
| r2 | Original 7003 shortcut | +185/−11 | CAND (`01882b7bf`) |
| r3 | Requires matching h_total, v_total and pix_clk_100hz | +188/−11 | NARROW |

Each patch changes 12 existing files. The code is otherwise identical.
Original 7003 and production Simple V5 both already contained the shortcut.
r1 deliberately removes it; r3 deliberately narrows it. Those alternatives
request a change to Atharva's earlier requirement to keep it unchanged.

Common changes preserve the reviewed combined-r1 implementation: the V5 resume
fixes, two 100 ms sleeps instead of polling, removal of the two genlock hunks,
and the scaling and connector fixture guards. They preserve normal S3 locking,
use the original scaling override, and add no GPU-reset workaround or UCLK work.
The sleep allows more time than the measured waits (up to about 53 ms at these
sites), at a measured resume cost of about 65 ms. It does not verify readiness.
The 0x4F1 write re-arms the companion after primary power-up; it remains unchecked,
as in the reviewed combined implementation. Genlock removal is shared by all
three; it is not a new change made for these exports.

The failure evidence favors r3 for further review: it preserves the equal-frame
shortcut while avoiding the observed unequal-mode failure. It is not a general
proof that these three comparisons cover every safe synchronization case.

Each export was applied to a pristine v7.3 index and every affected blob compared
with its source worktree. The complete AMD subtree matches the corresponding
existing experimental source. `verification.json` records the identities.
No diagnostic hunks or kernel test changes are included.

Checkpatch was run on all three exports. r1 has one missing-sign-off error and
three warnings; r2/r3 have that error and five warnings. One warning is the
unsigned Co-developed-by trailer; the rest are inherited line lengths and a
missing blank line after declarations. r3 adds no diagnostic over r2. These are
not lint-clean submission patches. Owners must confirm final sign-offs, and
assistance attribution must be finalized before amd-gfx submission. Codex
(GPT-6) assisted with this preparation; no approval or sign-off is inferred.

The 2 November unequal-mode results are 10/10 clean for NOSYNC and 10/10 valid
clean for NARROW, versus seven hard stops in seven second-client CAND trials
and one in one production V5 trial. DML-only still failed; OTG-only was clean.
The firmware explanation is provisional. Nobody observed these images, and
NOSYNC/NARROW still need resume, DPMS, Cinema and attended checks. A JPEG boot
timeout remains unresolved. iMac20,2 is untested. See the accompanying draft
for the measured details; publishing these review branches is not submission.
