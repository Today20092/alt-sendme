# Peer-app Android share URI handling

Research date: 2026-07-24

## Conclusion

Copying an Android share-sheet `content://` URI into app-private staging is
normal, but it is not an Android requirement. The receiver may instead stream
from `ContentResolver` while its URI grant remains valid. The choice depends on
the transfer engine and lifecycle:

- **Stage a private copy** when the existing engine requires filesystem paths,
  the user may send later, or the activity can finish before transfer.
- **Stream directly** when the engine accepts streams/file descriptors and the
  app can keep the URI grant valid for the complete transfer.

For Alt SendMe today, staging each incoming URI in a private batch directory is
the smallest reliable design because its shared send pipeline accepts local
paths. This is an Android adapter concern, not a cross-platform protocol
feature. The common layer should receive a list of sendable items; each platform
turns its native share references into items the existing transfer engine can
read.

## What Android requires

Android delivers binary shares as `content://` URIs in `EXTRA_STREAM`: one URI
for `ACTION_SEND`, and an `ArrayList<Uri>` for `ACTION_SEND_MULTIPLE`. The
receiver reads them through `ContentResolver`. Android grants URI-scoped read
access rather than transferring file bytes in the intent.

The grant is temporary. Android documents that a receiver can access a provider
URI under the grant while the receiving activity is active. Consequently,
direct streaming is valid for an immediate transfer, but retaining only the URI
for deferred/background work is fragile unless the provider and intent support
a persistable grant. A normal share intent should not be assumed to provide
one.

Android does not prescribe a cache copy. It provides both stream and file
descriptor APIs (`openInputStream` and `openFileDescriptor`), leaving lifecycle
management to the receiver.

Sources:

