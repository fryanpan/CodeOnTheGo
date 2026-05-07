# Building the XKCD Plugin: A Code on the Go Plugin Tutorial

This README is a step-by-step walkthrough of how the `random-xkcd-plugin`
module was built, starting from Code on the Go's bundled
**CodeOnTheGoPlugin** project template. The goal is that a developer who
has never written a CoGo plugin can read this front to back, follow along
in their own copy, and end up with a working plugin while learning every
plugin-specific concept the framework exposes.

Each step has the same shape:

- **What we add** — code, a manifest entry, a resource.
- **Why** — the plugin or sandbox concept the step teaches.
- **What you observe** — the behavior you should see after building and
  installing.

The polish — clipboard support, disk caching, FileProvider routing,
triple-tap detection — isn't required for *your* plugin. We keep all of
it here because each piece is the curriculum for one of the trickier
parts of the plugin sandbox model. Skip the steps that don't apply when
you're working on a different feature.

---

## What you'll build

A plugin that adds an **"XKCD"** tab to Code on the Go's editor bottom
sheet. Tapping the tab fetches a new random xkcd comic and renders it.
Double-tap copies the comic's URL to the clipboard; triple-tap copies
the comic image. The plugin works offline if a comic was previously
cached.

## What you'll learn

Plugin-specific concepts only — this README assumes you already know
Kotlin, Android Activities/Fragments, Gradle, and `kotlinx.coroutines`.

- The `IPlugin` lifecycle (`initialize` → `activate` → `deactivate` → `dispose`)
  and which method does what
- The `UIExtension` surface — bottom-sheet tabs, sidebar items, main
  editor tabs — and how each one differs
- Why plugin Fragments must wrap their inflater with
  `PluginFragmentHelper.getPluginInflater(...)` or crash with
  `Resources$NotFoundException`
- How plugin permissions work, how the sandbox enforces them, and what
  to declare for network/filesystem/clipboard access
- How a plugin reaches the host's filesystem (`Context.cacheDir`,
  `Context.filesDir`) without violating the sandbox
- Why plugin `<provider>` manifest entries are dead code, and how to
  route through the host IDE's `FileProvider` authority instead
- The `DocumentationExtension` three-tier tooltip and the Tier 3
  HTML walkthrough served at `http://localhost:6174/plugin/<id>/...`

## Prerequisites

- A working Code on the Go dev environment. Follow the team's developer
  onboarding wiki page if you haven't already; the rest of this tutorial
  assumes `flox activate -d flox/local` puts you in the right shell and
  `./gradlew :app:assembleDebug` builds the host IDE.
- The `random-xkcd-plugin/` source open alongside this README. You will
  refer back to specific files frequently — every step in this tutorial
  points at code that already exists in the module.
- Java/Kotlin/Android dev experience. No prior plugin experience needed.

---

## Step 1 — Create the plugin from the template

Inside Code on the Go on a device or emulator, choose
**New Project → CodeOnTheGoPlugin**. Fill in:

- **App name** — anything human-readable. We'll override it later.
- **Package name** — anything unique, e.g. `com.example.demoplugin`.
- **Author** — your name.
- **Include Sample Code** — leave this checked. The sample is what we
  customize step-by-step below.

The wizard expands the template into a new project and drops you into
its source. Open it and read each generated file — that's the starting
point this tutorial customizes.

**What the template generates** (this is the project the rest of this
tutorial transforms into the XKCD plugin):

| File | What it does |
|---|---|
| `build.gradle.kts` | Standard Android library config + the `com.itsaky.androidide.plugins.build` Gradle plugin that converts the APK into a `.cgp` (Code Go Plugin) bundle. |
| `src/main/AndroidManifest.xml` | Plugin metadata as `<meta-data>` entries: `plugin.id`, `plugin.name`, `plugin.permissions`, `plugin.main_class`, etc. *Not* a normal Android app manifest — no Activities, no Services. |
| `src/main/kotlin/<package>/<ClassName>.kt` | Plugin entry class. Implements `IPlugin` for lifecycle and (with sample code on) `UIExtension`, `EditorTabExtension`, and `DocumentationExtension`. |
| `src/main/kotlin/<package>/fragments/<ClassName>Fragment.kt` | Sample fragment with a status label and an action button; long-pressing the button shows the plugin's tooltip. |
| `src/main/res/layout/fragment_main.xml` | Layout for the sample fragment. |
| `src/main/res/values/strings.xml` | Plugin's string resources. |
| `src/main/res/drawable/ic_plugin.xml` | Sidebar icon. |
| `libs/plugin-api.jar` | Compile-only JAR exposing `IPlugin`, `PluginContext`, all the extension interfaces, and the host service interfaces (`IdeEditorTabService`, `IdeTooltipService`, …). |

