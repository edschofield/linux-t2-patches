# iMac 5K: Simple V5 for review

Simple V5 is the small patch set
[Atharva asked for](https://github.com/t2linux/wiki/issues/743#issuecomment-6290452376),
now tested with and without an external display on one iMac20,1 /
Radeon Pro 5300. It has seven patches on top of 7003, adding 129 lines and
removing 16 in eight existing kernel files. There are no new files and no
recovery worker. The first six patches are Simple V3 unchanged. The
seventh is a two-line change that fixes the one failure V3's watched tests
found.

- [V5: seven patches](https://github.com/edschofield/linux-t2-patches/tree/draft/imac5k-simple-v5)
- [V3 to V5 comparison](https://github.com/edschofield/linux-t2-patches/compare/draft/imac5k-simple-v3...draft/imac5k-simple-v5)
- [V3: six patches](https://github.com/edschofield/linux-t2-patches/tree/draft/imac5k-simple-v3)

| Patch | Change |
| --- | --- |
| 7004 | Re-arm `0x4F1` after the left tile's D0 write. |
| 7005 | Enable the left tile before the right. |
| 7006 | Keep the right tile's sink across power-downs, restore its published EDID after a forced probe, and wait for readiness before training. |
| 7007 | Ignore the right tile's idle lane bits while still failing an unsuccessful AUX read. |
| 7008 | Restart an unchanged right tile when the left tile restarts. |
| 7009 | Wait for right-tile HPD and a successful DPCD read before boot detection, with a 150 ms timeout. |
| 7010, new in V5 | Don't attach `max bpc` to the two internal tiles, so both stay at the EDID's 10 bpc. |

All seven keep Atharva's detection-time and atomic-tail `0x4F1` writes and
his iMac20,1/20,2 scope.

## Why 7010

In V3's watched test, the right tile's image disappeared, leaving the left
tile's image across the whole panel, after the
first `max bpc` change and came back at the next one. Every automated check
passed: both streams were enabled, scanning out, flipping and trained. A
controlled sequence showed the image loss follows unequal tile depths. With
the left tile at 8 bpc and the right at 10, only the left tile's image
showed, across the whole panel. At
8/8 and 10/10 both halves showed, and restarting either tile alone at 10/10
kept or restored both. Each tile has its own `max bpc` property, and a
change on one tile restarts only that tile's stream, so a single property
change reaches the bad state.

7010 leaves the property off both internal tiles. They then always run at
the EDID depth of 10 bpc, which is the default and the state in which every
check passed. External connectors, including the Cinema's, keep the
property. The cost is that the internal panel can no longer be set to
8 bpc. `max bpc` is optional, and GNOME started normally without it.

## What was tested

All of this is from one iMac20,1, with the Apple LED Cinema Display
attached through its hub and without it.

- **Unattended, 22 October:** 72 judged stages and ten trial boots, with
  both cablings and both images (production and a diagnostic build with
  extra logging and fault injection). That covers boots, DPMS ×10, native
  S3 ×10, blank-then-S3, scaled S3 at 2560x1440 and 3840x2160, both CRTC
  assignments, dependent restarts, the idle-check control and five injected
  faults. Every capture and held-state snapshot reported 10/10 bpc, and
  neither internal tile had `max bpc` wherever properties were read. DP-1 kept its 256-byte EDID and TILE through every dark period,
  with no DP-1 hotplug event.
- **Watched, 22–23 October:** cold starts, GNOME, DPMS, S3, blank-then-S3,
  scaled S3 and console handover all looked right with both cablings. The
  check that failed V3 now passes. Both halves came back after every
  operation, in both CRTC assignments, with and without the Cinema. A
  left-tile-only `Colorspace` change also kept both halves.

## Known limits

- **Brief blink when the right tile alone turns off or on.** The whole
  panel goes dark for a fraction of a second at each transition. The left
  tile isn't restarted, but its flips stall for 117–167 ms at both
  transitions. V3's runs show the same stall. I've accepted this; the cause
  isn't known.
- **Console drawing on the Cinema stalls the iMac's flips.** While the
  console's blinking cursor or `htop` draws on the Cinema, the moving-seam
  test shows late frames on the internal panel. This happens on V5 and on
  the 7.2.10 kernel I use daily alike; the daily kernel had more late
  frames. V5 also showed occasional one-frame offsets between the two
  tiles, as earlier 7.3 builds did. With the cursor hidden, V5 had no late
  frame or offset in either CRTC assignment. Under GNOME the console
  doesn't draw on the Cinema, so this isn't a desktop case, but the cause
  isn't known.
- **A USB error on one Cinema hub replug** (`usb 4-1: Failed to suspend
  device, error -19`, 19 µs after the hub dropped off during insertion).
  The display recovered normally. It happened once and wasn't retested.
- **A forced readiness timeout trains for about 2.3 s under `dc_lock`**
  before giving up. This was seen only with an injected fault; in V3's
  records, ordinary readiness waits took 0.8 to 53 ms.
- No iMac20,2 results; colour depth on the wire isn't measured; no daily
  use yet.

## Boot detection (7009, from V3)

Without 7009, DP-1's EDID arrived after fbcon had sized its framebuffer
for eDP-1 alone. Later fbdev probes kept that size, rejected the
2880-line tile modes and left the console at 3840x2160 with DP-1 disabled.
The DRM-debug boot logs from 21 October show:

| Event, seconds after boot | Without 7009 | With 7009 |
| --- | --- | --- |
| DP-1 boot detection | No sink at 5.461 s | Sink published at 5.748 s |
| First DP-1 EDID update | 9.811 s | 5.662 s |
| fbdev framebuffer allocation | 3840x2160 at 7.717 s | 5120x2880 at 8.000 s |
| Later probe | Both tiles' 2560x2880 modes rejected with `VIRTUAL_Y`; DP-1 left with 640x480 | Both tiles have 2560x2880 modes |

The diagnostic boots measured waits of 43 to 53 ms.

## Source

The hardware-tested source uses Linux 7.3, Atharva's full 5K attachment and
the project's pinned T2 prerequisites:

| Version | Tested source commit | Kernel release |
| --- | --- | --- |
| Base | `000f24044eacaf78ea65c62f7ef73c79a33a09b8` | `7.3.0-imac-atfix7` |
| V3 | `da76349f95da474043e3a993d6c620c81ef2b523` | `7.3.0-imac-hpd73-simple-v1-r3` |
| V5 | `73ca170aaca64dda9c4b602364a94fc3d0b24cd9` | `7.3.0-imac-hpd73-simple-v5` |

This branch starts at t2linux patch-repository commit
`e6965b4babc6c73383143c989b555b151fcfbc46` (Linux 7.3). Applying all 39 of
its patches to v7.3 reproduces the tested V5 `drivers/gpu/drm/amd` subtree
exactly. Other T2 drivers differ from the hardware-tested base, and this
full combination hasn't been boot-tested. `7004`–`7009` are byte-identical
to the V3 branch, and all seven patches have the same `git patch-id` as
the tested commits. They're unsigned review drafts, not a signed-off
submission. Claude and Codex helped with the code, testing and analysis.
