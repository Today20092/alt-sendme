# Android image share target research

Research date: 2026-07-24

## Conclusion

The correct Android contract is to register the app's activity for
`ACTION_SEND` with `image/*` (and `ACTION_SEND_MULTIPLE` if multiple images are
supported), receive the shared `content://` URI, and consume it through
`ContentResolver` under the sender's temporary URI grant. Do not treat the URI
as a filesystem path and do not request broad photo/storage permission.

For current Tauri 2, the smallest platform-aligned implementation is to use
`bundle.fileAssociations` with `androidIntentActionFilters` and handle
`RunEvent::Opened` for both cold and warm launches. Only add a custom Android
plugin if the project's installed Tauri version cannot deliver share intents
through `RunEvent::Opened` or testing shows that it loses metadata the send
flow needs.

## Android platform contract

### Manifest registration

An exported activity that receives shares needs matching intent filters:

```xml
<intent-filter>
    <action android:name="android.intent.action.SEND" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:mimeType="image/*" />
</intent-filter>

<intent-filter>
    <action android:name="android.intent.action.SEND_MULTIPLE" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:mimeType="image/*" />
</intent-filter>
```

Android's receiving guide uses exactly this shape and says the selected
activity is started with the incoming intent. It discourages `*/*` unless the
app can truly handle every content type. An activity with intent filters must
be exported so other apps can start it.

Sources:

- [Receive simple data from other apps](https://developer.android.com/training/sharing/receive)
- [`<activity>` manifest element](https://developer.android.com/guide/topics/manifest/activity-element)

### Reading the payload

For one image, inspect `Intent.action == ACTION_SEND`, validate that the MIME
type starts with `image/`, and read the `Uri` from `Intent.EXTRA_STREAM`. For
multiple images, use `ACTION_SEND_MULTIPLE` and the parcelable URI list in
`EXTRA_STREAM`. Android's current examples use `IntentCompat` for API-safe
parcelable extraction.

The sender-side Android contract likewise places a binary `content://` URI in
`EXTRA_STREAM`; multiple shares place an `ArrayList<Uri>` there. `ClipData`
may also carry URI grant information, but it is not a replacement for handling
the documented `EXTRA_STREAM` payload. A robust native adapter may accept a
URI from `clipData` as a fallback for nonconforming senders, while keeping
`EXTRA_STREAM` as the primary contract.

Sources:

- [Receive simple data: handle incoming content](https://developer.android.com/training/sharing/receive#handling-content)
- [Send simple data: binary and multiple content](https://developer.android.com/training/sharing/send)
- [`Intent.EXTRA_STREAM`](https://developer.android.com/reference/android/content/Intent#EXTRA_STREAM)

### URI access and permissions

The received value is normally a `content://` URI owned by Gallery, Photos, or
another provider. Open it with `ContentResolver.openInputStream`,
`openFileDescriptor`, or another resolver API. Never derive or require a
`_data` filesystem path.

The sending app grants access to that specific URI with
`FLAG_GRANT_READ_URI_PERMISSION`. Android documents this as temporary,
URI-scoped access lasting while the receiving activity is active. Therefore:

- no `READ_MEDIA_IMAGES` or legacy external-storage permission is required to
  consume a properly granted share URI;
- validate action, MIME type, URI scheme, count, and size because incoming
  intents are untrusted;
- perform stream/file I/O off the main thread;
- if sending continues after the activity can finish, copy the stream promptly
  into app-owned cache/staging storage, then pass that local file through the
  existing send pipeline.

Sources:

- [Content provider basics: temporary URI permissions](https://developer.android.com/guide/topics/providers/content-provider-basics#Intents)
- [Receive simple data: validate and process off the UI thread](https://developer.android.com/training/sharing/receive#handling-content)
- [`ContentResolver`](https://developer.android.com/reference/android/content/ContentResolver)

## Lifecycle and task behavior

Both entry paths must work:

1. **Cold start:** read the activity's initial intent during creation/startup.
2. **Warm start:** if Android reuses the activity, process the new intent from
   `onNewIntent`.

`singleTop` sends a new intent to `onNewIntent` only when the activity is
already at the top. `singleTask` also reuses an existing instance, but clears
activities above it and changes task behavior. Android cautions that most apps
should retain default task behavior unless there is a demonstrated need.
Consequently, launch mode should not be changed merely to fix payload parsing;
the implementation must handle both initial and new intents under the launch
mode Tauri generates.

Sources:

- [Tasks and the back stack: launch modes](https://developer.android.com/guide/components/activities/tasks-and-back-stack#TaskLaunchModes)
- [`Activity.onNewIntent`](https://developer.android.com/reference/android/app/Activity#onNewIntent(android.content.Intent))

## Tauri 2 implementation choice

Tauri's official mobile file-association support generates Android intent
filters from `bundle.fileAssociations`. Its Android action options are `Send`,
`SendMultiple`, and `View`. Tauri delivers opened file URLs as
`RunEvent::Opened` on Android and explicitly requires handling two cases:
runtime delivery when already running, and startup delivery before the
frontend is ready.

The recommended bridge is therefore:

1. Declare the image MIME/extensions and Android `Send` action in
   `bundle.fileAssociations`; include `SendMultiple` only if the UI supports it.
2. In Rust, capture every `RunEvent::Opened { urls }`.
3. Store cold-start URLs in managed state until the webview asks for them.
4. Emit the same URLs to the frontend for warm-start events.
5. Route both paths into one existing “selected files”/send-state function.
6. If downstream code needs ordinary paths, resolve/copy `content://` input
   into app-owned staging storage before invoking that function.

Official Tauri documentation also exposes a custom Android plugin
`onNewIntent(intent)` hook and plugin events. That is the fallback when the
installed Tauri release predates mobile file associations, or if device tests
show `RunEvent::Opened` does not preserve the required share payload.

Sources:

- [Tauri: File Associations on Mobile](https://v2.tauri.app/learn/mobile-file-associations/)
- [Tauri: Mobile Plugin Development — lifecycle and `onNewIntent`](https://v2.tauri.app/develop/plugins/develop-mobile/#onnewintent)

## Verification matrix

Test on a physical device with logcat for each path:

- app not running → share one JPEG/PNG from Gallery;
- app foregrounded → share another image;
- app backgrounded → share another image;
- share from both the OEM Gallery and Google Photos/files provider;
- reject a non-image or inaccessible URI cleanly;
- if supported, share multiple images;
- begin a send after leaving the share activity to prove staging survives the
  temporary grant's lifetime.

The pass condition is that the Send tab contains the shared image without a
second picker interaction, and the same ingestion function is used for cold
and warm launches.
