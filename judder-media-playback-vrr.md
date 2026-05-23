# Frame judder with video playback

[← 2026](2026)

## Overview

- VRR displays are great for games, where frame production is inherently
irregular, but fixed-cadence video can still judder if the display pipeline
doesn't pace frames correctly.
- For example, on a 40-60 Hz VRR panel I noticed unmistakable judder when
playing 30 fps content in mpv on Wayland. 50 fps content looked fine.

### Media playback judder
![](/images/judder_with_media_playback_vrr.png)

- On a 40-60 Hz VRR panel, refresh rate can vary only between 16.67 ms and 25 ms.
- But 30 fps video delivers a new frame every 33.33 ms, which falls below the
panel’s minimum VRR rate.
- Because of this, frames cannot be displayed at a uniform cadence and instead
alternate between longer and shorter display times, about 41.67 (25+16.67) ms and 25 ms.
- The average frame rate remains correct, but the uneven frame pacing appears
as visible judder.

## What can userspace do to fix this?
- The client or media player passes the content frame rate using a new
[content-frame-rate-v1](https://gitlab.freedesktop.org/NaveenKumar/wayland-protocols/-/commits/test-content-frame-rate?ref_type=heads) protocol to the compositor.

- The compositor should do the low framerate compensation (LFC) to choose the
smallest stable refresh multiple that fits within the VRR range.

- For 30 fps content on a 40 to 60 Hz panel, that target becomes 60 Hz, so each
video frame is shown for exactly two scanouts.

- This restores a stable presentation cadence and removes judder.

- The compositors can implement this by deriving a virtual display mode from the
current KMS mode and [retiming vtotal, vsync_start, vsync_end, and vrefresh](https://gitlab.gnome.org/naveenk2/mutter/-/commit/4598557c5d2ec60b42ed3ffb4b6eb1e87357212b#line_229db3239_A169) to
the computed target refresh rate so that the panel scans out at exactly
target_hz.

### Media playback judder fixed
![](/images/fix_judder_with_media_playback_vrr.png)

## More details
To fix this, I carried out a PoC, you can check the full implementation details
here: [Fixing judder issue with video playback](https://k2naveen.github.io/wayland/vrr/2026/05/23/fixing-judder-issue-with-video-playback.html)
