---
layout: post
title: "Enabling async page flip (tearing) in GNOME mutter"
date: 2026-06-18 04:00:00 +0000
categories: [wayland, tearing]
tags: [wayland, tearing, mutter, gnome, kms, async-page-flip, drm]
---

Tearing support on Wayland sounds simple at first glance: if a client says it
prefers latency over perfect vblank alignment, let the compositor flip the new
buffer immediately with `DRM_MODE_PAGE_FLIP_ASYNC`.

In practice, getting that to work in GNOME mutter required changes across the
Wayland protocol layer, compositor policy, Clutter frame scheduling, KMS plane
capability parsing, and the atomic commit path.

This post walks through the mutter branch that adds tearing support,
`wip/add-tearing-support`, and explains why the final implementation is more
than just setting one atomic flag.

## End-to-end flow

![Async page flip tearing flow](/images/async-page-flip-tearing-flow.png)

## What async page flip changes

With normal presentation, the compositor schedules a frame against the next
refresh deadline and the KMS commit becomes visible on a vblank boundary. That
avoids tearing, but it also means an extra wait if a fullscreen client finishes
rendering just after the compositor has already targeted the current refresh.

Async page flip changes the tradeoff:

- submit the new primary plane framebuffer immediately
- let scanout switch mid-frame if needed
- reduce latency for fullscreen content that prefers it
- accept visible tearing as the tradeoff

That model only really makes sense when the content owns the whole output and
is not competing with other desktop composition work.

## The protocol piece: `wp_tearing_control_v1`

The first requirement is a way for Wayland clients to express preference.
The branch adds support for the `wp_tearing_control_manager_v1` protocol and a
per-surface `wp_tearing_control_v1` object in
`src/wayland/meta-wayland-tearing-control.c`.

The protocol is intentionally small. Clients set one of two presentation hints:

- `ASYNC` if they prefer tearing for lower latency
- `VSYNC` if they prefer normal synchronized presentation

On the mutter side this is stored in the pending Wayland surface state as an
`allow_tearing` bit, which then gets applied as part of normal surface state
processing.

That is an important design point: the protocol does not force tearing. It only
lets the client opt in.

## Mutter policy: only when it is actually safe

Once the protocol state exists, mutter still needs to decide whether that hint
should affect the output.

The policy in this branch is conservative:

- only a fullscreen surface can drive tearing
- the surface must be a Wayland surface
- the output must support atomic async page flip
- hardware cursor handling must not interfere

This logic lives in `src/compositor/meta-compositor-view-native.c` and
`src/backends/native/meta-onscreen-native.c`.

The compositor tracks the current fullscreen actor for each stage view. If the
topmost actor is a fullscreen Wayland surface, mutter checks whether that
surface has tearing enabled and then requests tearing on the onscreen object.
If not, it forces the output back to synchronized presentation.

That means the feature is output-scoped, but driven by the currently eligible
fullscreen surface.

## Async is not just a flag: frame clock changes matter

If the compositor keeps scheduling work as if presentation must align to
vblank, then enabling async page flip will not buy much latency reduction.

To address that, the branch adds a new Clutter frame clock mode,
`CLUTTER_FRAME_CLOCK_MODE_UNLOCKED`, and switches to it whenever tearing is
active.

The key idea is that unlocked mode stops treating the refresh cycle as the
primary scheduling constraint. Instead of pacing frames around the normal
vblank deadline model, the compositor can submit work as soon as it is ready.

In the branch, `meta_onscreen_native_before_redraw()` updates the frame clock
mode like this:

- tearing enabled -> `UNLOCKED`
- VRR enabled -> `VARIABLE`
- otherwise -> `FIXED`

That is one of the most important parts of the series. Without it, the kernel
might accept async flips, but the compositor would still behave like a mostly
vsynced system.

## KMS capability plumbing: async formats are separate

A second non-obvious part is scanout format support.

Some hardware exposes a normal `IN_FORMATS` blob for plane formats and
modifiers, but async flips may support only a subset of those combinations.
This branch adds support for `IN_FORMATS_ASYNC` on KMS planes and stores a
separate async format/modifier table per plane.

