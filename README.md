Llama Music Player

Llama Music Player is a professional, open-source Android music player with advanced audio processing capabilities. It features independent pitch and tempo control, a 60-preset 10-band equalizer, comprehensive ID3 tag editing, folder management, and theme customization.

Features

Core Playback

· Audio Formats: MP3, FLAC, OGG, M4A, AAC, WAV, and more (all formats supported by Android MediaCodec)
· Background Playback: Continuous playback with notification controls
· Bluetooth Support: Headset connection detection and media button handling
· Audio Focus Management: Proper ducking and interruption handling
· Wake Lock: Prevents device sleep during playback

Pitch and Tempo Control

· Pitch Shifting: -6 to +6 semitones - Change the key without affecting speed
· Tempo Control: 0.5x to 1.5x - Change playback speed without affecting pitch
· Real-time Processing: Effects apply immediately during playback

Equalizer

· 10-Band Equalizer: Full control over frequency response
· 60 Presets: Genre-specific presets (Rock, Jazz, Classical, Hip Hop, etc.)
· Custom Presets: Save and restore your own EQ settings
· Bass Boost: 0-100% adjustable bass enhancement
· Surround Effect: 0-100% virtual surround sound

Metadata Editing

· Title, Artist, Album: Edit standard ID3 tags
· Genre: Set or change music genre
· Year, Track Number: Additional metadata fields
· Lyrics: Add or edit song lyrics
· Album Art: Add, change, or remove album artwork
· Format Support: MP3, FLAC, OGG, M4A, WMA

Playlist Management

· Folder Import: Import entire music folders via Storage Access Framework (SAF)
· Loose Tracks: Support for individual files opened from other apps
· Search: Live search across title, artist, album, and genre (128 char max)
· Sort Modes: 10 sort modes including Title, Artist, Album, Genre, Date
· Duplicate Prevention: Automatic detection and removal of duplicate entries

User Interface

· Theme Colors: 30 color options across multiple families
· Full-screen Immersive: Edge-to-edge display with system UI hiding
· Touch Feedback: Visual animations for all interactive elements
· Responsive Layout: Portrait orientation optimized

Notifications

· Media Controls: Play, pause, next, previous, close
· Album Art Display: Shows current track artwork
· Sync Status: Indicates when playlist is updating
· Foreground Service: Ensures playback continues in background

Technical Architecture

Audio Processing Pipeline

Audio File -> MediaCodec (Decode) -> SoundTouch (Pitch/Tempo) -> AudioTrack (Playback)

Player Architecture

The player uses SoundTouchPlayer as the default player with full pitch and tempo control. If SoundTouch fails at runtime, it automatically falls back to the standard MediaPlayer. The transition is seamless with no user-visible interruption.

Thread Safety

The application uses AtomicBoolean for state flags, synchronized blocks for critical sections, Handler postings to the main thread for UI updates, and ExecutorService for background database operations.

Download and Installation

Direct APK Download

1. Go to the Actions tab on GitHub
2. Select the latest workflow run
3. Download llama-music-player-apk from Artifacts
4. Install on your Android device (enable "Unknown Sources")

Requirements

· Android Version: 4.4 (KitKat) / API 19 or higher
· RAM: 512 MB minimum
· Storage: 50 MB
· Permissions: Storage, Bluetooth, Notifications

Building from Source

Prerequisites

· Android Studio Hedgehog or later
· JDK 17
· Android SDK 34
· NDK 25.1.8937393
· CMake 3.22.1

Local Build

Clone the repository and navigate to the project directory. Run ./gradlew assembleRelease to build the APK. The output will be located at app/build/outputs/apk/release/app-release.apk.

Cloud Build (GitHub Actions)

1. Fork or clone this repository
2. Create a ZIP file of your project named llama-music-player.zip
3. Upload to your repository root
4. Go to the Actions tab
5. Select Build Llama Music Player APK
6. Click Run workflow
7. Download the APK from Artifacts

Required Local Assets

Before building, ensure your llama-music-player.zip contains:

