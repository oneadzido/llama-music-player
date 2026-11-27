# Llama Music Player v1.7.0

**Release date:** November 27, 2026

## Overview

Version 1.7.0 localizes the metadata-loading progress count using the active app locale. The existing loading message and ToastManager overlay composition remain unchanged, so the localized message continues to appear before the localized progress fraction, separated by the existing middle dot.

## Fixed

- The genre-loading progress fraction now formats both values with the active app locale, using the same `NumberFormat` pattern already used for localized time and metadata values.
- Digit grouping remains disabled, preserving the compact `current/total` fraction.
- The progress fraction remains the overlay body; `ToastManager` continues composing the localized title and body with its existing middle-dot separator.

## Preserved

- The existing `ToastManager` title/body composition and Android text-rendering path.
- Genre selection, metadata extraction, progress throttling, cancellation, heartbeat, and completion handling.
- The existing loading-message translations and custom-message override.

For the complete release history, see [CHANGELOG.md](CHANGELOG.md).