That work lives mainly in `src/backends/native/meta-kms-plane.c`.

Why this matters:

- a plane may be scanout-capable in general
- but not all formats or modifiers are valid for async flip
- compositor-side validation needs to know the async subset up front

Without that separation, mutter could choose a format that works for normal
page flips and only discover much later that async presentation is impossible.

## Output policy: cursor and direct scanout interactions

Async commits are heavily restricted in the kernel and drivers. In particular,
anything beyond the primary plane update can make an async commit invalid.

This branch handles that in a few ways:

- mutter inhibits the hardware cursor when tearing is enabled
- direct scanout and fullscreen candidate selection are tied into the tearing
  policy
- async commits are only attempted on paths that are close to pure primary
  plane flips

That cursor inhibition is especially important. A cursor plane update during a
tearing frame can turn what should be a simple async primary flip into a more
complex atomic transaction that the driver will reject.

## Making async robust: pre-test and fallback to sync

The last piece turned out to be essential in practice.

Even when a client is eligible for tearing, not every single frame is safe to
submit asynchronously. During transitions you may still have plane state
changes that are valid for normal commits but not for async ones.

Examples include:

- sync to async toggles
- direct scanout handoff transitions
- transient plane state changes that drivers reject for async

To make this robust, the branch adds an async pre-test in
`src/backends/native/meta-kms-impl-device-atomic.c`:

1. build the real atomic request
2. if `PAGE_FLIP_ASYNC` is requested, first issue a `TEST_ONLY` async commit
3. if the async test fails, clear `PAGE_FLIP_ASYNC`
4. submit the same frame synchronously instead of hard-failing

That fallback is what makes repeated sync <-> async switching stable in
practice. The compositor still prefers async when possible, but it does not get
stuck on transient `EINVAL` failures from the atomic path.

## User-facing enablement

The branch also adds an experimental mutter feature flag and settings plumbing
for tearing.

The new experimental feature key is `tearing` in
`org.gnome.mutter experimental-features`, and the branch also wires the state
through `MetaSettings` so the compositor can expose and honor it consistently.

At the KMS level, mutter separately checks whether the device advertises
`DRM_CAP_ATOMIC_ASYNC_PAGE_FLIP` before trying to use the feature.

So the effective gate is:

- feature enabled in mutter
- client opted in through tearing control
- output is fullscreen and eligible
- KMS device supports atomic async page flip

## Why this series ended up larger than expected

The interesting lesson from this work is that tearing support is not really one
feature. It is a chain of smaller requirements that all need to line up:

- a client protocol to express preference
- compositor policy for when that preference is honored
- a frame clock mode that does not artificially preserve vblank pacing
- async-specific plane format validation
- KMS commit handling that can fall back gracefully

Any one of those missing pieces produces a system that either does not tear
when requested, tears unreliably, or falls over when the frame path changes.

## Where the patches live

The mutter implementation is on this branch:

- [`naveenk2/mutter:wip/add-tearing-support`](https://gitlab.gnome.org/naveenk2/mutter/-/commits/wip/add-tearing-support)

The main commits in the series are:

- `build: Bump libdrm requirement to >= 2.4.120`
- `atomic: Check for atomic tearing capability`
- `settings: Add experimental feature for tearing`
- `wayland: Add support for tearing control protocol`
- `backends/native: Add IN_FORMATS_ASYNC and improve async (tearing) handling`
- `Add unlocked frame clock mode for tearing support`
- `kms/atomic: pre-test async and fallback to sync`

## Practical outcome

With this branch, mutter can finally honor a fullscreen Wayland client's
tearing hint in a way that is coherent end to end:

- the client opts in through Wayland
- mutter enables tearing only for the right surface and output state
- the frame clock stops artificially holding frames to vblank cadence
- the KMS backend uses async flips when the hardware state really supports it
- and if one frame is not async-safe, mutter falls back to sync instead of
  failing the presentation path

That combination is what makes tearing support usable rather than merely
present.
