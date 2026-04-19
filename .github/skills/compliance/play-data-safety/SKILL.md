---
name: play-data-safety
description: Complete and maintain the Google Play Data Safety form accurately, aligned to shipped SDKs. Use this when launching, adding SDKs, or auditing data practices.
---

# Play Data Safety

## Instructions

Google Play requires every app to declare, via the **Data Safety** form, exactly what data is collected or shared, why, and whether it's encrypted in transit. False declarations are a policy violation and a common source of takedowns. This skill shows how to build an accurate, verifiable declaration.

### 1. Form Structure

Four sections:

1. **Data collection and security** — is any data collected or shared? encryption in transit? deletion mechanism?
2. **Data types** — for each type, purpose, whether collected/shared, whether optional.
3. **Security practices** — encryption in transit, independent security review, deletion request URL.
4. **App categories** — ad IDs, account management, etc.

### 2. Data Type Categories

Google's categories (high level — each has sub-types):

- Personal info (name, email, address, user IDs, phone).
- Financial info (payment, purchase history, credit info).
- Health and fitness.
- Messages (emails, SMS, in-app messages).
- Photos and videos.
- Audio files.
- Files and docs.
- Calendar.
- Contacts.
- App activity (interactions, search history, installed apps).
- Web browsing.
- App info and performance (crash logs, diagnostics, other app performance).
- Device or other identifiers (device IDs, advertising IDs).
- Location (precise and approximate).

### 3. Allowed Purposes

A single data type can be declared with multiple purposes:

- App functionality
- Analytics
- Developer communications
- Advertising or marketing
- Fraud prevention, security, and compliance
- Personalization
- Account management

Every purpose needs at least one concrete use in-app or in an SDK.

### 4. Build a Declaration Spreadsheet

Audit the app and SDKs, then record in `docs/data-safety.yaml`:

```yaml
declarations:
  - data_type: email_address
    collected: true
    optional_collection: false
    shared: false
    purposes: [account_management, app_functionality]
    sources: [app_signup, firebase_auth]

  - data_type: advertising_id
    collected: true
    shared: true
    shared_with: ["admob"]
    purposes: [advertising]
    sources: [admob_sdk]

  - data_type: crash_logs
    collected: true
    shared: false
    purposes: [analytics, app_functionality]
    sources: [crashlytics]

  - data_type: approximate_location
    collected: true
    shared: false
    purposes: [app_functionality]
    sources: [app_direct]
```

### 5. Map SDKs to Declarations

Common SDK → data types:

| SDK | Likely declarations |
|-----|---------------------|
| Firebase Analytics | App activity, device identifiers, crash logs |
| Firebase Crashlytics | Crash logs, diagnostics, device identifiers |
| Firebase Auth | Email, user IDs |
| AdMob | Advertising ID (shared), approximate location |
| Facebook SDK | Advertising ID, app activity, device identifiers |
| Branch / Adjust / AppsFlyer | Device identifiers, app activity; often shared |

Check each SDK's official data-safety documentation and update the spreadsheet when bumping versions.

### 6. Encryption in Transit

You must declare if data is encrypted in transit. If your app and all embedded SDKs use HTTPS, declare **Yes**. Android 9+ default network security config blocks cleartext traffic; keep it:

```xml
<!-- AndroidManifest.xml -->
<application android:usesCleartextTraffic="false" ... />
```

Don't add per-domain cleartext exceptions without a compelling reason.

### 7. Data Deletion

Declare how users can request deletion:

- In-app flow (preferred): a "Delete my account" option in settings.
- Web URL.
- Email address (less preferred; Google flags manual-only flows).

Implement the in-app path with a server-side job that cascades deletion across backend stores and within 30 days by default.

### 8. Submission via Play Publishing API

Data Safety is not part of `supply`; use the Play Publishing API directly to keep it in version control.

```bash
TOKEN=$(./scripts/play-token.sh)
curl -X POST \
  "https://androidpublisher.googleapis.com/androidpublisher/v3/applications/$PKG/edits/$EDIT/datasafety" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @docs/data-safety-payload.json
```

Alternatively, generate the payload from `data-safety.yaml` and paste into the Console UI until the API stabilises.

### 9. Audit Cadence

- Every SDK bump PR: reviewer checks `docs/data-safety.yaml` diff.
- Quarterly: full audit — run `./scripts/grep-collectors.sh` to surface any new APIs that send data off-device.
- Annually: legal/privacy review.

Example collector grep:

```bash
# scripts/grep-collectors.sh
rg -n 'HttpUrl|fetch\(|URLSession|post\(|send\(' --type kotlin --type swift \
   --glob '!**/test/**' --glob '!**/Tests/**'
```

### 10. Consequences of Mismatch

- **Warning email** from Play with a 14-day fix window.
- **Removal from search / featured placement**.
- **Takedown** for repeated or egregious misdeclaration.
- **Policy strike** which, accumulated, closes the developer account.

None of these are rare. Take the form seriously.

### 11. Interaction with Other Policies

- **Families policy** (apps for kids): stricter rules on advertising IDs and personalization.
- **Background location**: requires separate prominent disclosure and declarations form.
- **Health Connect / Fitness APIs**: additional developer policy attestations.

### 12. Checklist
- [ ] `docs/data-safety.yaml` committed and up to date with shipped SDKs.
- [ ] Declaration submitted via API or UI; screenshot archived in `docs/data-safety-snapshots/`.
- [ ] `usesCleartextTraffic` disabled; HTTPS everywhere verified.
- [ ] In-app deletion flow implemented; deletion URL public.
- [ ] SDK bumps include a Data Safety review in the PR checklist.
- [ ] Quarterly grep-audit run; no undeclared network collectors.
- [ ] Legal/privacy sign-off on record annually.
