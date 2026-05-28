# Quick Start
Double-click `start_android_emulator.bat` in the root folder.

# Commands (Full Path)
If `emulator` or `adb` commands are not recognized (not in PATH), use full paths:

C:\Users\Dell\AppData\Local\Android\Sdk\platform-tools\adb.exe start-server
C:\Users\Dell\AppData\Local\Android\Sdk\emulator\emulator.exe -list-avds
C:\Users\Dell\AppData\Local\Android\Sdk\emulator\emulator.exe -avd Pixel_9a

# Start with Increased RAM (e.g. 4GB)
C:\Users\Dell\AppData\Local\Android\Sdk\emulator\emulator.exe -avd Pixel_9a -memory 4096

# Flutter Run
# If emulator is already running (use ID from 'flutter devices'):
flutter run -d emulator-5554

# To launch and run (use AVD name if not running):
flutter run -d Pixel_9a

# Troubleshooting & Reset
If the app fails to launch or "Log Reader" errors occur:


# Release apk build
flutter clean; flutter build apk --release --obfuscate --split-debug-info=build/app/outputs/symbols --tree-shake-icons

# App bundle build
flutter build appbundle --release --obfuscate --split-debug-info=build/app/outputs/symbols --tree-shake-icons

# Debug apk build
flutter clean; flutter build apk --debug --obfuscate --split-debug-info=build/app/outputs/symbols   --tree-shake-icons



### 1. Wipe Emulator Data (Factory Reset)
`C:\Users\Dell\AppData\Local\Android\Sdk\emulator\emulator.exe -avd Pixel_9a -wipe-data`

### 2. Force Clean Build
`flutter clean`
`flutter pub get`

### 4. Check Account Sync Status
`C:\Users\Dell\AppData\Local\Android\Sdk\platform-tools\adb.exe shell dumpsys content`

# Common Prompts
0. Project Context: Get context from all *.md files.
1. Update Workflows, Walkthroughs, and Error Handling.
2. Update Product documentations with new changes.
3. Update Readme file with new changes.
