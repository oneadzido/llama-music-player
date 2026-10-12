# Llama Music Player v1.3.0

**Release date:** Oct 12, 2026

## Overview

Llama Music Player is now a multilingual app. Sixteen languages ship
in this release, and every user-visible string the app shows is now a
resource rather than a hardcoded English literal. Choosing a language
from the new picker recreates the UI on the new locale in the first
frame after the dialog dismisses; no restart of the app is needed.

The localisation work touches every activity because every activity
resolves its strings through `getString`, every layout now references
`@string` rather than carrying English text, and every activity wraps
its base context through `LanguageManager.wrap` in
`attachBaseContext`. The set of supported languages is declared three
times — in `LanguageManager.LANGUAGE_TAGS`, in
`res/xml/locales_config.xml`, and in the `resourceConfigurations`
block of the app module's build file — and the three lists describe
the same set.

Three user-visible corrections ride alongside the localisation. The
top bars in every activity and every dialog lay their title out with
a weight, so a long translation truncates rather than overlapping a
neighbouring button. The sort-mode header carries the space between
the prefix and the mode label; the space is supplied by a constant on
the activity rather than a trailing space on a string resource, which
the resource compiler strips. And a language change made while the
Credits dialog is on top reaches the player the moment the dialog
closes rather than on the next lifecycle transition.

A fourth correction belongs to the same release: the album-art cache
is now process-scoped. The bitmaps are a fact about the songs, not
about any activity. An activity-scoped cache is discarded on every
recreate, and the carousel's next read then misses and schedules a
fresh decode. `AlbumArtCache` is a singleton that outlives every
activity, and both consumers — the carousel and the notification
controller — hold a reference to its underlying `LruCache` rather
than owning a cache of their own.

## Added

- **`LanguageDialog`** — a scrollable picker with the same shape as
  the Preset and Theme dialogs. Each row shows a language's endonym,
  the language's name in its own script, so a speaker recognises
  their language without first switching the app to it. The current
  language's row carries a CURRENT indicator in the accent colour and
  is scrolled into the vertical centre of the list when the dialog
  opens.
- **Language button in Credits** — the Credits dialog's version header
  now carries the VERSION label at the leading edge, the language
  button in the middle, and the version number at the trailing edge.
  The button matches the playlist's SORT button in height, text size,
  letter spacing, and border selector. Its label is the endonym of the
  currently applied language and updates the moment a new language is
  chosen.
- **Sixteen languages** — English (default), Español, Français,
  Deutsch, Italiano, Português, Русский, 中文, 日本語, 한국어, हिन्दी,
  العربية, Türkçe, Bahasa Indonesia, Nederlands, Polski.
- **`AlbumArtCache`** — a process-scoped singleton that owns the
  decoded album art bitmaps, keyed by song path. Both
  `MusicPlayer` and `NotificationController` hold a reference to its
  underlying `LruCache`, so a recreate does not evict the bitmaps and
  the carousel's first read after the recreate finds the entry the
  previous instance decoded. Two consumers read and write it, and both
  see the same bitmaps for the same path.

## Changed

- **Every user-visible string is now a resource.** The app previously
  held most of its user-visible text as hardcoded English literals
  in Java source and in layout files. The entire set has been
  extracted to `res/values/strings.xml` and translated into the
  fifteen supported languages. Every string the app shows is now
  read through `getString(R.string.…)` or referenced as
  `@string/…` in a layout.
- **Locale application through wrapped base context.** Each activity
  overrides `attachBaseContext(Context)` and wraps the base context
  with the selected locale before any activity state exists. A
  locale change requires the activity to be recreated, which the
  app does via a `LANGUAGE_CHANGED` broadcast that every activity
  listens for and answers with `recreate()`.
- **Sort labels in the playlist header.** The playlist's sort header
  now reads from `playlist_sort_label_*` resources, so a translator
  can localise the direction markers without changing the code.
- **The sort-mode header carries its space.** The separator between
  the sort-mode prefix and the mode label is a Java-side constant on
  `PlaylistActivity`, not a trailing space on the prefix resource,
  because the Android resource compiler strips trailing whitespace
  from a non-formatted string. The header now reads `SORT MODE
  [TITLE A-Z]` in every locale; no locale file carries a trailing
  space.
- **Layout attributes are relative.** Every padding, margin, and
  alignment in the seven layout files uses the Start/End variants,
  so the layouts mirror correctly under RTL locales.
- **Equalizer band labels follow the app's language.** The label
  above each equalizer band slider — the one that names the band's
  center frequency — is produced by
  `EqualizerManager.getFrequencyLabel(int)` and reads its unit symbol
  from `equalizer_frequency_*` resources. Russian supplies the
  designations GOST specifies, `Гц` and `кГц`; every other locale
  supplies the SI symbols `Hz` and `kHz`. The decimal separator
  between a kilohertz value and its tenths is not in the resource —
  it comes from the JVM default locale, which `LanguageManager.wrap`
  has already set to the app's selected language.
