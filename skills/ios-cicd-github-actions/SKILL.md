---
name: ios-cicd-github-actions
description: Set up iOS CI/CD in GitHub Actions: pin macOS/Xcode, import signing certs, fetch provisioning profiles via App Store Connect API, build an IPA, and upload to TestFlight or the App Store.
---

# iOS CI/CD with GitHub Actions (Build, Sign, Upload)

This skill helps design, implement, and debug a **framework-agnostic iOS CI/CD pipeline** using GitHub Actions.
It focuses on the essential Apple requirements: build environment, signing, provisioning, archiving, and TestFlight upload.

---

## What “done” looks like

A GitHub Actions workflow that:

1. Uses a **pinned macOS runner**
2. Uses a **pinned Xcode version**
3. Imports an **Apple Distribution certificate** into a temporary keychain
4. Downloads **App Store provisioning profiles** using App Store Connect API keys
5. Builds a **Release archive** and exports an **IPA**
6. Uploads the IPA to **TestFlight**

---

## Canonical CI/CD order (do not reorder)

### 1. Pin the build environment
- Explicit `runs-on` (never `macos-latest`)
- Explicit Xcode version
- Set a job timeout

### 2. Checkout source
- Use `actions/checkout`
- Enable submodules if required

### 3. Install build dependencies
- Language SDKs, CLIs, package managers
- Restore dependencies deterministically (lockfiles)

### 4. Import signing certificate
- Use an Apple Distribution `.p12`
- Import into a temporary keychain
- Never store raw certificates in the repo

### 5. Download provisioning profiles
- Use App Store Connect **API key authentication**
- Match bundle identifier exactly
- Use `IOS_APP_STORE` profile type for TestFlight

### 6. Build, archive, export IPA
- Build in Release
- Produce an `.ipa` suitable for App Store distribution
- Export to a predictable path

### 7. Upload to TestFlight
- Authenticate with the same App Store Connect API key
- Ensure build number is valid and incremented

---

## Required GitHub Secrets

### Apple Distribution Certificate
- `APPSTORE_CERTIFICATE_P12` – base64-encoded `.p12`
- `APPSTORE_CERTIFICATE_P12_PASSWORD` – password for the `.p12`

### App Store Connect API
- `APPSTORE_ISSUER_ID`
- `APPSTORE_KEY_ID`
- `APPSTORE_PRIVATE_KEY` – contents of the `.p8` file

### App Metadata
- `IOS_BUNDLE_ID` – e.g. `com.example.myapp`

---

## Minimal Framework-Agnostic Workflow Template

Replace the build step with your own build system.
Your build **must produce an IPA** at the specified path.

```yaml
name: iOS CI

on:
  workflow_dispatch:
  push:
    branches: [ main ]

jobs:
  ios:
    runs-on: macos-15
    timeout-minutes: 45

    env:
      IOS_BUNDLE_ID: ${{ secrets.IOS_BUNDLE_ID }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Select Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: "16.3"

      - name: Import Apple Distribution Certificate
        uses: apple-actions/import-codesign-certs@v4
        with:
          create-keychain: true
          keychain-password: ${{ secrets.APPSTORE_CERTIFICATE_P12_PASSWORD }}
          p12-file-base64: ${{ secrets.APPSTORE_CERTIFICATE_P12 }}
          p12-password: ${{ secrets.APPSTORE_CERTIFICATE_P12_PASSWORD }}

      - name: Download Provisioning Profile
        uses: apple-actions/download-provisioning-profiles@v4
        with:
          bundle-id: ${{ env.IOS_BUNDLE_ID }}
          profile-type: IOS_APP_STORE
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}

      - name: Build, Archive, Export IPA
        shell: bash
        run: |
          set -euo pipefail
          echo "Run your build system here"
          echo "Ensure an IPA is written to artifacts/MyApp.ipa"
          mkdir -p artifacts

      - name: Upload to TestFlight
        uses: apple-actions/upload-testflight-build@v1
        with:
          app-type: ios
          app-path: artifacts/MyApp.ipa
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
```

---

## Debugging Playbook

Identify the failure stage first:

1. **Runner / Xcode**
   - Wrong Xcode version
   - Toolchain not available

2. **Certificate Import**
   - Invalid base64 encoding
   - Wrong `.p12` password
   - Missing private key

3. **Provisioning**
   - Bundle ID mismatch
   - API key lacks App Manager permissions
   - Wrong profile type

4. **Build Signing**
   - Team ID mismatch
   - Entitlements mismatch
   - Signing identity not found in keychain

5. **IPA Export**
   - Archive created but IPA missing
   - Export method incorrect

6. **TestFlight Upload**
   - Build number reuse
   - API key scope issues
   - Missing compliance metadata

Always request:
- Bundle ID
- Distribution target (TestFlight vs App Store)
- Failing log lines
- IPA output path

---

## Skill Behavior Expectations

When this skill is active:
- Produce **copy‑pasteable workflows**
- Prefer explicit versions and secret names
- Avoid framework‑specific assumptions
- Optimize for repeatability and security
