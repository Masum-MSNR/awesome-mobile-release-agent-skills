---
name: android-signing
description: Generate keystores, enrol in Play App Signing, and understand v1/v2/v3/v4 APK signature schemes. Use this when setting up Android release signing.
---

# Android Signing

## Instructions

Signing is the single most brittle step of an Android release. Get it right once, lock it down, and never touch it again.

### 1. Keystore Generation

Generate a 25-year RSA 2048 upload key (never commit the file):

```bash
keytool -genkey -v \
  -keystore upload-keystore.jks \
  -keyalg RSA -keysize 2048 -validity 9125 \
  -alias upload \
  -storetype JKS \
  -storepass "$KEYSTORE_PASSWORD" \
  -keypass   "$KEY_PASSWORD" \
  -dname "CN=My Company, OU=Mobile, O=My Company, L=City, S=State, C=US"
```

Store the `.jks` file, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD` in your CI secret store and 1Password. **Losing the key means losing the ability to update the app** unless you're on Play App Signing.

### 2. Gradle Wiring

`android/key.properties` (git-ignored):

```properties
storeFile=../upload-keystore.jks
storePassword=...
keyAlias=upload
keyPassword=...
```

`android/app/build.gradle.kts`:

```kotlin
val keystoreProperties = Properties().apply {
    val f = rootProject.file("key.properties")
    if (f.exists()) load(FileInputStream(f))
}

android {
    signingConfigs {
        create("release") {
            storeFile = file(keystoreProperties["storeFile"] as String)
            storePassword = keystoreProperties["storePassword"] as String
            keyAlias = keystoreProperties["keyAlias"] as String
            keyPassword = keystoreProperties["keyPassword"] as String
        }
    }
    buildTypes {
        release { signingConfig = signingConfigs.getByName("release") }
    }
}
```

### 3. Play App Signing (mandatory for new apps)

Google holds the **app signing key**; you hold the **upload key**. Benefits:

- Lost upload key can be reset by Play support (app signing key cannot be).
- Google re-signs each install, enabling automatic APK optimisation.
- Required for Android App Bundles, which are the only accepted format for new apps since August 2021.

Enrol at Play Console → Setup → App integrity → App signing. Upload your upload certificate (`.pem`), not the private key.

### 4. Signature Schemes v1 / v2 / v3 / v4

| Scheme | Protects | Added |
|--------|----------|-------|
| v1 (JAR) | Individual files inside the APK | Android 1.0 |
| v2 (APK) | The entire APK (faster install, tamper-proof) | Android 7.0 |
| v3 (APK) | v2 + key rotation | Android 9.0 |
| v4 (APK) | Incremental install (`.apk.idsig`) | Android 11 |

AGP 8.4+ signs with v1+v2+v3 by default. v4 is emitted automatically for `bundletool build-apks --local-testing`.

### 5. Extracting the Upload Certificate

```bash
keytool -export -rfc \
  -keystore upload-keystore.jks \
  -alias upload \
  -file upload_certificate.pem
```

Upload this `.pem` to Play Console when enrolling.

### 6. Verify a Signed AAB / APK

```bash
apksigner verify --verbose --print-certs app-release.apk
bundletool validate --bundle=app-release.aab
```

Confirm the SHA-256 matches the fingerprint Play shows under App integrity.

### 7. CI: Decoding the Keystore

```yaml
- name: Decode keystore
  run: |
    echo "$ANDROID_KEYSTORE_BASE64" | base64 -d > android/upload-keystore.jks
    cat > android/key.properties <<EOF
    storeFile=../upload-keystore.jks
    storePassword=${{ secrets.KEYSTORE_PASSWORD }}
    keyAlias=${{ secrets.KEY_ALIAS }}
    keyPassword=${{ secrets.KEY_PASSWORD }}
    EOF
```

### 8. Key Rotation (v3)

If you suspect the upload key is compromised:

1. Generate a new keystore.
2. Play Console → App integrity → Request upload key reset (upload new `.pem`).
3. Wait for Google's approval (typically 1–2 business days).
4. Update CI secrets and delete the old `.jks` everywhere.

The app signing key is never rotated for existing apps.

### 9. Common Gotchas

- **`jarsigner` alone is insufficient** for modern Android; always use `apksigner` (invoked by AGP).
- **`storeFile` paths are relative to `android/`**, not the project root — a leading `../` is usually required.
- **Keystore type mismatch (JKS vs PKCS12)** after `keytool` upgrades. Convert with `keytool -importkeystore -srcstoretype JKS -deststoretype PKCS12`.
- **Different upload key per variant** (e.g. dev/staging/prod) — staging builds are often signed with a debug-like keystore checked into the repo.

### 10. Checklist
- [ ] Upload keystore generated with RSA 2048+, 25-year validity.
- [ ] Enrolled in Play App Signing; upload certificate uploaded to Play Console.
- [ ] `key.properties` and `*.jks` git-ignored; CI decodes from base64 secret.
- [ ] `signingConfig` wired to `release` build type only.
- [ ] `apksigner verify` run on CI artifact before upload.
- [ ] Fingerprints recorded in 1Password alongside the keystore.
- [ ] Rotation procedure documented in `docs/signing-rotation.md`.
