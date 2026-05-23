---
layout: post
title: "Fixing frame judder with video playback"
date: 2026-05-23 10:30:00 +0530
categories: [wayland, vrr]
tags: [wayland, vrr, mutter, mpv, kms, content-frame-rate]
---

Variable refresh rate displays are great for games, where frame production is
inherently irregular: the panel scans out the moment a frame is ready, so there
is no vsync stutter and no input lag from holding a frame back to the next
refresh boundary. For video playback the situation is different. A 24 fps
movie produces frames at a perfectly fixed cadence; the question is just what
the panel does in between them.

On a 40-120 Hz VRR panel I noticed unmistakable judder when playing 24 fps
content in mpv on Wayland. 50 fps content looked fine. This post walks through
how the problem manifests in the KMS layer, what the missing piece is, and how
a small Wayland protocol plus a low-frame-cadence path in mutter fix
it.

## What judder looks like in the KMS log

mutter has a debug topic that logs the last four observed present intervals
per CRTC. With `MUTTER_DEBUG_TOPICS=kms` enabled, playing the 24 fps clip on
the unpatched stack produces lines like:

```
KMS_DEADLINE: CRTC 149 VRR present intervals { 8333, 16667, 8333, 16667 }  → min 8333 µs
KMS_DEADLINE: CRTC 149 VRR present intervals { 16667, 8333, 16667, 16667 } → min 8333 µs
KMS_DEADLINE: CRTC 149 VRR present intervals { 33332, 16669, 16663, 16670 } → min 16663 µs
KMS_DEADLINE: CRTC 149 VRR present intervals { 16664, 33333, 16671, 16663 } → min 16663 µs
KMS_DEADLINE: CRTC 149 VRR present intervals { 25002, 24999, 25001, 16664 } → min 16664 µs
```

Those are scanout intervals in microseconds. 8333 µs is 120 Hz, 16667 µs is
60 Hz, 25000 µs is 40 Hz, 33333 µs is 30 Hz. For 23.976 fps content the
nominal frame interval is **41 708 µs**, but the actual presents wander all
over the 40–120 Hz VRR window.

That wander is exactly what the eye picks up as judder. Each video frame is
held for an arbitrary number of scanout periods until the next frame is ready,
and the period itself is not stable.

## Why the compositor can't fix this on its own

VRR works in the direction of *latency*: as soon as a frame arrives, scan it
out. The panel firmware also enforces a minimum refresh rate (40 Hz here) by
re-sending the last line when nothing has arrived in a while. So at 24 fps,
between two real frames there is *always* at least one self-refresh from the
panel — and the exact moment it happens depends on when the compositor's frame
clock decides to wake up. The compositor has no idea the client intends to
post frames at a fixed 23.976 Hz cadence, so it cannot align anything.

The fix is to tell the compositor what cadence the client is producing.

## The `content-frame-rate-v1` protocol