- **Dialog top bars no longer overlap the title.** Every dialog's top
  bar is a horizontal linear layout with the title weighted to the
  space the two edge controls leave. The title is the only element
  that can yield, so a long Cancel label — Dutch `ANNULEREN`, German
  `ABBRECHEN` — and a long title — Dutch `VOORINSTELLINGEN`, German
  `VOREINSTELLUNGEN` — cannot overlap. The language button in the
  Credits dialog's version header follows the same rule, so a long
  endonym truncates rather than reaching the version number.
- **Activity top bars no longer overlap the title.** The top bar in
  `PlaylistActivity`, `FolderManager`, `EqualizerActivity`, and
  `MetadataEditor` uses the same shape. `MusicPlayer`'s top bar was
  already a horizontal linear layout with three weighted buttons.
- **A deferred language change reaches the player when the Credits
  dialog closes.** When a language change arrives while a dialog
  holds window focus against `MusicPlayer`, the recreate is
  deferred. The deferral is consumed the moment the activity's
  window regains focus, in `onWindowFocusChanged(boolean)`, so the
  player picks up the new language on the same turn the dialog
  closes. A fallback in `onResume()` covers lifecycle-driven cases
  where the activity was paused while the flag was set.
- **`Song`'s getters resolve placeholders through
  `CharacterMapper`'s predicates.** A raw field that is null, or that
  holds a placeholder written under a previous locale, is replaced
  with the current language's value on the getter's own call. A
  language change is reflected on every song the next time its
  metadata is read; the song is not reconstructed.
- **`PlaylistAdapter`'s subtitle assembly uses the placeholder
  predicates.** The subtitle joins artist, album, and genre with a
  bullet, and each field is checked against the corresponding
  `CharacterMapper.isUnknownX(String)` predicate before it is
  rendered. A field that carries a placeholder from any supported
  locale renders the current locale's placeholder in its place.
- **`StorageAccessHelper.getStorageType(String)` resolves its
  labels from resources.** `[Internal]` and `[SD Card]` now read
  through `getLocaleContext()`, so a German folder list shows
  `[Intern]` and an Arabic one shows `[داخلي]`. The labels carry no
  leading or trailing whitespace; the separator belongs to the
  caller.
- **`ThemeManager.loadColorNames()` reads through
  `LanguageManager.getLocalizedContext()`.** The theme names resolve
  against the app's selected language rather than the system locale,
  so a device whose system language differs from the app's selection
  presents the theme list in the app's language.
- **`StorageObserver`'s overlay message resolves from resources.**
  `library_syncing_playlist` is read at broadcast time through the
  app's selected locale.
- **`FolderAdapter`'s Loose Tracks label is resolved by the owning
  activity.** The adapter renders the plural string the activity
  obtained from `R.plurals.folder_manager_loose_tracks`; it does not
  assemble the label from parts.
- **`OverlayManager` gained `REASON_PLAYLIST_SORT`.** A fourth
  process-wide reason token for the playlist's sort cycle, alongside
  `REASON_COLD_START`, `REASON_METADATA_LOAD`, and
  `REASON_FOLDER_UPDATE_CYCLE`. A sort on a large library is fast;
  without a floor, the overlay would flash and vanish. The reason is
  released on every exit path, including activity destruction.
- **`AlbumArtCarousel` consumes the shared `LruCache` through
  `AlbumArtCache`.** The carousel no longer owns a bitmap as
  primary state; its cache is the process-scoped singleton the
  activity hands it.
- **`MusicPlayer` and `NotificationController` hold the shared
  cache.** Both classes call `AlbumArtCache.getInstance(...).asLruCache()`
  in their setup, so a recreate no longer discards the decoded
  bitmaps.
- **`BluetoothReceiver` no longer owns the pause on disconnect.**
  The service owns the pause; the receiver broadcasts a
  `BLUETOOTH_PAUSE` request that the service consumes. See the
  1.2.8 entry in `CHANGELOG.md` for the full reasoning, which this
  release inherits.

## Under the Hood

- **New classes** — `LanguageManager` owns the language set, the
  current tag, and the `wrap(Context)` helper; `LanguageDialog` is
  the picker itself, structured line for line like `PresetDialog`.
  `AlbumArtCache` is a new process-scoped singleton that owns the
  decoded album art bitmaps. `MusicPlayer` and
  `NotificationController` hold a reference to the singleton's
  underlying `LruCache` rather than owning a cache of their own, so a
  recreate does not evict the bitmaps and the carousel's first read
  after the recreate finds the entry the previous instance decoded.