**Build and install the unmodified template:**

```bash
cd <new-plugin-dir>
./gradlew assemblePluginDebug
```

This produces `build/plugin/<plugin-name>-debug.cgp`. Sideload that file
into Code on the Go (Settings → Plugin Manager → install from file).
After install:

- A new entry appears in the **left sidebar** (the template registers a
  `NavigationItem` via `getSideMenuItems()`).
- Tapping it opens a **main editor tab** with the sample fragment (the
  template registers an `EditorTabItem` via `getMainEditorTabs()`).
- Long-pressing the action button in the fragment shows the plugin's
  tooltip (the template registers a `PluginTooltipEntry` via
  `getTooltipEntries()`).

**What you just learned.** The template is showing you three different
UI surfaces on purpose. There are *four* UI surfaces total a plugin can
register:

| Surface | Extension method | What it shows |
|---|---|---|
| **Side menu** | `UIExtension.getSideMenuItems(): List<NavigationItem>` | Entries in the left sidebar. Tapping them runs an `action: () -> Unit`. |
| **Main editor tab** | `EditorTabExtension.getMainEditorTabs(): List<EditorTabItem>` | A full-screen editor tab — same area where source files open. |
| **Editor bottom sheet** | `UIExtension.getEditorTabs(): List<TabItem>` | A tab in the bottom sheet next to "Build Output", "App Logs", etc. |
| **Main menu** | `UIExtension.getMainMenuItems(): List<MenuItem>` | Entries in the IDE's overflow menu. |

The XKCD plugin only uses the **bottom-sheet** surface. We'll delete
the sidebar/main-editor-tab wiring in Step 2 and replace it with a
single `getEditorTabs()` registration.

---

## Step 2 — Make it XKCD: rename and reshape

We now transform the template into the XKCD plugin. Compare your
generated template to the files under `random-xkcd-plugin/` as you
read each substep.

### 2a. Rename the package and classes

Pick `com.codeonthego.xkcdrandom` (or your own equivalent) and update:

- `build.gradle.kts` — `namespace`, `applicationId`, and `pluginName` in
  the `pluginBuilder { … }` block. The string passed to `pluginName`
  becomes the prefix of the produced `.cgp` filename
  (`random-xkcd-debug.cgp`).
- `AndroidManifest.xml` — `plugin.id`, `plugin.name`, `plugin.description`,
  and **especially** `plugin.main_class`. The host loads the plugin by
  reflectively instantiating exactly the class named in `plugin.main_class`,
  via `DexClassLoader`. If this string doesn't match a real class, the
  plugin fails to load with a silent `ClassNotFoundException` in logcat.
  Also remove the template's `<meta-data android:name="plugin.sidebar_items" />`
  entry — we're not registering a sidebar item.
- The `XkcdRandomPlugin.kt` source file's package declaration.

> **Plugin concept — `plugin.id` is the namespace key for everything
> else.** The string in `plugin.id` shows up as the directory in
> `Context.filesDir`/`Context.cacheDir` the plugin sees, the path
> segment in the Tier 3 doc URL, and the key the host uses internally
> to track the plugin. Pick once, never change.

### 2b. Drop to one extension surface

The template's plugin class implements three extension interfaces:
`UIExtension`, `EditorTabExtension`, `DocumentationExtension`. The XKCD
plugin only needs two — `UIExtension` (for the bottom-sheet tab) and
`DocumentationExtension` (for the tooltip + walkthrough).

Replace the class header with:

```kotlin
class XkcdRandomPlugin : IPlugin, UIExtension, DocumentationExtension {
    // ...
}
```

Now look at `random-xkcd-plugin/src/main/kotlin/com/codeonthego/xkcdrandom/XkcdRandomPlugin.kt`.
The class implements only the two interfaces above and registers a
single bottom-sheet tab:

```kotlin
override fun getEditorTabs(): List<TabItem> = listOf(
    TabItem(
        id = TAB_ID,                          // "xkcd_bottom_tab"
        title = "XKCD",
        fragmentFactory = { XkcdPanelFragment() },
        order = 200
    )
)
```

> **Plugin concept — `fragmentFactory` returns a fresh instance every
> time.** The host calls it whenever the bottom sheet is asked to
> show your tab. Never return a cached singleton — Fragment lifecycle
> expectations require a clean instance each time.

> **Plugin concept — `order` is relative to other plugins, not built-in
> tabs.** Built-in tabs (Build Output, App Logs, Run, …) keep fixed
> slots. `order` only sorts plugin-supplied tabs against each other.

