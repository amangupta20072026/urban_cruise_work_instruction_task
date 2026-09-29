# `/.well-known/` files (host on `app.urbancruise.com`)

These are served by the domain, not the app. Both must be reachable
over HTTPS with a valid certificate, `Content-Type: application/json`,
and NO redirects.

---

## `assetlinks.json`

Path: `https://app.urbancruise.com/.well-known/assetlinks.json`

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.pinaak",
      "sha256_cert_fingerprints": [
        "<UPLOAD_KEY_SHA256>",
        "<PLAY_APP_SIGNING_SHA256>"
      ]
    }
  }
]
```

**How to fill in the fingerprints:**

```bash
# Upload key (the keystore you sign the AAB with locally):
keytool -list -v -keystore path/to/release.keystore -alias <alias> | grep SHA256

# Play App Signing key (Google-managed):
#   Play Console → your app → Setup → App integrity
#   → App signing key certificate → SHA-256 certificate fingerprint
```

Include BOTH values. Devices that install from Play verify against
the Play App Signing key; internal test installs verify against
the upload key.

---

## `apple-app-site-association`

Path: `https://app.urbancruise.com/.well-known/apple-app-site-association`
(No `.json` extension.)

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appIDs": ["<TEAM_ID>.com.pinaak"],
        "components": [
          { "/": "/bookings/*",     "comment": "Booking detail / pay / feedback" },
          { "/": "/trip/*",         "comment": "Live customer trip" },
          { "/": "/quotations/*",   "comment": "Quotation detail" },
          { "/": "/vendor/*",       "comment": "Vendor screens" },
          { "/": "/driver/*",       "comment": "Driver screens" },
          { "/": "/uc/*",           "comment": "UC screens" },
          { "/": "/notifications",  "comment": "Notification centre" },
          { "/": "/support",        "comment": "Support" }
        ]
      }
    ]
  }
}
```

**How to fill in the team id:**

Apple Developer → Membership → Team ID (10-char alphanumeric).
The `appIDs` entry is `<TEAM_ID>.<BUNDLE_ID>`.

---

## Verifying

### Android

```bash
# Install a signed build (App Links do NOT verify on debug by default).
adb install app-release.apk

# Verification status:
adb shell pm get-app-links com.pinaak

# Expected: `verified` next to https://app.urbancruise.com
# If `legacy_failure` or `verification_failed`, check the JSON.

# Force re-verify after fixing the file:
adb shell pm verify-app-links --re-verify com.pinaak

# Send an intent:
adb shell am start -W -a android.intent.action.VIEW \
  -d "https://app.urbancruise.com/bookings/01HXXX00000000000000000000" \
  com.pinaak
```

### iOS

```bash
# Confirm the file is served correctly:
curl -I https://app.urbancruise.com/.well-known/apple-app-site-association
# Must show: 200 OK, Content-Type: application/json, no Location header.

# Test on simulator (installed app):
xcrun simctl openurl booted \
  "https://app.urbancruise.com/bookings/01HXXX00000000000000000000"

# On a real device tethered to a Mac:
sudo swcutil dl -d app.urbancruise.com
# Shows verification result and any error.
```

---

## Ongoing hygiene

- After any change to `assetlinks.json` / AASA, next install triggers
  a fresh fetch. Existing installs cache verification for hours; use
  `pm verify-app-links --re-verify` on Android and reinstall on iOS
  to force re-check.
- When rotating the Android signing key: publish `assetlinks.json`
  containing BOTH the old and new SHA-256. Remove the old one only
  after every distributed build is signed with the new key.
- Deprecate a domain gracefully: keep serving both files for at least
  6 months after the last app version that references it stops being
  distributed. OS-cached verification lags.
