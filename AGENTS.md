# SwiftBackupPrem (fork)

Personal fork of `Juby210/SwiftBackupPrem` — an LSPosed/Xposed module that unlocks
Swift Backup premium and lets it point at a custom Firebase project.

## What it hooks

On `org.swiftapps.swiftbackup` load (`Module.applyHooks`):
- reads remote prefs (`settings` group) into `dx0`, builds `x21` targets.
- if `custom_firebase_app` is true (and app id / api key / db url / sender id /
  project id / client id all non-empty), it initializes a **custom `FirebaseApp`**
  from those prefs instead of the app's own baked config, and hooks
  `SwiftApp.getGoogleAuthAndroidClientId` to return `oauth_client_id`.
- forces premium true via the `common.V` / HomeViewModel setter+getter hooks.
- telemetry suppression when `disable_telemetry`.

### config keys (settings group)
`google_app_id`, `google_api_key`, `firebase_database_url`, `gcm_defaultSenderId`,
`google_storage_bucket`, `project_id`, `oauth_client_id`, `enable_premium`,
`disable_telemetry`, `enable_drive_discovery`, `custom_firebase_app`.

## This fork's fix (`1470c1c`)

upstream crashed with `ExceptionInInitializerError` → `Invalid Firebase Database url
specified: .` when `firebase_database_url` held the default `'.'`. Fixed in
`Module.java`: `xPrefs.reload()` before read + `isValidUrl` = startsWith `https://`
and not `'.'`/`'null'`/empty, so a bad URL falls back to the stock app instead of
crashing at `SwiftApp.onCreate`.

## Status / gotchas

- The fork source is fixed but this session ran the **patched `s1ddhants` binary** on
  device (edited its `settings.xml` + Vector `modules_config.db` by hand), not a build
  of this fork — a `gradle-8.6-bin.zip` disk-space failure blocked the local build. If
  you rebuild: free disk first, `./gradlew :app:assembleDebug`, sign, install.
- `custom_firebase_app` requires a **valid** `oauth_client_id`. For AppAuth's
  `redirect_uri=<pkg>:/oauth` it MUST be an **Android**-type Google OAuth client with
  "Enable custom URI scheme" turned on in GCP; a **Web** client is rejected by Google
  with `Custom scheme URIs are not allowed for 'WEB' client type`.
- After editing prefs in the manager UI or by hand, restart the framework daemon — the
  Vector/LSPosed `vectord` caches config in RAM and already-spawned app processes keep
  the OLD values (symptom: you change the client id but the auth URL still shows the old
  one). Kill/relaunch `vectord`; do not assume a file edit took effect live.
