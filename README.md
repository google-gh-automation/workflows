name: Android Build & APK

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build-apk:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Locate Gradle project root
        id: locate
        run: |
          SETTINGS_FILE=$(find . -maxdepth 3 \( -name "settings.gradle.kts" -o -name "settings.gradle" \) | head -n 1)
          if [ -z "$SETTINGS_FILE" ]; then
            echo "❌ ERROR: Android project files (settings.gradle.kts, app/, build.gradle.kts) not found in this repo!"
            exit 1
          fi
          PROJECT_DIR=$(dirname "$SETTINGS_FILE")
          echo "project_dir=$PROJECT_DIR" >> "$GITHUB_OUTPUT"
          echo "Found Gradle project at: $PROJECT_DIR"

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - name: Prepare keystore, env & google-services.json
        env:
          GOOGLE_SERVICES_JSON: ${{ secrets.GOOGLE_SERVICES_JSON }}
        working-directory: ${{ steps.locate.outputs.project_dir }}
        run: |
          if [ ! -f debug.keystore ]; then
            if [ -f debug.keystore.base64 ]; then
              base64 -d debug.keystore.base64 > debug.keystore
            else
              keytool -genkeypair -v -keystore debug.keystore \
                -storepass android -alias androiddebugkey -keypass android \
                -keyalg RSA -keysize 2048 -validity 10000 \
                -dname "CN=Android Debug,O=Android,C=US"
            fi
          fi
          if [ ! -f .env ] && [ -f .env.example ]; then
            cp .env.example .env
          fi
          if [ ! -f app/google-services.json ]; then
            if [ -n "$GOOGLE_SERVICES_JSON" ]; then
              if [[ "$GOOGLE_SERVICES_JSON" == \{* ]]; then
                printf '%s' "$GOOGLE_SERVICES_JSON" > app/google-services.json
              else
                printf '%s' "$GOOGLE_SERVICES_JSON" | base64 -d > app/google-services.json
              fi
            else
              echo "❌ ERROR: app/google-services.json is missing!"
              echo "Add a repository Actions secret named GOOGLE_SERVICES_JSON (contents of app/google-services.json) or upload app/google-services.json."
              exit 1
            fi
          fi

      - name: Accept Android SDK licenses
        run: |
          SDK_DIR="${ANDROID_HOME:-$ANDROID_SDK_ROOT}"
          mkdir -p "$SDK_DIR/licenses"
          printf "\n24333f8a63b6825ea9c5514f83c2829b004d1fee\n8933bad161af4178b1185d1a37fbf41ea5269c55\nd56f5187479451eabf01fb78af6dfcb131a6481e\n" >> "$SDK_DIR/licenses/android-sdk-license"
          printf "\n84831b9409646a918e30573bab4c9c91346d8abd\n" >> "$SDK_DIR/licenses/android-sdk-preview-license"

      - name: Setup Gradle 9.3.1
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '9.3.1'

      - name: Build Debug APK
        working-directory: ${{ steps.locate.outputs.project_dir }}
        run: gradle :app:assembleDebug --no-daemon --stacktrace

      - name: Run Unit Tests
        working-directory: ${{ steps.locate.outputs.project_dir }}
        run: gradle :app:testDebugUnitTest --no-daemon --stacktrace

      - name: Upload APK Artifact
        uses: actions/upload-artifact@v4
        with:
          name: SampleSection-debug-apk
          path: ${{ steps.locate.outputs.project_dir }}/app/build/outputs/apk/debug/*.apk
          if-no-files-found: error
