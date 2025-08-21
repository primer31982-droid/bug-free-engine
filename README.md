name: Build, Sign & Release Android APK/AAB

on:
  push:
    tags:
      - 'v*'   # Trigger on any version tag

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      with:
        fetch-depth: 0

    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        distribution: temurin
        java-version: 17

    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: 18

    - name: Install dependencies
      run: npm install

    - name: Build web app
      run: npm run build

    - name: Prepare Android build
      run: |
        npm install @capacitor/core @capacitor/cli @capacitor/android
        npx cap sync android

    - name: Bump Android version from Git tag
      run: |
        TAG=${GITHUB_REF#refs/tags/}
        CLEAN_TAG=$(echo $TAG | sed 's/-.*//')   # Remove -beta, -internal, etc.
        VERSION_CODE=$(grep versionCode android/app/build.gradle | awk '{print $2}')
        NEW_CODE=$((VERSION_CODE + 1))
        sed -i "s/versionCode .*/versionCode $NEW_CODE/" android/app/build.gradle
        sed -i "s/versionName \".*\"/versionName \"$CLEAN_TAG\"/" android/app/build.gradle

    - name: Decode keystore
      run: |
        echo $SIGNING_KEYSTORE | base64 -d > android/app/my-release-key.jks
      env:
        SIGNING_KEYSTORE: ${{ secrets.SIGNING_KEYSTORE }}

    - name: Build Signed Release APK & AAB
      working-directory: android
      run: |
        ./gradlew clean
        ./gradlew bundleRelease assembleRelease \
          -Pandroid.injected.signing.store.file=my-release-key.jks \
          -Pandroid.injected.signing.store.password=${{ secrets.SIGNING_STORE_PASSWORD }} \
          -Pandroid.injected.signing.key.alias=${{ secrets.SIGNING_KEY_ALIAS }} \
          -Pandroid.injected.signing.key.password=${{ secrets.SIGNING_KEY_PASSWORD }}

    - name: Upload to GitHub Release
      uses: softprops/action-gh-release@v1
      with:
        files: |
          android/app/build/outputs/apk/release/app-release.apk
          android/app/build/outputs/bundle/release/app-release.aab
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    - name: Determine Play Store Track
      id: track
      run: |
        TAG=${GITHUB_REF#refs/tags/}
        if [[ $TAG == *"-beta" ]]; then
          echo "track=beta" >> $GITHUB_OUTPUT
        elif [[ $TAG == *"-internal" ]]; then
          echo "track=internal" >> $GITHUB_OUTPUT
        else
          echo "track=production" >> $GITHUB_OUTPUT
        fi

    - name: Upload AAB to Google Play
      uses: r0adkll/upload-google-play@v1
      with:
        serviceAccountJsonPlainText: ${{ secrets.GOOGLE_PLAY_SERVICE_ACCOUNT }}
        packageName: com.example.documentproof   # <-- replace with your app ID
        releaseFiles: android/app/build/outputs/bundle/release/app-release.aab
        track: ${{ steps.track.outputs.track }}
        inAppUpdatePriority: 2
        status: completed
