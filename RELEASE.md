# Llama Music Player v1.4.6

**Release date:** November 12, 2026

Version 1.4.6 improves interface refresh reliability on OEM Android
firmware and prevents volume-key events intended for another app from
resuming Llama. For the full change history, see the
`[1.4.6]` entry in the [CHANGELOG](CHANGELOG.md). For technical
specifications, installation, and usage, see the
[README](README.md).

---

## What's in this release

### Theme and control changes repaint immediately

Theme application, configuration changes, and transport-control
refreshes now explicitly request layout and drawing. This addresses
firmware that defers a view's repaint after text, color, or enabled-state
changes. Toast overlays also invalidate their own surfaces after a theme
color update. The fix is unconditional and does not depend on the device
manufacturer.

### Language changes refresh the resource configuration

Language-change receivers now use a shared activity recreation helper.
It refreshes the current resource configuration before recreating the
activity and suppresses the transition animation, helping the first
frame after a language change use the selected locale.

### Volume-key resume respects audio focus

Both volume-resume paths now consult the same service-owned audio-focus
state. Raising the music volume while another app owns focus no longer
starts Llama over that app's audio. The volume-zero pause behaviour is
preserved, and an active interruption record remains eligible for
recovery.