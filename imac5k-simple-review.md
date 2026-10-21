# iMac 5K: Simple V2 and V3 for review

These are revisions 2 and 3 of the small Simple V1 series, offered for
review while hardware validation continues. They follow
[Atharva's request for a smaller patch set](https://github.com/t2linux/wiki/issues/743#issuecomment-6290452376).
V2 has five patches, adding 105 lines and removing 15. V3 adds a sixth
patch: a 22-line wait before detecting the right tile at boot. Both change
eight existing kernel files, with no new kernel files or recovery worker.

- [V2: five patches](https://github.com/edschofield/linux-t2-patches/tree/draft/imac5k-simple-v2)
- [V3: six patches](https://github.com/edschofield/linux-t2-patches/tree/draft/imac5k-simple-v3)
- [V2 to V3 comparison](https://github.com/edschofield/linux-t2-patches/compare/draft/imac5k-simple-v2...draft/imac5k-simple-v3)

The first five patch files and this note are identical on both branches.
The comparison contains only the added boot-wait patch, `7009`.

| Patch | Change |
| --- | --- |
| 7004 | Re-arm `0x4F1` after the left tile's D0 write. |
| 7005 | Enable the left tile before the right. |
| 7006 | Keep the right tile's sink across power-downs, restore its published EDID after a forced probe, and wait for readiness before training. |
| 7007 | Ignore the right tile's idle lane bits while still failing an unsuccessful AUX read. |
| 7008 | Restart an unchanged right tile when the left tile restarts. |
| 7009, V3 only | Wait for right-tile HPD and a successful DPCD read before boot detection, with a 150 ms timeout. |

V2 already includes the D0 re-arm and primary-first ordering. Both versions
keep Atharva's detection-time and atomic-tail `0x4F1` writes. Patch 7009
adds no register write.

The comparison was tested on one iMac20,1 / Radeon Pro 5300 with an LED
Cinema Display attached. In V2, DP-1's EDID arrived after fbcon had chosen
a framebuffer for eDP-1 alone. Later fbdev probes use the existing
framebuffer's dimensions, so they rejected the 2880-line tile modes.
The correct EDID eventually arrived, but the console stayed at 3840x2160
with DP-1 disabled throughout the two-minute passive observation.

The saved DRM-debug boot logs from 21 October 2026 show:

| Event, seconds after boot | V2 | V3 |
| --- | --- | --- |
| DP-1 boot detection | No sink at 5.461 s | Sink published at 5.748 s |
| First DP-1 EDID update | 9.811 s | 5.662 s |
| fbdev framebuffer allocation | 3840x2160 at 7.717 s | 5120x2880 at 8.000 s |
| Later probe | Both tiles' 2560x2880 modes rejected with `VIRTUAL_Y`; DP-1 left with 640x480 | Both tiles have 2560x2880 modes |

All 14 recorded V3 boots passed the native-tile state checks: nine used
the production image, including one with DRM debug logging, and five used
the separate diagnostic image. Each had both tiles at 2560x2880 on separate
CRTCs and DP-1's 256-byte EDID. The five diagnostic boots measured waits of
43,279 to 53,322 microseconds. The boot checks did not verify the visible
image. These results support acquiring DP-1 before the console's first
mode selection; they do not prove that every patch is individually necessary.

R3's unattended S3 and DPMS checks with the Cinema passed their state and
log gates. Attended visual checks, cold starts and tests without the Cinema
remain pending in this record. A reversed-CRTC seam run recorded three
one-frame tile offsets in 40 seconds. A forced readiness timeout also
showed training continuing under `dc_lock` for 2.29 seconds; failing the
enable immediately on timeout remains a possible follow-up. This series
is not yet a validated complete fix. It retains Atharva's iMac20,1/20,2
scope, but has no iMac20,2 test results.

The hardware-tested source uses Linux 7.3, Atharva's full 5K attachment and
the project's pinned T2 prerequisites. The exact source commits are:

| Version | Tested source commit | Kernel release |
| --- | --- | --- |
| Base | `000f24044eacaf78ea65c62f7ef73c79a33a09b8` | `7.3.0-imac-atfix7` |
| V2 | `8404bf6e8c29280d3248bad379f97d8b1c893f55` | `7.3.0-imac-hpd73-simple-v1` |
| V3 | `da76349f95da474043e3a993d6c620c81ef2b523` | `7.3.0-imac-hpd73-simple-v1-r3` |

These review branches start at t2linux patch-repository commit
`e6965b4babc6c73383143c989b555b151fcfbc46` (Linux 7.3). Applying its
patches and the review exports to v7.3 reproduces each tested
`drivers/gpu/drm/amd` subtree exactly. Other T2 drivers differ from the
hardware-tested base; this full combination has not been boot-tested.
The exported patches preserve the tested code and descriptions, with
`7004`-`7009` filenames and unnumbered mail subjects so the shared files
remain identical. They are unsigned review drafts, not a signed-off
submission. Claude and Codex assisted with the code and analysis.
