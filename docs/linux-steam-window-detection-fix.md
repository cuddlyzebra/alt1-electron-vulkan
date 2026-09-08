# Fix: RuneScape window not detected under Steam/Proton on Linux

## Summary

On Linux, when RuneScape is launched as a non-Steam shortcut added to Steam (e.g. to get
GPU-accelerated Proton rendering for the Jagex Launcher / RS client), the native capture
layer failed to recognize the game window at all. The app would start, connect to the X
server's RECORD extension successfully, then immediately report `no rs instance found` and
shut its reader threads back down — even though the correct window was on screen and had a
fresh, valid X window ID.

The root cause was in `IsRsWindowProperties` in `os_x11_linux.cc`: window matching required
an exact `WM_CLASS` match against a short fixed list, including a single hardcoded Steam
AppID (`steam_app_1343400`) for the official RuneScape Steam listing. Non-Steam shortcuts
don't get that AppID — Steam assigns each one its own synthetic, per-install `steam_app_<id>`
class name instead — so any setup running RuneScape via a manually-added Steam shortcut could
never match, no matter how correctly it was configured.

The fix relaxes the classname check: once the window title is confirmed to start with
`RuneScape`, any `steam_app_*` classname is accepted, not just the one hardcoded ID.

## Symptoms

```
native: X record extension version: 1.13
no rs instance found
native: record thread exiting
native: window thread exiting
```

- Build and dependency install completed cleanly.
- Electron launched and the debugger attached without issue.
- The native module successfully connected to X and the RECORD extension.
- Despite this, the reader immediately failed to find an RS window and tore down its threads.

## Investigation

1. **Ruled out a stale window ID.** Confirmed the `--window-id` passed in was captured fresh
   (via `wmctrl`/`xdotool`) immediately before the run, not left over from an earlier session —
   so this wasn't simple ID staleness.
2. **Inspected the window's actual X properties.** Ran `xprop -id <window-id>` while RuneScape
   was running at that ID, checking `WM_CLASS` and `WM_NAME`/`_NET_WM_NAME` — the two properties
   the native reader uses to identify an RS window.
3. **Traced the matching logic.** Followed the match down into `IsRsWindowProperties` in
   `os_x11_linux.cc`, which checks the window title against the literal string `RuneScape`,
   then checks the window class against a fixed list of known class names.
4. **Found the mismatch.** The window's title matched RuneScape correctly, but its `WM_CLASS`
   was a Steam-generated `steam_app_<id>` string specific to that local Steam shortcut —
   not `steam_app_1343400`, and not any of the other hardcoded values in the class list.

## Root cause

The original `IsRsWindowProperties` matched title and classname together, requiring an exact
hit against the fixed `rsClassNames` list — which contains exactly one Steam AppID:

```cpp
static constexpr std::array<std::string_view, 5> rsClassNames = {
	rsName,
	"steam_app_1343400",   // <- only the official Steam listing's AppID
	"rs2client.exe",
	protonName,
	"steam_app_default",
};

bool IsRsWindowProperties(std::string title, std::string classname)
{
	if (title == "")
		return false;
	auto it = std::find_if(rsClassNames.begin(), rsClassNames.end(), [&](std::string_view s) {
		return (title.compare(0, strlen(rsName), rsName) == 0) && (s == classname);
	});
	return it != rsClassNames.end();
}
```

That covers RuneScape launched via Steam's official listing, but not RuneScape added as a
**non-Steam shortcut** — which Proton users commonly do specifically to get Vulkan/GPU
acceleration for the Jagex Launcher on Linux. Steam assigns those shortcuts their own
per-install `steam_app_<id>`, which will never equal `1343400`, so the exact-match `find_if`
never succeeds and the window is silently rejected.

## Fix

`IsRsWindowProperties` now separates the title check from the classname check, and falls back
to accepting *any* `steam_app_*` classname once the title is already confirmed to be
RuneScape:

```diff
 bool IsRsWindowProperties(std::string title, std::string classname)
 {
 	if (title == "")
 		return false;
-	auto it = std::find_if(rsClassNames.begin(), rsClassNames.end(), [&](std::string_view s) {
-		return (title.compare(0, strlen(rsName), rsName) == 0) && (s == classname);
-	});
-	return it != rsClassNames.end();
+	if (title.compare(0, strlen(rsName), rsName) != 0)
+		return false;
+
+	// Fast path: known exact classnames (native client, official Steam listing, etc.)
+	auto it = std::find_if(rsClassNames.begin(), rsClassNames.end(), [&](std::string_view s) {
+		return s == classname;
+	});
+	if (it != rsClassNames.end())
+		return true;
+
+	// Fallback: any Steam-wrapped window. Non-Steam shortcuts (e.g. the Jagex Launcher added
+	// manually to Steam for GPU-accelerated Proton rendering) get a synthetic per-install
+	// steam_app_<id> classname instead of RuneScape's official steam_app_1343400, so a single
+	// hardcoded ID can't cover every user. The title check above already confirms this is
+	// actually RuneScape, so it's safe to accept any steam_app_* class here.
+	static constexpr std::string_view steamPrefix = "steam_app_";
+	if (classname.size() >= steamPrefix.size() &&
+		classname.compare(0, steamPrefix.size(), steamPrefix) == 0) {
+		return true;
+	}
+	return false;
 }
```

The title check is what keeps this safe — it guarantees any `steam_app_*` window accepted
by the new fallback is actually running RuneScape, not just any Steam-wrapped app.

## Unrelated changes in the same commit

The working copy also included additions to the SHM capture path (`shm.cc`/`shm.h`) —
`dumpRaw()` plus `debugWidth()`/`debugHeight()`/`debugDepth()` helpers, and a
one-shot debug dump to `/tmp/alt1_capture_debug.raw` called from `OSCaptureMulti` in
`os_x11_linux.cc`. These are debug/instrumentation additions for capture-path work
(e.g. verifying Vulkan capture output) and are unrelated to the window-detection fix above;
they weren't part of this investigation and aren't detailed further here.

## How to verify you're hitting this issue

1. Confirm the reader is starting and connecting to X (you'll see `native: X record extension
   version: ...` in the log) but immediately reports `no rs instance found`.
2. Get your RS window's ID: `wmctrl -l` or `xdotool search --name "RuneScape"`.
3. Run `xprop -id <window-id>` and check `WM_CLASS`.
4. If `WM_CLASS` looks like `steam_app_<some-number-other-than-1343400>`, you're hitting
   exactly this issue — RuneScape was added to Steam as a non-Steam shortcut, and the old
   hardcoded AppID check doesn't recognize your install's generated class name.
