# AGENTS.md

## Cursor Cloud specific instructions

AnkiDroid is a multi-module **Gradle** Android app (Kotlin). It builds an APK; there is no
server/database to run. The standard build/test/lint commands live in `.github/workflows/`
(`tests_unit.yml`, `lint.yml`) and `.github/workflows/README.md` — prefer those as the source of truth.

### Environment (already provisioned in the VM snapshot)
- **JDK 21** is required (build aborts outside JDK 21–25). `JAVA_HOME`, `ANDROID_HOME`,
  `ANDROID_SDK_ROOT`, and the SDK `PATH` are exported in `~/.bashrc`.
- The **Android SDK** lives at `~/android-sdk` (`platform-tools`, `platforms;android-36`,
  `platforms;android-35`, `build-tools;36.0.0`). `local.properties` (untracked) sets `sdk.dir` and
  `fatal_warnings=false`.
- **Gotcha:** non-interactive shells do NOT source `~/.bashrc`. When running Gradle from a script,
  export the vars first or the SDK won't be found:
  ```bash
  export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
  export ANDROID_HOME=$HOME/android-sdk ANDROID_SDK_ROOT=$HOME/android-sdk
  ```

### Build / test / lint / run
- Build debug APK: `./gradlew assemblePlayDebug` (outputs in `AnkiDroid/build/outputs/apk/play/debug/`).
- Unit tests (Robolectric, run on the JVM — no device needed): first `./gradlew robolectricSdkDownload`,
  then `./gradlew jacocoUnitTestReport` (full suite is large; for quick checks target a module/class,
  e.g. `./gradlew :libanki:testDebugUnitTest --tests "com.ichi2.anki.libanki.SchedulerTest"`).
- Lint (full CI command is heavy):
  `./gradlew lintPlayDebug :api:lintDebug :libanki:lintDebug ktLintCheck lintVitalFullRelease lint-rules:test`.
- **Running the GUI app / emulator:** the cloud VM has **no KVM / hardware virtualization**
  (`emulator -accel-check` reports `/dev/kvm is not found`). An x86_64 AVD *does* still start
  **headless** under QEMU's software CPU emulation (TCG) — boots to `sys.boot_completed=1`, and the
  freshly-built debug APK installs and launches (verified: `IntroductionActivity` / `DeckPicker`).
  **However it is impractically slow:** cold first boot ~8–9 min (warm boot ~3 min) and the system
  throws frequent `SystemUI`/`Pixel Launcher isn't responding` ANRs, so it is **not suitable for
  routine instrumented (`jacocoAndroidTestReport`) or interactive GUI testing**. Prefer the
  JVM/Robolectric unit tests in `:libanki` (collection/notes/scheduler) for normal verification.
  - To start it anyway (install `emulator` + `system-images;android-35;google_apis;x86_64`, create AVD `test_avd`):
    ```bash
    "$ANDROID_HOME"/emulator/emulator @test_avd -no-window -no-audio -no-snapshot -no-accel \
      -gpu swiftshader_indirect -no-boot-anim &
    adb wait-for-device   # then poll: adb shell getprop sys.boot_completed
    ```

### Localization tool (auxiliary, `tools/localization`)
- Node tool using **yarn 4** via corepack. Install with `corepack yarn install` inside the folder.
- Scripts: `corepack yarn lint`, `corepack yarn checkformat`, `corepack yarn build`, `corepack yarn test`.