- **Modified classes** — `CreditsDialog` gained a three-child version
  header, a `createLanguageButton`, a `showLanguageDialog`, and a
  `LANGUAGE_CHANGED` broadcast on selection. `MusicPlayer`,
  `PlaylistActivity`, `FolderManager`, `EqualizerActivity`, and
  `MetadataEditor` each gained an `attachBaseContext` override, a
  `LANGUAGE_CHANGED` receiver that answers with `recreate()`, and
  the corresponding registration and unregistration.
  `NotificationController`'s channel names, action labels, and
  placeholders now read from resources. `PlaylistManager`'s overlay
  messages now read from resources. `CharacterMapper`'s four
  placeholder producers now resolve from string resources, and its
  four predicates match against both the current locale's value and
  the English base. `Song`'s four metadata getters now delegate the
  placeholder decision to those predicates. `PlaylistAdapter`'s
  subtitle assembly uses the same predicates. `StorageAccessHelper`'s
  storage-type labels and `ThemeManager`'s theme names and
  `StorageObserver`'s overlay message and `FolderAdapter`'s loose-track
  label all read from resources in the app's selected language.
  `OverlayManager` gained `REASON_PLAYLIST_SORT`. `AlbumArtCarousel`
  consumes the shared `LruCache` through `AlbumArtCache`.
- **`EqualizerManager.getFrequencyLabel(int)`** — the label above
  each equalizer band slider resolves its unit symbol from
  `equalizer_frequency_hz`, `equalizer_frequency_khz`, and
  `equalizer_frequency_khz_decimal`. The method's signature and its
  place in the band-control layout are unchanged; only the strings it
  resolves from are resource-backed.
- **`PresetDialog`, `ThemeDialog`, `LanguageDialog`, `CreditsDialog`** —
  `createTopBar` returns `LinearLayout`; the title takes weight 1 and
  truncates with an ellipsis; the Cancel and trailing controls take
  their natural widths. `CreditsDialog`'s `createVersionHeader`
  follows the same shape, with the language button weighted 1.
- **`activity_playlist`, `activity_folder_manager`,
  `activity_equalizer`, `activity_metadata_editor`** — the top bar in
  each layout is a horizontal linear layout with the title weighted
  to the space the two edge buttons leave.
- **`PlaylistActivity`** — `SORT_MODE_LABEL_SEPARATOR` is the
  constant that separates the sort-mode prefix from the mode label,
  replacing the two implicit space-bearing concatenations in
  `updateSortHeaderDisplay` and `updateTitleAndCount`.
- **`MusicPlayer`** — `consumePendingLanguageRecreate()` replaces the
  inline check in `onResume()` and is also called from
  `onWindowFocusChanged(boolean)` when the window regains focus.
- **New resources** — `res/values/strings.xml` (rewritten with the
  full key set, including the three `equalizer_frequency_*` keys the
  band labels read); `res/values-es`, `-fr`, `-de`, `-it`, `-pt`,
  `-ru`, `-zh`, `-ja`, `-ko`, `-hi`, `-ar`, `-tr`, `-in`, `-nl`,
  `-pl` (new directories, each with a `strings.xml`);
  `res/xml/locales_config.xml` (new, points at the platform's per-app
  language picker).
- **`AndroidManifest.xml`** — `android:supportsRtl="true"` and
  `android:localeConfig="@xml/locales_config"` added to the
  `<application>` element.
- **`app/build.gradle`** — `resourceConfigurations` bounds the
  release artifact to the sixteen locales the picker offers, so
  transitive AndroidX dependencies do not contribute their own
  translations of library-internal strings.
- **Note on translation quality** — the Latin-script and
  Cyrillic-script locale files were produced with high confidence.
  The Chinese, Japanese, Korean, Hindi, Arabic, Turkish, and
  Indonesian files would benefit from a native speaker's review.

---

## System Requirements

- **Minimum SDK:** Android 6.0 (API 23)
- **Target SDK:** Android 15 (API 35)
- **Recommended RAM:** 1 GB or higher

---

## Download

Download the APK from the Assets section of this release.

---

## Installation

1. Download the APK file
2. Enable "Unknown Sources" in your device settings
3. Open the APK file and tap "Install"

---

## First Time Setup

1. Tap the MANAGER button on the main screen
2. Tap IMPORT FOLDER to select your music folder
3. Tap UPDATE to save your selection
4. The app will scan your folder and build the playlist

---

## How to Use

| Feature | How to Access |
|---------|---------------|
| Equalizer | Tap EQUALIZER button on main screen |
| Metadata Editor | Tap METADATA button on main screen |
| Folder Manager | Tap MANAGER button on main screen |
| Playlist | Tap PLAYLIST button on main screen |
| Themes | Tap LLAMA button on main screen, then THEMES |
| Language | Tap LLAMA button on main screen, then the language button in the version header |
| Volume gesture | Swipe up or down along the album-art edge |
| Previous / next song | Swipe left or right across the album art, or tap the transport buttons |
| Playlist fast-scroll | Drag vertically along the left or right edge of the playlist |
| Sort mode | Tap SORT in the playlist header |

---

## Credits

- jaudiotagger by jthink Ltd
- FontAwesome by Fonticons, Inc.

---

## License

Licensed under the MIT License

Copyright (c) 2026 Richard Korbla Adzido

Made with love in Ghana