### 2c. The IPlugin lifecycle

Read the four lifecycle overrides in `XkcdRandomPlugin.kt`:

```kotlin
override fun initialize(context: PluginContext): Boolean { … }
override fun activate(): Boolean { … }
override fun deactivate(): Boolean { … }
override fun dispose() { … }
```

| Method | When the host calls it | What you do |
|---|---|---|
| `initialize(ctx)` | Once, just after the host loads the plugin's classes. Returns `Boolean` — `false` here means the host skips `activate()` entirely. | Stash the `PluginContext` (used for logging, services, resources). Wrap the body in `try/catch` so a stray exception in your setup doesn't crash the host. |
| `activate()` | After `initialize()` returns true. | Light it up — start any background work, register listeners. The XKCD plugin doesn't need to do anything here. |
| `deactivate()` | When the user disables the plugin (or the IDE is shutting down). | Tear down what you started in `activate()`. |
| `dispose()` | Final teardown after `deactivate()`. | Release any heavy resources you hold. |

> **Plugin concept — `PluginContext` is your handle to the host.** It
> exposes a `logger` (writes to logcat with a per-plugin tag), a
> `services` registry (look up `IdeEditorTabService`, `IdeTooltipService`,
> etc.), and access to the host's resources / asset paths. Stash the
> reference in `initialize` and use it from every other lifecycle method.

### 2d. Build and install again

```bash
./gradlew assemblePluginDebug
```

Sideload. After install you should see:

- The sidebar entry from the template is **gone** (we dropped
  `EditorTabExtension` and the `getSideMenuItems()` override).
- An **"XKCD" tab appears in the editor bottom sheet** alongside Build
  Output, App Logs, etc.
