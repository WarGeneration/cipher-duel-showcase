<!--
English translation of NEXT_STEPS.md — the working plan the agent keeps up to date. The original is in Ukrainian.
Each step is marked with who does it: [me] — the human (accounts, payments, keys), [Claude] — the agent (code and configuration).
-->

# Next steps

Updated 2026-10-21. Production = `main`: avatars and choosing a digit position are deployed.
Marks: **[me]** — done by me (accounts, payments, keys), **[Claude]** — code and configuration.

## 1. Push notifications on Android (FCM) — next
Goal: game invitations and friend requests arrive even when the app is closed.

- **[me]** Create a project in the Firebase Console and add the Android app `com.cipherduel` (with the release key's SHA-1).
- **[me]** Download `google-services.json` (it goes into `mobile/app/`).
- **[me]** Create a service-account key (JSON) for sending from the server. It is a secret: it goes only to the server,
  never into the repository or the chat.
- **[Claude]** Server: a device-token table (`POST/DELETE /api/me/devices`) and sending via FCM HTTP v1 when an invitation,
  a friend request or an accepted request arrives (only if the person has no open `/ws/me` socket).
- **[Claude]** App: getting the token, the notification permission (Android 13+), a notification channel, tapping a notification
  opens the invitation or friends. Strings in en/uk. Removing the token on sign-out.
- **[Claude]** Tests, updating `API.md` and CLAUDE.md, deployment (only after my "deploy").
- Later for iOS: an APNs key from Apple uploaded to Firebase; the server code stays the same.

## 2. Preparing for Google Play
- **[me]** A Google Play developer account ($25 one-time, identity verification).
- **[Claude]** A privacy policy page and a web page for account deletion on the server (Play requires both).
- **[Claude]** An AAB build (`bundleRelease`), store screenshots from the screen gallery, descriptions in en/uk.
- **[me]** In Play Console: enable Play App Signing and give the agent the Play key's SHA-256. It goes into
  `assetlinks.json` (links `/f/…`, `/r/…`) and into the Android OAuth client in Google Cloud.
- **[me]** Move Google Auth Platform from Testing to Production — right now only test users can sign in with Google.
- **[me]** Play Console questionnaires: Data safety, content ratings, target audience.
- Note: a new personal Play account must first run a closed test (currently 12 testers for 14 days) before it gets
  access to production. Testers are worth finding in advance.

## 3. iOS and TestFlight (waiting for Apple)
The Apple Developer Program application was submitted around 11 October. If there's still no answer, contact support:
developer.apple.com/contact → Membership and Account → Program Enrollment.

After approval:
- **[me]** An App Store Connect record, bundle ID `com.cipherduel`.
- **[me]** An App Store Connect API key (Issuer ID, Key ID, `.p8`) entered into Codemagic as an integration
  (the key itself stays with me). Also give the agent the app's Apple ID and the Team ID.
- **[me]** Google Cloud: an iOS OAuth client for `com.cipherduel`, give the agent its Client ID.
- **[Claude]** Put `APP_STORE_APPLE_ID` into `codemagic.yaml`, `GOOGLE_IOS_CLIENT_ID` and the URL scheme into
  `mobile/iosApp/project.yml`, add the client to the server's allowed Google client IDs and deploy.
- **[Claude]** The first signed build in Codemagic. Likely fixes to signing or Xcode that can't be checked on Linux.
- **[me]** Install TestFlight on an iPhone and test the app.
- Then push on iOS (APNs, see step 1).

## Later
- An own domain instead of the current free subdomain (then update the links, the certificate, `assetlinks`, OAuth).
