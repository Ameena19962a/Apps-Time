# TimeAnchor — native app shell

This wraps the [TimeAnchor](../timeanchor.html) prototype as a real, installable app using
[Capacitor](https://capacitorjs.com/): the same HTML/CSS/JS, plus:

- **Persistent storage** — your schedule is saved on-device (`localStorage`) and survives closing the app.
- **Real reminders** — scheduled OS-level notifications when a block starts, when it's 5 minutes from ending,
  and when its time is up. These fire even if the app is backgrounded or fully closed, and always track the
  real clock (the on-screen 1x/10x/60x speed control is a testing aid only — it never changes when a real
  reminder fires).
- A day's blocks act as a repeating template: marking things "done" resets automatically at the next real day.
- **Optional accounts + cloud sync** — sign in and the same timeline follows you across devices. Off by default
  (the app runs fully local-only until configured); see "Cloud accounts" below to turn it on.

## Cloud accounts (optional)

Off by default. To turn it on:

1. Create a free [Supabase](https://supabase.com) project (no card required).
2. In its **SQL Editor**, run:

   ```sql
   create table if not exists public.timeanchor_state (
     user_id uuid primary key references auth.users(id) on delete cascade,
     data jsonb not null,
     updated_at timestamptz not null default now()
   );

   alter table public.timeanchor_state enable row level security;

   create policy "Users can view their own state"
     on public.timeanchor_state for select using (auth.uid() = user_id);
   create policy "Users can insert their own state"
     on public.timeanchor_state for insert with check (auth.uid() = user_id);
   create policy "Users can update their own state"
     on public.timeanchor_state for update using (auth.uid() = user_id);
   ```

3. In **Project Settings → API**, copy the **Project URL** and the **anon / public** key (never the
   `service_role` key — that one must stay server-side only, and this app has no server).
4. In `www/index.html`, find `SUPABASE_URL` and `SUPABASE_ANON_KEY` near the top of the `<script>` block and
   replace the two placeholder strings with those values.
5. `npx cap sync android` (and `ios`, once you have that platform added) to copy the updated web assets into
   the native project, then rebuild.

With those two values filled in: launch shows a sign-in screen (email + password, with a "continue without an
account" escape hatch for local-only use); each account's categories/blocks/settings are stored as one JSON row
per user in `timeanchor_state`, protected by the row-level security policies above so a user can only ever read
or write their own row. Local `localStorage` is still kept as a fast cache and offline fallback — cloud saves
are debounced (~700ms after the last change) rather than firing on every keystroke.

**This only works with real network access** — the native app, or a normally-hosted copy of this page. It will
not work inside a sandboxed preview (like a claude.ai Artifact) that blocks outside network requests by design.

## What's already done

- `www/index.html` — the app itself (no build step; it's a static page Capacitor loads into a native WebView).
- `android/` — a real, generated Android Studio project with the [`@capacitor/local-notifications`](https://capacitorjs.com/docs/apis/local-notifications)
  plugin wired in.
- `package.json` / `capacitor.config.json` — Capacitor project config (`appId: com.timeanchor.audhd`).

**I could not produce a built `.apk`/`.aab` or an iOS project from this environment.** Two hard blockers, not
just missing config:

1. This sandbox's network policy blocks `dl.google.com`, which is where the Android SDK platform/build-tools
   come from — Gradle can't finish a build without them, no matter how it's configured.
2. iOS builds require Xcode, which only runs on macOS. There is no path to an iOS build from a Linux sandbox
   regardless of network access — you'll need a Mac for that step.

Everything below is what to run **on your own machine** to get from this scaffold to an installed app.

## Android — build & install

Requires [Android Studio](https://developer.android.com/studio) (which bundles the SDK) or the command-line
SDK tools, plus Node.js.

```bash
cd app
npm install
npx cap sync android
npx cap open android   # opens android/ in Android Studio
```

In Android Studio: let Gradle sync finish, then **Run ▶** with a device or emulator selected. That's a debug
build installed straight to the device — no store account needed to try it yourself.

To build an installable APK from the command line instead (once the SDK is set up so `ANDROID_HOME` is set
and licenses are accepted):

```bash
cd app/android
./gradlew assembleDebug
# APK lands at android/app/build/outputs/apk/debug/app-debug.apk
```

### Publishing to the Play Store

1. Generate a signing key (`keytool -genkey -v -keystore timeanchor.keystore ...`) and configure it in
   `android/app/build.gradle` under `signingConfigs` — Android Studio's **Build > Generate Signed Bundle**
   wizard does this for you interactively.
2. Build a release **AAB** (`./gradlew bundleRelease`), not a debug APK.
3. Create a Google Play Console account ($25 one-time) and a new app listing, then upload the AAB.
4. You'll need a privacy policy URL, a feature graphic, screenshots, and an app icon (see below).

## iOS — build & install (needs a Mac)

```bash
cd app
npm install
npx cap add ios      # generates the ios/ project — must be run on macOS with Xcode + CocoaPods installed
npx cap sync ios
npx cap open ios      # opens ios/App/App.xcworkspace in Xcode
```

In Xcode: select your Apple ID under Signing & Capabilities, pick a device or simulator, and Run.

### Publishing to the App Store

1. Enroll in the [Apple Developer Program](https://developer.apple.com/programs/) ($99/yr).
2. In Xcode, **Product > Archive**, then use the Organizer to upload to App Store Connect.
3. Create the App Store Connect listing: screenshots, description, privacy policy URL, app icon, and answer
   the App Privacy questionnaire (this app collects nothing — everything is local to the device).
4. Submit for review.

## App icon / splash screen

Capacitor ships placeholder icons. Once you have a real icon (1024×1024 PNG) and want proper sizes generated
for both platforms:

```bash
npm install -D @capacitor/assets
npx capacitor-assets generate --iconBackgroundColor '#f6f4ef' --splashBackgroundColor '#f6f4ef'
```

## Notification permissions

- **Android 13+** requires the runtime `POST_NOTIFICATIONS` permission, which `@capacitor/local-notifications`
  already requests for you (`initNotifications()` in `www/index.html` calls this on launch).
- **iOS** will show its own native permission prompt the first time notifications are requested — no extra
  code needed.
- If permission is denied, the app still works — it just shows a one-line, dismissible notice explaining that
  reminders are off, per the app's no-alarm, no-punitive-copy design.

## Keeping the web layer and the standalone prototype in sync

`../timeanchor.html` (repo root) is the plain browser demo — in-memory only, no notifications, meant for quick
sharing/testing in a tab. `app/www/index.html` is the same UI with persistence and notification scheduling
layered on for the real app. They're intentionally two files: changes to the core timeline/timer/reflow logic
should generally be made in both, but the storage/notification code only belongs in `app/www/index.html`.