- [Android: receive simple data from other apps](https://developer.android.com/training/sharing/receive#handling-content)
- [Android: send binary and multiple content](https://developer.android.com/training/sharing/send#send-binary-content)
- [Android: temporary content-provider permissions](https://developer.android.com/guide/topics/providers/content-provider-basics#Permissions)
- [`ContentResolver.openInputStream`](https://developer.android.com/reference/android/content/ContentResolver#openInputStream(android.net.Uri))
- [`ContentResolver.openFileDescriptor`](https://developer.android.com/reference/android/content/ContentResolver#openFileDescriptor(android.net.Uri,%20java.lang.String))

## LocalSend: verified implementation

LocalSend registers both `ACTION_SEND` and `ACTION_SEND_MULTIPLE` with `*/*`, so
Android can offer it for one or many files of any advertised MIME type.

[LocalSend manifest at inspected commit](https://github.com/localsend/localsend/blob/c92866b5be7f7834b0b604661f7db3ca027dd37e/app/android/app/src/main/AndroidManifest.xml#L55-L67)

For cold start it asks `share_handler` for the initial payload; for warm start
it listens to the plugin stream. It converts all returned attachments into the
same sending-file model and opens the Send tab.

[LocalSend share lifecycle](https://github.com/localsend/localsend/blob/c92866b5be7f7834b0b604661f7db3ca027dd37e/app/lib/config/init.dart#L257-L320)

### Share sheet: copied to private cache

LocalSend's `share_handler` dependency reads `EXTRA_STREAM` for both actions.
On Android, its path conversion opens a `content://` URI with
`ContentResolver.openInputStream` and copies the bytes to a file under
`context.cacheDir`. The plugin returns that cached filesystem path to
LocalSend. LocalSend deliberately avoids clearing the cache on startup when an
initial share exists because otherwise “the shared file will be lost.”

Therefore the currently verifiable LocalSend share-sheet behavior is **private
cache staging**, not end-to-end streaming from the original provider URI.

Sources:

- [`share_handler`: one and multiple `EXTRA_STREAM` inputs](https://github.com/ShoutSocial/share_handler/blob/a5e4ec8d8b385b084ea99bf70fe0f8167fd9b8a6/share_handler_android/android/src/main/kotlin/com/shoutsocial/share_handler/ShareHandlerPlugin.kt#L204-L219)
- [`share_handler`: URI conversion and cache result](https://github.com/ShoutSocial/share_handler/blob/a5e4ec8d8b385b084ea99bf70fe0f8167fd9b8a6/share_handler_android/android/src/main/kotlin/com/shoutsocial/share_handler/ShareHandlerPlugin.kt#L222-L269)
- [`share_handler`: `openInputStream` copied into `context.cacheDir`](https://github.com/ShoutSocial/share_handler/blob/a5e4ec8d8b385b084ea99bf70fe0f8167fd9b8a6/share_handler_android/android/src/main/kotlin/com/shoutsocial/share_handler/FileDirectory.kt#L85-L119)
- [LocalSend cache-preservation comment](https://github.com/localsend/localsend/blob/c92866b5be7f7834b0b604661f7db3ca027dd37e/app/lib/config/init.dart#L283-L288)

### In-app picker: direct URI/file-descriptor path

This must not be confused with LocalSend's newer in-app Android file picker.
LocalSend's changelog says picker-selected files are no longer copied to cache.
Its native bridge validates a `content://` URI and opens a read-only
`ParcelFileDescriptor`, detaching the descriptor for its native/Rust consumer.
That is direct provider-backed I/O and is possible because this newer sending
pipeline was designed to accept a descriptor rather than only a pathname.

Sources:

- [LocalSend changelog: no picker cache copy](https://github.com/localsend/localsend/blob/c92866b5be7f7834b0b604661f7db3ca027dd37e/app/assets/CHANGELOG.md#L115-L121)
- [LocalSend native content-URI file descriptor bridge](https://github.com/localsend/localsend/blob/c92866b5be7f7834b0b604661f7db3ca027dd37e/app/android/app/src/main/kotlin/org/localsend/localsend_app/MainActivity.kt#L91-L119)

## Blip: implementation not publicly verifiable

Blip's public site describes Android file transfer and its user-facing
features, but no official public Android source repository or technical
documentation was found that discloses how it consumes share-sheet URIs.
Therefore it is not responsible to claim that Blip copies or streams. Its
behavior can be measured experimentally, but that would still not prove its
internal architecture.

Source:

- [Blip official site](https://blip.net/)

## Recommended Alt SendMe architecture

### Now

1. The Android adapter accepts `ACTION_SEND` and `ACTION_SEND_MULTIPLE` with
   `*/*`.
2. It extracts every `EXTRA_STREAM` URI, using `ClipData` only as a
   compatibility fallback.
3. Off the UI thread, it queries display name/size where available and copies
   each URI through `ContentResolver` into one private batch directory.
4. It passes the resulting path list into the existing multi-file Send-tab
   flow.
5. It retains the batch until transfer/cancellation, then removes it; startup
   cleanup removes abandoned old batches.

The stable cached path may also make retry or resume behavior easier because
the original Android URI grant can expire. That is a plausible benefit, not a
verified explanation for Alt SendMe's original design; its resumability path
must be traced separately before treating cache retention as a requirement.

This supports images, video, audio, documents, archives, and mixed file
selections without broad storage permission. MIME is descriptive metadata, not
the basis for refusing an otherwise readable file. Folders remain separate
because Android's standard share contract transfers content items, not portable
directory trees.

### Later, only if staging becomes a measured problem

The cleaner long-term cross-platform boundary is a send source that can be
either a local path or a platform-owned readable handle/stream. Android can
then open a `ParcelFileDescriptor` from `ContentResolver`; Apple platforms can
use their security-scoped/share-extension mechanisms; desktop keeps ordinary
paths. The network protocol remains unchanged because all sources ultimately
produce the same byte stream and metadata.

Do not build that abstraction for this bug unless cache duplication is proven
to cause unacceptable storage, latency, or very-large-file failures. LocalSend
itself demonstrates both designs: cache staging for share-sheet intake and
descriptor streaming where its newer picker pipeline supports it.
