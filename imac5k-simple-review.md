This branch contains alternative **r3**.

# iMac 5K display support: three patch alternatives

These patches add support for the two halves of the internal iMac20,1/iMac20,2
screen and restore the right half after sleep. Each is a single replacement
for patches 7003–7010 in the existing T2 Linux patch collection. Apply only one
alternative to Linux v7.3 (`c03c1a14507dfaf8b856c2608f9ac216021fc222`).
The other T2 patches on these branches are unchanged.

The driver normally compares display timings before treating two displays as
synchronized. Atharva's original iMac patch bypasses those comparisons for the
internal pair. These alternatives differ only in that shortcut:

| Version | Timing decision | Patch size |
| --- | --- | --- |
| r1 | Remove the shortcut and use the normal comparisons | +178/−11 lines |
| r2 | Keep the original shortcut unchanged | +185/−11 lines |
| r3 | Take the shortcut only when horizontal total, vertical total and pixel clock match | +188/−11 lines |

Each changes 12 existing files. r3 checks `h_total`, `v_total` and
`pix_clk_100hz`. r1 and r3 are proposed changes to Atharva's earlier request
to keep the shortcut unchanged.

All three initialise the display before the GPU power-management controller,
re-enable the right receiver after left-half power-up, retain its detected
monitor across sleep, and restore its connector identification data when needed.
They enable the left half first and restart the right half when necessary.
They also handle the right receiver's idle lane status during link training and
use the monitors' reported colour depths without per-connector depth caps.

The right receiver gets a fixed 100 ms sleep before initial detection and before
training its link. Earlier measured waits at these sites reached about 53 ms.
The fixed sleep adds about 65 ms to resume compared with the daily-use kernel's
readiness polling; it does not itself check whether the receiver is ready.
The write to receiver register `0x4F1` remains unchecked. The two original
per-frame synchronization changes are omitted in all three versions. Normal
suspend-to-RAM locking and the original scaling behavior are preserved. Small
null checks protect existing test fixtures. No diagnostic or kernel test-file
changes are included.

## What was tested

On 2 November, the display code corresponding to r1 completed 10 unequal-mode
trials without a GPU timeout; r3 completed 10 valid trials without that failure.
The trials kept the left half at its native resolution and set the right half
to its advertised 2048×2304 mode. With r2, all seven trials that opened another
GPU client after the mode change ended in a timeout, failed ring reset and full
GPU reset that did not complete. The daily-use kernel, which also contains the
original shortcut, failed its one control trial in the same way.

Further experiments separated the shortcut's two callers. Keeping it only for
the display mode library's bandwidth calculations still caused the failure.
Keeping it only for hardware timing-generator synchronization did not. This
supports r3 for further review, but does not prove that its three comparisons
cover every safe synchronization case. The proposed firmware explanation is
unconfirmed.

On 3 November, the kernel corresponding to r3 passed a short unattended smoke
test without the Cinema Display: one boot check, five display power off/on
cycles, three suspend-to-RAM cycles, one suspend at each of 2560×1440 and
3840×2160, one blank-then-suspend cycle, one seam timing run and one restart
check in each CRTC assignment. Depth stayed at 10 bits per component wherever
recorded, and the connector identification data did not change. This is not
full acceptance.

Nobody observed the panel during these trials. r3 still needs the full resume
and display power off/on repetitions, a cold start, external Cinema Display and
attended visual checks; r1 has had none of these. r2 passed the automated resume
and display-transition checks without the Cinema Display, but one boot check found
a JPEG power-gating timeout that remains unresolved. iMac20,2 has not been tested.
No benchmark comparison has been reproduced for these alternatives.

## Source and submission checks

Each patch applies to clean v7.3 and reproduces all 12 changed files from its
corresponding tested kernel. To reproduce the complete AMD driver tree also
requires T2 patch 2009, which adds a GPU-probe deferral check, and patch 6001,
which sets machine-specific power-management feature masks. Their role must
be resolved before claiming the standalone upstream patch is validated.
The whole collection of T2 patches on these branches is not claimed to have
been newly boot-tested. The accompanying verification JSON records source
identities and hashes for reproducibility.

The kernel's checkpatch tool reported one missing-sign-off error for each
alternative, plus three warnings for r1 and five for r2/r3. One warning concerns
the unsigned co-developer credit; the others concern inherited line lengths and
a missing blank line after declarations. These counts were rechecked on the final
descriptions on 3 November. Final sign-offs
and the requested Assisted-by attribution still need to be added before
submission to the AMD graphics mailing list. Claude and Codex (GPT-6) helped
with testing and preparation. These are review patches, not approved submissions.
