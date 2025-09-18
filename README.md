# Firebase App Distribution via GitHub Actions (Flutter)

A compact, copy-paste friendly guide to automatically build and distribute your Flutter app (APK / AAB / IPA) using **GitHub Actions** and **Firebase App Distribution** with a **service account JSON** (recommended, token-deprecated).

---

## Quick overview

1. Register Android/iOS apps in Firebase and add their config files to your Flutter project.
2. Create a Firebase service account (JSON) and give it project permissions.
3. Store the JSON and Firebase App IDs as **GitHub Secrets**.
4. Add the workflow YAML to `.github/workflows/firebase_distribute.yml`.
5. Push to `main` (or `dev` if you prefer) — CI builds and uploads your artifact to Firebase.

---

## Prerequisites

* Flutter project with working `android/` and/or `ios/` folders.
* Firebase project with Android and/or iOS apps registered.
* GitHub repository for the project and permission to add Actions & Secrets.
* (Optional) For iOS: code signing setup for real-device IPA builds.

---

## 1. Register apps in Firebase

1. Go to **Firebase Console → Project Settings → General → Your apps**.
2. If not already added, `Add app → Android` and `Add app → iOS`. Use exact package / bundle ids from your Flutter project:

   * Android package: `android/app/src/main/AndroidManifest.xml` (applicationId)
   * iOS bundle id: `ios/Runner.xcodeproj` → Runner target → General → Bundle Identifier
3. Download and add the config files:

   * Android: `google-services.json` → place at `android/app/google-services.json`
   * iOS: `GoogleService-Info.plist` → place at `ios/Runner/GoogleService-Info.plist`

> Tip: After adding the files run `flutter clean && flutter pub get` locally to confirm nothing breaks.

---

## 2. Create service account JSON (CI auth)

1. In Firebase Console → Project Settings → Service accounts → Click **Generate new private key** (or use Google Cloud Console → IAM & Admin → Service Accounts).
2. Download the JSON file.
3. (Optional) In GCP IAM, you can give the service account one of: **Project Editor**, **Firebase Admin**, or a narrower role with App Distribution permissions. If unsure, `Editor` works.

---

## 3. Add GitHub Secrets

Navigate to **GitHub → Settings → Secrets and variables → Actions → New repository secret** and add:

* `FIREBASE_SERVICE_ACCOUNT` → paste the **entire JSON file content** (single-line or multi-line works).
* `FIREBASE_ANDROID_APP_ID` → Firebase Android App ID (found in Project Settings → General → Android app card; looks like `1:12345:android:abcdef`).
* `FIREBASE_IOS_APP_ID` → Firebase iOS App ID (if you have an iOS app; looks like `1:12345:ios:abcdef`).
* (Optional) `FIREBASE_DISTRIBUTION_GROUPS` → e.g. `testers` or comma-separated emails.

---

## 4. Add workflow file

Create `.github/workflows/firebase_distribute.yml` and paste the following **optimized** workflow. It builds Android on Linux and iOS only when manually triggered (saves macOS minutes). Edit labels like `groups` and `flutter-version` to match your project.

```yaml
name: Firebase App Distribution

on:
  push:
    branches: [ main ]
  workflow_dispatch: {}

jobs:
  android:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: 'stable'

      - name: Install dependencies
        run: flutter pub get

      - name: Build Android APK
        run: flutter build apk --release

      - name: Upload Android to Firebase App Distribution
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_ANDROID_APP_ID }}
          serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          groups: testers
          file: build/app/outputs/flutter-apk/app-release.apk

  ios:
    # Run iOS step only when manually triggered to save macOS minutes
    if: ${{ github.event_name == 'workflow_dispatch' }}
    runs-on: macos-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: 'stable'

      - name: Install dependencies
        run: flutter pub get

      - name: Build iOS (no codesign)
        run: flutter build ipa --release --no-codesign

      - name: Upload iOS to Firebase App Distribution
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_IOS_APP_ID }}
          serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          groups: testers
          file: build/ios/ipa/Runner.ipa
```

### Alternatives

* Use `firebase-tools` CLI directly with `--service-credentials service-account.json`. Example step:

```bash
npm install -g firebase-tools
echo "${{ secrets.FIREBASE_SERVICE_ACCOUNT }}" > service-account.json
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-release.apk \
  --app ${{ secrets.FIREBASE_ANDROID_APP_ID }} \
  --service-credentials service-account.json \
  --groups "testers" \
  --release-notes "CI build"
```

---

## 5. Commit & push workflow

```bash
git add .github/workflows/firebase_distribute.yml
git commit -m "ci: add firebase app distribution workflow"
git push origin main
```

Once pushed, go to **GitHub → Actions** to watch the run logs.

---

## 6. How to trigger

* Push to `main` (configured trigger).
* Or open the Actions tab and click **Run workflow** (manual trigger) — useful for the iOS job.

---

## Troubleshooting (common issues)

* **App not registered / invalid App ID**: Confirm the Android/iOS App exists in Firebase Console and copy the App ID value (not the package name).
* **Service account permission errors**: Ensure the service account has adequate project permissions (Editor/Firebase Admin or App Distribution-specific roles).
* **Missing artifact path**: Verify the produced APK/AAB/IPA path after local build and update `file:` if different.
* **`firebase-debug.log`**: If the CLI fails, inspect this log for detailed errors.
* **iOS code signing**: `--no-codesign` produces unsigned IPA (useful for testing on simulators or internal workflows). For real-device installs, set up code signing (certs & provisioning) in Actions.

---

## Optional: Make iOS job run only on tag or manual trigger

Change the `if:` on the `ios` job to run for tags (releases) or keep `workflow_dispatch` for manual runs.

---

## Minimal checklist to reuse for other projects

1. Register apps in Firebase and add config files.
2. Create service-account JSON and store it as `FIREBASE_SERVICE_ACCOUNT` secret.
3. Add `FIREBASE_ANDROID_APP_ID` and `FIREBASE_IOS_APP_ID` secrets.
4. Drop workflow YAML into `.github/workflows/` and push.

---

If you want, I can also:

* Provide a version of the workflow that **builds AAB** instead of APK.
* Add iOS **code signing** steps (with secrets for cert & provisioning).
* Convert this into a GitHub repo README and commit it for you (if you give access).

---

Good luck — paste this README into your repo and you’re ready to reuse the pipeline across projects! 🎉