- Tapping the tab opens an empty Fragment (we haven't added behavior yet).

If the tab doesn't appear, check logcat for `ClassNotFoundException` —
that means `plugin.main_class` in the manifest doesn't match the
fully-qualified class name. If the tab appears but tapping it crashes,
check logcat for `Resources$NotFoundException` — that means the
Fragment's inflater isn't wrapped (we cover that next).

---

## Step 3 — The Fragment, with the inflater quirk

Open `src/main/kotlin/com/codeonthego/xkcdrandom/fragments/XkcdPanelFragment.kt`.
Most of it is normal Android — `onCreateView`, `onViewCreated`,
`findViewById` for each view binding. The plugin-specific piece is at
the top:

```kotlin
override fun onGetLayoutInflater(savedInstanceState: Bundle?): LayoutInflater {
    val inflater = super.onGetLayoutInflater(savedInstanceState)
    return PluginFragmentHelper.getPluginInflater(XkcdRandomPlugin.PLUGIN_ID, inflater)
}
```

> **Plugin concept — plugin Fragments must wrap their LayoutInflater.**
> Plugins are loaded via `DexClassLoader`. The `LayoutInflater` the
> host passes to your Fragment defaults to resolving `R.layout.*`
> against the **host IDE's** resources, not yours. If you skip this
> override, the inflater can't find your layout XML and you crash with
> `android.content.res.Resources$NotFoundException` the first time the
> Fragment shows.

The fix is `PluginFragmentHelper.getPluginInflater(pluginId, parent)` —
it wraps the parent inflater with a `Resources` instance backed by your
plugin's APK, so `R.layout.fragment_xkcd_panel`, `R.id.xkcd_root`, etc.
all resolve.

### What you observe after this step

The XKCD tab now opens a real layout from
`res/layout/fragment_xkcd_panel.xml` — currently a blank panel with a
legend and "no comic cached yet" text. We're ready to add behavior.

---

## Step 4 — The data class and the network client

We need a typed model for an xkcd comic, and an HTTP client to fetch
one. Both are plain Kotlin — no plugin-specific concepts here, but the
**permissions** discussion at the end of this step is critical.

### 4a. The data class

`src/main/kotlin/com/codeonthego/xkcdrandom/net/XkcdComic.kt`:

```kotlin
data class XkcdComic(
    val num: Int,
    val title: String,
    val alt: String,
    val imageUrl: String,
) {
    val pageUrl: String get() = "https://xkcd.com/$num/"
}
```

### 4b. The API client

`src/main/kotlin/com/codeonthego/xkcdrandom/net/XkcdApiClient.kt`. Two
endpoints, no auth:

- `GET https://xkcd.com/info.0.json` — latest comic
- `GET https://xkcd.com/<num>/info.0.json` — specific comic

Read the file end-to-end. It's intentionally tiny: OkHttp +
`org.json.JSONObject` (the one bundled with Android), no Retrofit, no
Moshi. The whole network surface is < 100 lines and reads top to bottom.

The `fetchRandom()` method picks a number in `[1, latestNum]`, retries
exactly once if it lands on 404 (xkcd's page-not-found comic returns
HTTP 404 on its JSON endpoint), and gives up gracefully. Returning
`null` rather than throwing keeps the Fragment's empty-state branch
reachable on a flaky network.

The `build.gradle.kts` already pulls OkHttp in:

```kotlin
implementation("com.squareup.okhttp3:okhttp:4.12.0")
```

### 4c. Wire the Fragment to the API

Look at `loadRandomComic()` and `fetchDecodeAndCache()` in
`XkcdPanelFragment.kt`. Two interesting bits:

```kotlin
private fun loadRandomComic() {
    if (loadJob?.isActive == true) return            // skip rapid-tap fan-out
    if (currentComic == null) showLoading()          // avoid blanking on warm start
    loadJob = viewLifecycleOwner.lifecycleScope.launch {
        val result = withContext(Dispatchers.IO) { fetchDecodeAndCache() }
        // … render or show empty state
    }
}
```

```kotlin
private suspend fun fetchDecodeAndCache(): Pair<XkcdComic, Bitmap>? {
    val comic = api.fetchRandom() ?: return null
    val bytes = api.openImageStream(comic.imageUrl)?.use { stream ->
        // bounded read with cooperative cancellation
        // …
    } ?: return null
    cache.save(comic, bytes)
    val bmp = BitmapFactory.decodeByteArray(bytes, 0, bytes.size) ?: return null
    return comic to bmp
}
```

The fetch + image decode both run on `Dispatchers.IO`. Decoding a 2 MB
PNG on the main thread will drop frames on a low-end device.

### 4d. **Sandbox lesson — declare `network.access`**

The xkcd plugin's `AndroidManifest.xml` includes:

```xml
<meta-data
    android:name="plugin.permissions"
    android:value="network.access,filesystem.read" />
```

Permissions are comma-separated values inside one `<meta-data>` entry —
*not* the `<uses-permission>` system Android apps use. The available
plugin permissions are:

| Permission | Unlocks |
|---|---|
| `network.access` | `java.net.URL`, `OkHttpClient`, anything that hits the network. The host's `PluginSecurityManager` enforces this on plugin-thread network calls. |
| `filesystem.read` | Read access to the plugin's `cacheDir` / `filesDir`, and to host paths the plugin is allowed to see. |
| `filesystem.write` | Symmetric write access. |
| `system.commands` | `Runtime.exec`, process spawning. |
| `ide.settings` | Read/write IDE preferences. |
| `project.structure` | Walk the user's open project. |

> **Plugin concept — what happens if you forget `network.access`.**
> The first network call from the plugin throws a `SecurityException`
> at runtime. `getJson()` only catches `IOException`, so the exception
> propagates up the coroutine and shows up in logcat as an unhandled
> error. The host doesn't show a "permission denied" toast — it just
> refuses the syscall. Always declare every permission you actually
> use, and only those.

### What you observe after this step

Tapping the XKCD tab fetches a random comic and renders it. If you
toggle airplane mode and reopen the tab, you see the offline empty
state instead of a crash.

---

## Step 5 — Cache the last comic on disk

Open `src/main/kotlin/com/codeonthego/xkcdrandom/cache/XkcdDiskCache.kt`.
A "last comic only" cache: one JSON metadata file + one PNG, both under
`context.cacheDir/xkcd/`.

```kotlin
class XkcdDiskCache(context: Context) {
    private val dir: File = File(context.cacheDir, DIR_NAME).apply { mkdirs() }
    private val metaFile = File(dir, "last.json")
    private val imageFile = File(dir, "last.png")

    fun save(comic: XkcdComic, pngBytes: ByteArray) {
        if (pngBytes.size > MAX_IMAGE_BYTES) return    // bounded
        metaFile.writeText(toJson(comic))
        imageFile.writeBytes(pngBytes)
    }

    companion object {
        private const val DIR_NAME = "xkcd"
        const val MAX_IMAGE_BYTES = 5 * 1024 * 1024    // 5 MB
    }
}
```

The Fragment renders from the cache on first show, then kicks off a
fresh fetch:

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    // …
    renderFromCache()
    if (savedInstanceState == null) loadRandomComic()
}
```

> **Plugin concept — plugins reach the filesystem only through the
> host's `Context`.** Plugin code receives a `Context` from the host
> (passed into the Fragment, available via `requireContext()`).
> `cacheDir` and `filesDir` on that `Context` resolve to per-plugin
> sandbox directories the host manages. **Don't hard-code paths.**
> If you do, the sandbox refuses the syscall and you crash with
> `SecurityException`.

> **Plugin concept — `cacheDir` vs `filesDir`.** Use `cacheDir` for
> regenerable data (this comic — we can re-download). The host (and
> Android) is free to evict it under storage pressure. Use `filesDir`
> for data the user expects to persist (in this plugin, we use it
> briefly in Step 7 for the FileProvider hop, but the cached comic
> itself lives in `cacheDir`).

> **Plugin concept — bound your reads.** `MAX_IMAGE_BYTES = 5 MB` caps
> both the network read (`fetchDecodeAndCache` aborts mid-stream) and
> the disk cache (`save` rejects oversized inputs). Without the cap, a
> pathological response could fill the cache dir or OOM the decoder.
> This is one of the recurring code-review themes for CoGo plugins.

### What you observe after this step

Open the XKCD tab, fetch a comic, then close and reopen the tab. The
previous comic appears immediately (from cache), then a new fetch kicks
off. Toggle airplane mode and reopen: the cached comic still renders
instead of the empty state.

---

## Step 6 — Copy the URL on double-tap

The double-tap behavior is plain Android — `ClipboardManager`,
`ClipData.newPlainText`. Look at `copyUrlToClipboard()` in the Fragment:

```kotlin
private fun copyUrlToClipboard() {
    val comic = currentComic ?: return
    val cm = requireContext().getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
    cm.setPrimaryClip(ClipData.newPlainText("xkcd-url", comic.pageUrl))
    toast(getString(R.string.toast_url_copied, comic.pageUrl))
}
```

> **Plugin concept — clipboard write goes through the host's
> `Context`.** `getSystemService(CLIPBOARD_SERVICE)` returns the host
> IDE's `ClipboardManager`, which calls into the platform clipboard.
> No plugin-specific permission gate sits in the way *for plain text*.
> Image copy is the harder path — see Step 7.

We don't dispatch this directly from the touch listener. Instead, we
feed every tap through a `TapCountClassifier` (Step 8) and route the
classifier's result:

```kotlin
private fun handleClassification(c: TapCountClassifier.Classification?) {
    if (!isAdded || view == null) return
    when (c) {
        Classification.SINGLE -> loadRandomComic()
        Classification.DOUBLE -> copyUrlToClipboard()
        Classification.TRIPLE -> copyImageToClipboard()
        null -> { /* nothing to do */ }
    }
}
```

### What you observe after this step

Double-tap the XKCD tab → toast appears, clipboard contains the comic's
xkcd.com URL. Paste it into any text field to confirm.

---

## Step 7 — Copy the image on triple-tap (the sandbox curriculum)

This is the most plugin-specific step in the tutorial. Read
`copyImageToClipboard()` in `XkcdPanelFragment.kt` carefully — most of
the design notes below sit in comments next to that function.

```kotlin
private fun copyImageToClipboard() {
    val source = cache.loadImageFile() ?: run {
        toast(getString(R.string.toast_image_copy_failed)); return
    }
    val ctx = requireContext()
    viewLifecycleOwner.lifecycleScope.launch {
        val target = withContext(Dispatchers.IO) {
            val shareDir = File(ctx.filesDir, "xkcd_share").apply { mkdirs() }
            val out = File(shareDir, "last.png")
            try { source.copyTo(out, overwrite = true); out } catch (_: Exception) { null }
        }
        if (target == null) { toast(/* … */); return@launch }
        val authority = "${ctx.packageName}.providers.fileprovider"
        val uri = FileProvider.getUriForFile(ctx, authority, target)
        val clip = ClipData.newUri(ctx.contentResolver, "xkcd-image", uri)
        val cm = ctx.getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
        cm.setPrimaryClip(clip)
        toast(getString(R.string.toast_image_copied))
    }
}
```

There are three plugin-specific concepts at work here.

### 7a. Why we don't register our own `<provider>`

In a normal Android app, sharing an image to the clipboard means
declaring a `<provider>` in the manifest with a `FileProvider` class
and a `file_provider_paths.xml`. **That doesn't work for plugins.**

> **Plugin concept — `<provider>` declarations in a plugin manifest are
> dead code.** Plugins are loaded via `DexClassLoader`, not installed
> as Android apps. Android's `PackageManager` never sees the plugin's
> manifest, so it never registers your `<provider>`, `<activity>`, or
> `<service>`. Anything that requires registration with the OS won't
> work — only plugin-API surfaces (extension interfaces, host services)
> are wired up at runtime.

The escape valve: **route through the host IDE's `FileProvider`
authority.** The host (Code on the Go) ships its own `FileProvider`
with authority `${packageName}.providers.fileprovider` and a
`file_provider_paths.xml` that exposes `filesDir` via
`<files-path name="files" path="." />`. Any file we drop under
`ctx.filesDir/...` can be served from that authority.

```kotlin
val authority = "${ctx.packageName}.providers.fileprovider"
val uri = FileProvider.getUriForFile(ctx, authority, target)
```

`ctx.packageName` is the host's package name (because `ctx` is the
host's `Context`), so this composes to the right authority without
hard-coding it.

### 7b. Why we copy through `filesDir`, not `cacheDir`

`file_provider_paths.xml` exposes `filesDir`, not `cacheDir`. If we
hand `FileProvider.getUriForFile` a path under `cacheDir`, it throws
`IllegalArgumentException: Failed to find configured root that contains
…`. So we copy the cached PNG into `filesDir/xkcd_share/last.png` first,
then derive the content URI from there.

The copy is up to 5 MB — keep it off the main thread (`withContext(Dispatchers.IO)`).

### 7c. Why `ClipData.newUri` (not `newPlainText`)

`ClipData.newUri(resolver, label, uri)` queries the `ContentResolver`
for the URI's MIME type. Because our path ends in `.png` and the host's
`FileProvider` resolves that to `image/png`, the resulting clip
advertises `image/*` to paste targets. Paste into Messages, Gmail, any
image-aware app — it pastes the image, not the URL string.

### What you observe after this step

Triple-tap the XKCD tab → toast says "Comic image copied to clipboard".
Open Messages, long-press the input, choose Paste — the comic image
appears as an attachment.

---

## Step 8 — The tap classifier (optional)

This step covers `src/main/kotlin/com/codeonthego/xkcdrandom/ui/TapCountClassifier.kt`
and the related touch-listener wiring in `XkcdPanelFragment`. **It's
generic Android UX, not a plugin-specific concept.** Skip this section
if you don't need triple-tap detection in your own plugin — it's
included here because the ticket called for it, and because it's a
reusable state machine you can lift wholesale.

### Why a custom classifier

`android.view.GestureDetector` resolves single tap and double tap, but
not triple. Adding triple on top of GestureDetector requires hand-rolled
state on a parallel timeline, at which point a 25-line state machine is
clearer than a hybrid.

`TapCountClassifier` is intentionally pure (no clocks, no `Handler`,
no Android imports). The Fragment supplies `now` and decides when to
call `resolve()`, which makes the classifier unit-testable in plain
JUnit — see `src/test/kotlin/.../TapCountClassifierTest.kt`.

### How the Fragment uses it

```kotlin
private fun handleTap() {
    val now = SystemClock.uptimeMillis()
    val burstClosedEarly = tapClassifier.onTap(now)
    if (burstClosedEarly) {
        // Triple-tap → resolve immediately for snappy feedback
        mainHandler.removeCallbacks(resolveBurstRunnable)
        handleClassification(tapClassifier.resolve())
        return
    }
    // Otherwise wait one window for more taps — re-arm timeout each tap
    mainHandler.removeCallbacks(resolveBurstRunnable)
    mainHandler.postDelayed(resolveBurstRunnable, TapCountClassifier.DEFAULT_WINDOW_MS)
}
```

### Scroll-vs-tap

The XKCD panel is a `ScrollView`, so every fling/scroll ends in
`ACTION_UP`. To keep scroll-end from being classified as a tap, the
listener tracks `ACTION_DOWN` coordinates and only feeds the classifier
if the finger moved less than the system touch slop:

```kotlin
val touchSlop = ViewConfiguration.get(view.context).scaledTouchSlop
root.setOnTouchListener { _, event ->
    when (event.actionMasked) {
        MotionEvent.ACTION_DOWN -> { downX = event.x; downY = event.y }
        MotionEvent.ACTION_UP -> {
            val dx = event.x - downX; val dy = event.y - downY
            if (dx * dx + dy * dy <= touchSlop * touchSlop) {
                handleTap(); root.performClick()
            }
        }
    }
    false  // never consume — let the ScrollView keep scrolling
}
```

Returning `false` is important — consuming the event would break scroll
behavior on tall comics.

### What you observe after this step

Single-tap → new comic. Double-tap → URL copied. Triple-tap → image
copied. Long scrolls don't trigger any of the three.

---

## Step 9 — The documentation surface

The `DocumentationExtension` interface gives a plugin three tiers of
in-IDE documentation. Look at the bottom of `XkcdRandomPlugin.kt`.

### 9a. Tooltip entries

```kotlin
override fun getTooltipCategory(): String = "plugin_xkcd"

override fun getTooltipEntries(): List<PluginTooltipEntry> = listOf(
    PluginTooltipEntry(
        tag = TOOLTIP_TAG_TAB,                          // "xkcd.tab"
        summary = "Random xkcd comic. Tap to roll a new one.",
        detail = """
            <p>This panel pulls a random comic from <b>xkcd.com</b>.</p>
            <ul>
              <li><b>Tap</b> — fetch a new random comic.</li>
              <li><b>Double-tap</b> — copy the comic's URL to the clipboard.</li>
              <li><b>Triple-tap</b> — copy the comic image to the clipboard.</li>
            </ul>
        """.trimIndent(),
        buttons = listOf(
            PluginTooltipButton(
                description = "Code walkthrough",
                uri = "index.html",
                order = 0
            )
        )
    )
)
```

| Tier | Field | What the user sees |
|---|---|---|
| 1 | `summary` | Short string shown when the user long-presses your tab/button. |
| 2 | `detail` | HTML rendered inside the tooltip when the user taps "See More". |
| 3 | `buttons[].uri` | A button labeled `description`. Tapping it opens an HTML page served by the host at `http://localhost:6174/plugin/<plugin.id>/<uri>`. |

The XKCD plugin's tab itself is the anchor — the host calls
`tooltipService.showTooltip(anchorView, category = "plugin_xkcd",
tag = "xkcd.tab")` automatically when the user long-presses the tab.
For other anchors (a button, an icon), you'd call the tooltip service
yourself; the template's sample fragment shows that pattern.

### 9b. Tier 3 asset bundling

```kotlin
override fun getTier3DocsAssetPath(): String? = "docs"
```

This says: *"My Tier 3 walkthrough lives under `src/main/assets/docs/`."*

At plugin install time the host's `Tier3AssetWalker` indexes everything
under that directory and serves each file at:

```
http://localhost:6174/plugin/<plugin.id>/<relative-path>
```

For this plugin: `http://localhost:6174/plugin/com.codeonthego.xkcdrandom/index.html`,
`/css/walkthrough.css`, etc. Files reference each other with relative
paths — so the HTML's `<link rel="stylesheet" href="css/walkthrough.css">`
just works.

The `uri` in the `PluginTooltipButton` is relative to the asset path,
so `uri = "index.html"` resolves to the URL above.

### What you observe after this step

Long-press the **XKCD** tab title in the bottom sheet:

1. The summary chip appears: "Random xkcd comic. Tap to roll a new one."
2. Tap "See More" → the detail HTML expands.
3. Tap "Code walkthrough" → the IDE opens
   `http://localhost:6174/plugin/com.codeonthego.xkcdrandom/index.html`
   in an in-IDE WebView. Read the full walkthrough; the asset bundle
   includes its own CSS.

---

## Step 10 — Build, install, and verify end-to-end

### Build

```bash
cd random-xkcd-plugin
./gradlew assemblePluginDebug
```

The `com.itsaky.androidide.plugins.build` Gradle plugin runs as part
of `assemblePluginDebug` — it takes the produced APK and renames it
to `build/plugin/random-xkcd-debug.cgp`. **Caveat:** if you run
`assemblePluginDebug` twice in a row without cleaning, the second run
can produce a stub `.cgp` because the rename step deletes its source
APK on success but the second run still finds Gradle's intermediates
"up to date". Always:

```bash
./gradlew clean assemblePluginDebug
```

when you actually want to produce a fresh artifact for sideloading.

### Run unit tests

```bash
./gradlew testDebugUnitTest
```

Confirms `TapCountClassifierTest` passes. The classifier is the only
file we test at the unit level — the rest is mostly UI glue, which we
exercise via the Kaspresso instrumented tests under
`app/src/androidTest/kotlin/com/itsaky/androidide/plugins/xkcd/`.

### Install and verify

Sideload `build/plugin/random-xkcd-debug.cgp` via Settings → Plugin
Manager → install from file. Then verify each behavior:

- [ ] **Tab appears.** "XKCD" shows up in the editor bottom sheet next
  to "Build Output" / "App Logs".
- [ ] **Cold-start fetch.** Open the tab → spinner → comic renders.
- [ ] **Single-tap fetches a new comic.** Tap the panel → spinner →
  different comic.
- [ ] **Double-tap copies the URL.** Toast says "URL copied: …" — paste
  into a text field to confirm.
- [ ] **Triple-tap copies the image.** Toast says "Comic image copied
  to clipboard" — paste into Messages to confirm.
- [ ] **Offline fallback works.** Toggle airplane mode, kill the IDE,
  reopen. The XKCD tab shows the last cached comic. Tap to attempt a
  fetch → toast says "Could not load comic — check your connection".
- [ ] **Tooltip three tiers work.** Long-press the tab title, see the
  summary; "See More" reveals the detail HTML; "Code walkthrough"
  opens the Tier 3 asset page.

If any of these fail, check logcat for the specific exceptions called
out in the relevant step above.

---

## Permissions reference

Every permission this plugin declares, what it unlocks, and what code
exercises it.

| Permission | Why we declare it | Code that uses it | What you'd see without it |
|---|---|---|---|
| `network.access` | Fetch xkcd JSON + image | `XkcdApiClient.fetchLatest`, `fetchByNumber`, `openImageStream` | `SecurityException` on the OkHttp call → empty state, stack trace in logcat |
| `filesystem.read` | Read the disk cache on cold start | `XkcdDiskCache.loadComic`, `loadImageFile`; `XkcdPanelFragment.renderFromCache` | Cold-start panel always shows empty state, even when a previous fetch wrote the cache |

Other permissions to know about (not used by this plugin):

| Permission | When you'd want it |
|---|---|
| `filesystem.write` | Persist user data outside `cacheDir` (which is always writable). The xkcd plugin only writes to `cacheDir` and a `filesDir/xkcd_share` scratch path the host's `FileProvider` already permits; if you write *user* data, declare this. |
| `system.commands` | `Runtime.exec` or any process spawn. |
| `ide.settings` | Read/write the IDE's preferences (e.g., theme settings). |
| `project.structure` | Walk the user's open project. |

> **Heuristic.** Declare every permission you actually use. Don't
> declare anything you don't — the user sees the permission list and
> a plugin that asks for `system.commands` "just in case" looks
> hostile.

---

## The sandbox model in one screen

Plugins run inside the host IDE's process, but the plugin classes are
loaded by a **`DexClassLoader`** that's separate from the host's class
loader. This has three durable consequences:

1. **No plugin manifest registration.** The OS never sees the plugin's
   `AndroidManifest.xml`, so `<activity>`, `<service>`, `<provider>`,
   and `<receiver>` declarations are dead code. Anything that requires
   OS-side registration must go through the host.

2. **`R.*` resolves against whichever `Resources` your code is using.**
   The host's `LayoutInflater` resolves to host resources by default.
   Wrap with `PluginFragmentHelper.getPluginInflater(pluginId, parent)`
   to flip it to your plugin's APK.

3. **Filesystem access is mediated through the host's `Context`.**
   `cacheDir` and `filesDir` resolve to per-plugin sandbox directories.
   Hard-coded paths (`/sdcard/...`) get rejected by the security
   manager.

The standard escape valves:

- For UI surfaces — register an extension (`UIExtension`,
  `EditorTabExtension`, `DocumentationExtension`).
- For services (clipboard, project structure, editor tabs) — fetch via
  `context.services.get(IdeXxxService::class.java)` or
  `requireContext().getSystemService(...)`.
- For file sharing across apps — copy under `ctx.filesDir` and use
  `FileProvider.getUriForFile(ctx, "${ctx.packageName}.providers.fileprovider", file)`.

When in doubt: if it requires an OS registration that the host hasn't
already done for you, it doesn't work. Find the host service or
extension surface that maps to your need.

---

## Where to go next

- **Plugin Development Guide (wiki)** — the full API reference for
  every extension interface and host service.
- **`plugin-api/src/main/kotlin/com/itsaky/androidide/plugins/extensions/`**
  in the CodeOnTheGo source — every available extension surface, in
  one folder. Read the source if the wiki lags reality.
- **Sibling demo plugins** —
  - `forms-plugin` shows form-style input handling.
  - `maps-plugin` shows a more complex view with persistent state.
  Both use the same scaffolding as this one and are good reference
  reads when your plugin grows beyond what fits in a tab.
- **`assets/docs/index.html`** in this module — a denser code
  walkthrough served at runtime via the Tier 3 docs system. Long-press
  the XKCD tab and tap "Code walkthrough" to read it inside the IDE.
- **Static plugin checker** (when it lands in `appdevforall/scripts/plugin-check/`)
  — catches the dead-code patterns called out in Step 7 (plugin-side
  `<activity>` / `<provider>` declarations, missing
  `PluginFragmentHelper` wraps) before the plugin reaches a device.
