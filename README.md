# Call Recorder (ACR-style) — Android

A personal call-recording app built in **Kotlin + Jetpack Compose**. It does what you listed:

| Feature | Where it lives |
|---|---|
| Detect incoming & outgoing calls | `receiver/CallReceiver.kt` + `receiver/CallStateTracker.kt` |
| Show caller number | resolved in the receiver (incoming) and from the call log (outgoing) in `service/CallRecordingService.kt` |
| Record microphone audio (your voice) | `recorder/CallAudioRecorder.kt` |
| Save recordings | written to `Android/data/com.acr.recorder/files/recordings/*.m4a` |
| Upload recordings to your server | `upload/UploadWorker.kt` (WorkManager + OkHttp, auto-retry) |
| Call logs + metadata | Room DB in `data/` shown in `ui/CallLogScreen.kt` |

---

## ⚠️ Read this first — what Android actually allows

This is the single most important thing about call recorders, and it's why the feature you wrote was **"record your voice"** rather than "record both sides":

- **Android ≤ 9:** apps could record both sides of a call.
- **Android 10+ (2019 onward):** Google **blocked** third-party apps from capturing the *remote* party's audio. The `VOICE_CALL` / `VOICE_DOWNLINK` sources are reserved for system/OEM apps.
- **What works now:** recording the **microphone**. That reliably captures **your voice**, and captures **both sides only when the call is on speakerphone**. The app tries `VOICE_COMMUNICATION` first (some OEMs route more through it) and falls back to `MIC`.
- True two-way recording on a modern stock phone is not possible without **root**, a **rooted/Magisk module**, or **manufacturer support** (some Xiaomi/Samsung/Realme builds expose it).
- **Play Store:** Google's policy bans call recording via the Accessibility API, so apps like this are generally **not publishable** on Play. This project is meant for **sideloading on your own device.**

## ⚖️ Legal note (not legal advice)

Recording calls is regulated and the rules differ by country and even by who you're calling. Some places require **all parties to consent**. You are responsible for using this lawfully where you are and recording only calls you're entitled to record. I'm not a lawyer — if you're unsure, check your local law.

---

## Build & run

1. Install **Android Studio** (Hedgehog or newer).
2. `File → Open` this `ACRApp` folder. Let Gradle sync (it downloads AGP 8.5, Kotlin 1.9, Compose, Room, WorkManager, OkHttp).
3. Plug in a **physical phone** (call recording can't be tested on an emulator) with USB debugging on.
4. Press **Run**. On first launch grant: **Microphone, Phone, Call logs, Notifications**.
5. Make a call. While connected you'll see a "Recording call" notification; when it ends a row appears in the list.

> Tip: for the clearest capture of both sides, turn on **speakerphone**.

## Wire up the upload server

The included `server/` is a tiny Node endpoint so uploads work immediately.

```bash
cd server
npm install
node server.js          # listens on port 3000
```

Find your computer's LAN IP (e.g. `192.168.1.20`), then in the app's **Upload server URL** field enter:

```
http://192.168.1.20:3000/upload
```

Recordings + metadata land in `server/uploads/` (`calls.json` holds the metadata log). Replace this with your real backend later — the app POSTs `multipart/form-data` with a `file` part plus `number`, `direction`, `startTime`, `durationSec`, `recordId`.

> The sample server is **plain HTTP** for local testing. A real server should use **HTTPS**; otherwise add a network-security config to allow cleartext to your host.

## How detection works

`CallReceiver` listens for `PHONE_STATE`. `CallStateTracker` interprets transitions:

- `IDLE → RINGING → OFFHOOK` = **incoming**, answered → start recording
- `IDLE → OFFHOOK` = **outgoing** → start recording
- `… → IDLE` = call ended → stop, save row, enqueue upload

The outgoing number isn't delivered in the broadcast on Android 9+, so the service reads the newest entry from the system **call log** (needs the `READ_CALL_LOG` permission) to fill it in.

## Things you'll likely want to add

- A real authenticated HTTPS backend (add a token header in `UploadWorker`).
- A "delete after upload" toggle to save space.
- Auto-start after reboot (`BOOT_COMPLETED` receiver) so it survives restarts.
- A built-in player to listen to recordings in-app.
- Encryption of files at rest.

## Project layout

```
ACRApp/
├── app/src/main/AndroidManifest.xml
├── app/src/main/java/com/acr/recorder/
│   ├── App.kt, MainActivity.kt
│   ├── receiver/   CallReceiver.kt, CallStateTracker.kt
│   ├── service/    CallRecordingService.kt
│   ├── recorder/   CallAudioRecorder.kt
│   ├── data/       CallRecord.kt, CallRecordDao.kt, AppDatabase.kt, Prefs.kt
│   ├── upload/      UploadWorker.kt
│   └── ui/          CallLogScreen.kt
└── server/         server.js, package.json   (sample receiver)
```