· app/src/main/assets/fonts/fa-solid-900.ttf (FontAwesome font - REQUIRED)
· app/src/main/java/com/ark/llama/ (all Java source files)
· app/src/main/res/ (all resources)
· app/src/main/cpp/CMakeLists.txt
· app/src/main/cpp/pitch_shifter.cpp
· app/src/main/cpp/soundtouch/ (SoundTouch source files)
· app/src/main/AndroidManifest.xml
· app/libs/jaudiotagger-3.0.1.jar (ID3 tag library)
· app/build.gradle
· build.gradle (top-level)
· settings.gradle
· gradle.properties
· gradle/wrapper/gradle-wrapper.jar
· gradle/wrapper/gradle-wrapper.properties

Build Configuration

· minSdkVersion: 19
· targetSdkVersion: 34
· compileSdkVersion: 34
· Java Version: 1.8
· NDK Version: 25.1.8937393
· CMake Version: 3.22.1

Project Structure

The project follows the standard Android app structure with the main code in app/src/main/java/com/ark/llama/. The native C++ code for pitch shifting is in app/src/main/cpp/ with the SoundTouch library in the soundtouch subdirectory. Resources including layouts, drawables, and animations are in app/src/main/res/. The GitHub Actions workflow for building the APK is located at .github/workflows/build-apk.yml.

Java Classes (32 classes)

· AudioProcessor.java - Wrapper for SoundTouch audio processing
· BluetoothReceiver.java - Handles Bluetooth headset events
· CharacterMapper.java - Cleans and maps music metadata
· CreditsDialog.java - Displays app credits and license information
· EqualizerActivity.java - Audio effects control interface
· EqualizerManager.java - Manages equalizer and audio effects
· FolderAdapter.java - Adapter for folder list in RecyclerView
· FolderManager.java - Manages music folders
· FontAwesome.java - Utility for FontAwesome icons
· MetadataEditor.java - ID3 tag editor activity
· MetadataReader.java - Reads audio file metadata
· MetadataWriter.java - Writes metadata with backup/restore
· MusicLibrary.java - Manages music library and scanning
· MusicPlayer.java - Main playback activity
· MusicService.java - Background playback service
· NavigationController.java - Centralized button state management
· NotificationService.java - Manages media notifications
· PitchShifter.java - JNI wrapper for SoundTouch
· PlaylistActivity.java - Playlist display and management
· PlaylistAdapter.java - Adapter for playlist RecyclerView
· PlaylistService.java - Background playlist synchronization
· PresetDialog.java - Equalizer preset selection dialog
· Song.java - Song data model
· SongDatabase.java - SQLite database for song caching
· SoundTouchPlayer.java - High-quality audio player with pitch/tempo
· StorageAccessHelper.java - SAF operations helper
· StorageObserver.java - Monitors storage changes
· ThemeDialog.java - Theme selection dialog
· ThemeHelper.java - Applies theme colors to UI components
· ThemeManager.java - Manages theme color persistence
· ToastManager.java - Custom toast notifications

Third-Party Libraries

· SoundTouch 2.3.3 - LGPL 2.1 - Pitch and tempo shifting
· jaudiotagger 3.0.1 - LGPL/MPL - ID3 tag reading and writing
· FontAwesome 6.0 - SIL OFL 1.1 - Vector icons
· AndroidX - Apache 2.0 - Android support libraries

License

MIT License

Copyright (c) 2026 Richard Korbla Adzido

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

Author

Richard Korbla Adzido

· Email: oneadzido@gmail.com
· GitHub: oneadzido

Acknowledgments

· SoundTouch library by Olli Parviainen
· jaudiotagger by jthink Ltd
· FontAwesome by Fonticons, Inc.
· Android Open Source Project

Version History

Version 1.0.0 (2026) - Initial release

Support

For issues, feature requests, or questions, please check the Issues page on GitHub and create a new issue with detailed description including device model, Android version, and steps to reproduce.

Contributing

Contributions are welcome. Please fork the repository, create a feature branch, make your changes, and submit a pull request.

Made with love in Ghana.