A new Wayland protocol,
[`wp_content_frame_rate_v1`](https://gitlab.freedesktop.org/NaveenKumar/wayland-protocols/-/commit/f3b2ed2c49a9bdb4419b70baab1f0e0af097d259), lets a client
attach a rational frame rate (numerator / denominator) to a surface. The
compositor is free to use that hint to schedule its frame clock and, more
importantly, to retune the panel.

Three components needed changes:

1. **wayland-protocols** – the XML for the protocol.
2. **mpv** – bind the manager, create a per-surface object, and call
   `set_frame_rate` whenever the video track's container FPS changes.
3. **mutter** – receive the hint, plumb it from the Wayland surface state into
   `MetaOnscreenNative`, and compute a *virtual* refresh mode for the CRTC.

The mutter side is where the interesting logic lives.

## Low Frame Cadence in mutter

When VRR is on and the client supplies a content frame rate, mutter computes:

```
content_fps = numerator / denominator
vrr_min_hz  = 1_000_000 / max_refresh_interval_us
N           = max(ceil(vrr_min_hz / content_fps), 1)
target_hz   = content_fps * N
```

`N` is the smallest integer multiplier that brings the content rate up to (or
above) the panel's VRR floor. If the resulting `target_hz` lies within
`[vrr_min, vrr_max]`, mutter clones the current KMS mode and rewrites
`vtotal`, `vsync_start`, `vsync_end`, and `vrefresh` so that the panel scans
out at exactly `target_hz`. From that point on every video frame is displayed
for `N` scanout periods of fixed length.

For the two cases tested:

| content fps | VRR window | N | target | result |
|-------------|------------|---|--------|--------|
| 49.984      | 40–120 Hz  | 1 | 49.984 Hz | content rate matches scanout 1:1 |
| 23.976      | 40–120 Hz  | 2 | 47.952 Hz | each frame shown for two scanouts |

The `N=2` row is the actual fix for the judder I started with.

## Tracing the patched stack

To verify each link of the chain I added `MP_VERBOSE` logs in mpv and
`meta_topic(META_DEBUG_KMS, ...)` / `META_DEBUG_WAYLAND` logs in mutter. Then
I played the same 23.976 fps clip:

mpv (`-v` plus `WAYLAND_DEBUG=1`):

```
-> wl_registry#2.bind(38, "wp_content_frame_rate_manager_v1", 1, new id #29)
-> wp_content_frame_rate_manager_v1#29.get_surface_content_frame_rate(..., wl_surface#5)
-> wp_content_frame_rate_v1#39.set_frame_rate(24000, 1001)
```

mutter (`MUTTER_DEBUG_TOPICS=wayland,kms`, captured from the journal):

```
content-frame-rate: surface=5 set_frame_rate 24000/1001 (23.976 fps)
KMS: LFC virtual refresh target 47.952 Hz (content 24000/1001, N=2)
KMS: Setting CRTC (149) virtual refresh mode to 47.952 Hz
KMS: [atomic] Setting CRTC 149 (/dev/dri/card1) property 'VRR_ENABLED' to 1
```

And, the payoff, the KMS deadline topic after the mode switch settles:

```
KMS_DEADLINE: CRTC 149 VRR present intervals { 20829, 20834, 20836, 20838 }  → min 20829 µs
KMS_DEADLINE: CRTC 149 VRR present intervals { 20834, 20833, 20838, 20829 }  → min 20829 µs
KMS_DEADLINE: CRTC 149 VRR present intervals { 20833, 20838, 20829, 20834 }  → min 20829 µs
```

Every present interval is within ±5 µs of 20 833 µs, which is exactly half the
24 fps frame interval. The panel is now scanning out at a perfectly steady
47.952 Hz and the player is delivering one new frame every other scanout.

For the 49.984 fps clip the equivalent capture shows the panel locked at
~50 Hz (20 000 µs ±10 µs) for the entire playback window.

## Lifecycle

Closing mpv between clips emits the expected teardown:

```
KMS: [atomic] Setting CRTC 149 ... property 'VRR_ENABLED' to 0
```

Opening the next clip re-establishes the hint and re-enables VRR + virtual
mode with whatever new `N` is appropriate for the new content rate. No
manual configuration, no user-visible mode change.

## Visual result

Judder is gone for the 24 fps case. The 50 fps case, which was already
acceptable, now scans out at exactly the content rate rather than relying on
whatever cadence the frame clock happened to converge to. Cursor movement and
desktop animations remain VRR (they're driven by the regular frame clock,
which is what the `MODE_VARIABLE` clutter mode is for).

## Where the patches live

* [`wayland-protocols XML`](https://gitlab.freedesktop.org/NaveenKumar/wayland-protocols/-/commit/f3b2ed2c49a9bdb4419b70baab1f0e0af097d259) – `0001-add-content-frame-rate-protocol-v1.patch`
* [`mpv`](https://github.com/k2naveen/mpv/commits/test-content-refresh-rate/) – `video/out/wayland_common.c`, `player/misc.c`, `video/out/vo.h`
* [`mutter`](https://gitlab.gnome.org/naveenk2/mutter/-/commits/test-content-frame-rate) – `src/wayland/meta-wayland-content-frame-rate.c`, plumbing through
  `meta-wayland-surface.c`, and the LFC path in
  `src/backends/native/meta-onscreen-native.c` plus
  `src/backends/native/meta-crtc-kms.c` / `meta-kms-mode.c`

If you have a wide-range VRR panel and a low-fps clip handy, the same logging
above is enough to confirm the chain end to end on your own setup.
