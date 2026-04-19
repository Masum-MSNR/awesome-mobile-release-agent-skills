---
name: alternative-stores
description: Publish Android apps to Amazon Appstore, Huawei AppGallery, Samsung Galaxy Store, and deal with HMS vs GMS. Use this when targeting non-Google Android markets.
---

# Alternative Android Stores

## Instructions

Non-Google markets matter for regional reach (Huawei in China and Europe), accessory integration (Samsung), and Amazon's Fire ecosystem. Each store has its own submission API, signing quirks, and review workflow.

### 1. When to Target Each Store

| Store | Audience | Signing | APIs |
|-------|----------|---------|------|
| Amazon Appstore | Fire tablets, Fire TV, some Windows devices | Amazon signs for you | GMS typically unavailable |
| Huawei AppGallery | Huawei devices (Europe, Middle East, China) | You sign; AppGallery Connect verifies | HMS (Huawei Mobile Services) |
| Samsung Galaxy Store | All Samsung devices | You sign | GMS + Samsung SDKs |
| Xiaomi GetApps, Oppo, Vivo | China market | You sign | GMS absent in China |

### 2. Amazon Appstore

Submit via [Developer Console](https://developer.amazon.com/apps-and-games/console) or the [App Submission API](https://developer.amazon.com/docs/app-submission-api/overview.html).

Key differences from Play:

- Accepts APK or AAB. Amazon re-signs APKs on device distribution.
- No Google Play Services — use Amazon Device Messaging (ADM) in place of FCM on Fire devices.
- Simpler staged rollout: either live or not.

Automate with `curl`:

```bash
# Get token
TOKEN=$(curl -sX POST https://api.amazon.com/auth/o2/token \
  -d "grant_type=client_credentials" \
  -d "client_id=$AMAZON_CLIENT_ID" \
  -d "client_secret=$AMAZON_CLIENT_SECRET" \
  -d "scope=appstore::apps:readwrite" | jq -r .access_token)

# Upload APK to the edit
curl -X PUT "https://developer.amazon.com/api/appstore/v1/applications/$APP_ID/edits/$EDIT_ID/apks/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @app-release.apk
```

### 3. Huawei AppGallery

Required if you ship on devices without Google Mobile Services (most new Huawei phones since 2019).

Setup:

1. Register at [AppGallery Connect](https://developer.huawei.com/consumer/en/service/josp/agc).
2. Integrate **HMS Core** replacements for common GMS APIs:
   - Google Maps → HMS Map Kit
   - Firebase Messaging → HMS Push Kit
   - Google Sign-In → HMS Account Kit
   - Google Location → HMS Location Kit
3. Use the [G+H strategy](https://developer.huawei.com/consumer/en/codelab/HMSPushKit/index.html): detect at runtime which services are available and route calls accordingly.

Automate uploads with the [AppGallery Connect Publishing API](https://developer.huawei.com/consumer/en/doc/AppGalleryConnect/agcapi-appinfo_v1):

```bash
# Exchange client credentials for token
TOKEN=$(curl -s -X POST "https://connect-api.cloud.huawei.com/api/oauth2/v1/token" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"client_credentials\",\"client_id\":\"$HUAWEI_CLIENT_ID\",\"client_secret\":\"$HUAWEI_CLIENT_SECRET\"}" \
  | jq -r .access_token)
```

Review takes 1–3 business days for China, often same-day for other regions.

### 4. Samsung Galaxy Store

Target the ~30% of Android devices that are Samsung. Mostly compatible with Google Play AABs, but:

- Samsung IAP SDK replaces Play Billing for Galaxy Store purchases.
- Galaxy Watch / Foldable-specific features (multi-window, flex mode) unlock bonus placement.
- Use the [Seller Portal API](https://developer.samsung.com/galaxy-store/api-reference.html):

```bash
curl -X POST "https://devapi.samsungapps.com/seller/contentSubmit" \
  -H "Authorization: Bearer $SAMSUNG_TOKEN" \
  -H "service-account-id: $SAMSUNG_ACCOUNT_ID" \
  -H "Content-Type: application/json" \
  -d '{"contentId":"000000123456"}'
```

### 5. One AAB, Multiple Stores

Build variant-per-store so the correct SDKs are included:

```kotlin
// app/build.gradle.kts
android {
    flavorDimensions += "store"
    productFlavors {
        create("google")  { dimension = "store"; buildConfigField("String", "STORE", "\"google\"") }
        create("huawei")  { dimension = "store"; buildConfigField("String", "STORE", "\"huawei\"") }
        create("amazon")  { dimension = "store"; buildConfigField("String", "STORE", "\"amazon\"") }
        create("samsung") { dimension = "store"; buildConfigField("String", "STORE", "\"samsung\"") }
    }
}
```

Then:

```bash
./gradlew :app:bundleHuaweiRelease
./gradlew :app:bundleAmazonRelease
```

### 6. Push Notifications Cross-Store

Runtime detection:

```kotlin
val usesHms = GoogleApiAvailability.getInstance()
    .isGooglePlayServicesAvailable(this) != ConnectionResult.SUCCESS
if (usesHms) HmsInstanceId.getInstance(this).getToken(appId, "HCM")
else         FirebaseMessaging.getInstance().token
```

Send the token to your backend with a `provider` field: `fcm` / `hms` / `adm`.

### 7. Fastlane Plugins

- `fastlane-plugin-huawei_appgallery_connect`
- `fastlane-plugin-amazon_app_submission`
- `fastlane-plugin-samsung_galaxy_store`

Prefer these over hand-rolled `curl` when supported.

### 8. Policy Differences

- Amazon forbids apps that only link out to web stores (bypassing their IAP).
- Huawei China has stricter content review (no political content, gambling, anti-government).
- Samsung permits richer metadata (rich text descriptions, What's New sections per locale).

### 9. Checklist
- [ ] Decision made per target region: which stores are worth the submission cost.
- [ ] Variant-per-store build flavors in Gradle; store-specific SDKs only included where needed.
- [ ] HMS replacements integrated with runtime GMS/HMS detection.
- [ ] API credentials for each store stored separately in CI; scoped to publish-only.
- [ ] Policy diff documented per store in `docs/store-policies.md`.
- [ ] Release cadence matches Play (all stores released from the same commit, within 72h).
