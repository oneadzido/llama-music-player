# Privacy Policy for Llama Music Player

**Last Updated:** November 7, 2026

---

## Overview

Llama Music Player does not collect, transmit, or store any personal
information. This document describes how the application handles your
data.

---

## No Data Collection

Llama Music Player does **not** collect, transmit, or store any
personal information.

Llama does not request your name, email address, or any other
personally identifiable information. It does not monitor your
location, gather device identifiers, access your contacts, messages,
or call history, or integrate with any advertising or analytics
network.

---

## On-Device Storage and Backup

Llama Music Player does not operate an account service or send your
library information to its developer or to an app-operated remote
service. The application stores its working data on your device.

Android system backup is enabled in the application manifest. If
backup is enabled in your device or account settings, Android may
include application data in the backup destination configured for
your device. That transfer is managed by Android's backup service,
not by Llama Music Player, and follows the backup settings you choose.

The app stores the following information locally:

- **Song metadata and favorites** (title, artist, album, genre, and
  favorite state) — stored in the local SQLite database
- **Album artwork** — cached in memory for efficient display; embedded
  artwork edits are written to the selected audio file
- **Equalizer preferences** — stored in SharedPreferences
- **Playback state** (current track, position, repeat and shuffle
  settings) — stored in SharedPreferences
- **Playback scope** — the selected mode and filter values are stored
  in SharedPreferences
- **Selected music folders** — stored in SharedPreferences
- **Theme and language preferences** — stored in SharedPreferences

The sixteen languages the app ships are resource folders bundled in
the APK. No translation service, API key, or network call is
involved. Selecting a language changes the display strings on your
device; it does not tell anyone anything about you.

---

## Permissions

Llama Music Player declares only the permissions used by its music
library, playback, and audio features. Android presents runtime
permission requests when the related feature needs access:

- **Music library access** — `READ_MEDIA_AUDIO` on Android 13 and
  later, or `READ_EXTERNAL_STORAGE` on earlier supported versions.
- **Notifications** — `POST_NOTIFICATIONS` on Android 13 and later,
  so playback and library notifications can appear in the shade.
- **Bluetooth** — `BLUETOOTH_CONNECT` on Android 12 and later for
  headset connection state and media-button routing.
- **Album artwork** — image access is requested by the Metadata Editor
  when the user selects artwork, not as part of the initial library
  permission flow.
- **Audio and playback** — `MODIFY_AUDIO_SETTINGS` and `WAKE_LOCK`
  support audio effects and sustained playback. The foreground-service
  permissions are declared in the manifest; Android grants the
  media-playback service-type permission at install time rather than
  presenting it as a runtime prompt.
- **Battery optimization** — when the user chooses uninterrupted
  playback support, the app opens Android's system settings for the
  battery-optimization exemption. The choice is managed by Android.

Each permission is used for the feature described above. The app does
not use these grants to collect or transmit personal information.

---

## Third-Party Services

Llama Music Player integrates **no third-party analytics, advertising,
or tracking services**.

The app uses two open-source libraries:

- **jaudiotagger** — for reading and writing ID3 metadata tags
- **FontAwesome** — for vector icon rendering

Both libraries operate entirely on your device and do not transmit
any data.

---

## Security

The app has no account system, analytics endpoint, or advertising
network. Its database and preferences are stored in application
storage. Android's optional system backup may copy app data according
to the device's backup settings; the app does not control that
service. Your selected folders, music files, and playback preferences
remain managed through Android and the device's storage permissions.

---

## Children's Privacy

Llama Music Player does not collect any personal information from any
user, including children under the age of 13. The app contains no
social features, user accounts, or mechanisms for data collection.

---

## Policy Updates

This privacy policy may be revised from time to time. Any updates are
reflected by the "Last Updated" date at the top of this document.

---

## Contact

If you have any questions about this privacy policy, please reach out
to the developer:

**Richard Korbla Adzido**
oneadzido@gmail.com

---

**Llama Music Player respects your privacy.**