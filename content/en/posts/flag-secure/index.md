---
title: "Behind the Screen: Detecting and Preventing Screenshots in Android"
summary: "Screenshot security in Android is more complicated than it looks, not just FLAG_SECURE"
categories: ["Post","Blog",]
tags: ["compose","security","screenshot"]
#externalUrl: ""
showSummary: true
date: 2025-12-05
---

### Screenshot security in Android is more complicated than it looks.

Most developers begin with the obvious solution — `FLAG_SECURE` — and quickly discover:

- It hurts UX  
- It prevents debugging and QA  
- It blocks Recents previews  
- Support cannot reproduce or visualize issues  
- Users hate when they can’t screenshot their own data  

This is why modern apps use a hybrid approach:

- Screenshot detection  
- Recents-screen protection  
- Per-screen security (Jetpack Compose)  
- `FLAG_SECURE` only where truly necessary  

This post covers everything from **Android 10 → Android 14+**.

---

## 1. Why `FLAG_SECURE` Works — and Why It Hurts UX & Debugging

Android provides a simple, powerful way to block screenshots:

```kt
window.setFlags(
    WindowManager.LayoutParams.FLAG_SECURE,
    WindowManager.LayoutParams.FLAG_SECURE
)
```

When enabled, Android blocks:

* Screenshots
* Screen recordings
* Casting / mirroring
* Recents screen previews

It’s instant and OS-level.

> But the downside is huge:

No screenshots for:

* QA bug reports
* UX/UI review
* Developer debugging
* Customer support
* Users sharing receipts, queue numbers, or results

So instead of blocking everything globally, a better strategy is **detection + selective blocking**.

---

## 2. Screenshot Detection + Notify

Sometimes you don't want to block screenshots—you just want to:

* Warn the user
* Mask sensitive UI
* Log the event
* Trigger additional security steps

Android provides two ways to detect screenshots.

---

### 2.1 Android 14+ — Official Screenshot Detection API

```kt
class MainActivity : ComponentActivity() {
    val TAG = this.javaClass.name
    val screenCaptureCallback =
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            Activity.ScreenCaptureCallback {
                Toast.makeText(this, "Screen captured", Toast.LENGTH_SHORT).show()
            }
        } else null

    override fun onStart() {
        super.onStart()
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            registerScreenCaptureCallback(
                mainExecutor,
                screenCaptureCallback as Activity.ScreenCaptureCallback
            )
        }
    }

    override fun onPause() {
        super.onPause()
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            unregisterScreenCaptureCallback(screenCaptureCallback as Activity.ScreenCaptureCallback)
        }
    }
}
```

Add to `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.DETECT_SCREEN_CAPTURE" />
```

---

### 2.2 Android 10–13 — MediaStore Observer Detection

```kt
class ScreenshotDetector(private val context: Context) {
    private val observer = object : ContentObserver(Handler(Looper.getMainLooper())) {
        override fun onChange(selfChange: Boolean, uri: Uri?) {
            Toast.makeText(context, "Screenshot detected", Toast.LENGTH_SHORT).show()
        }
    }

    fun start() {
        context.contentResolver.registerContentObserver(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            true,
            observer
        )
    }

    fun stop() {
        context.contentResolver.unregisterContentObserver(observer)
    }
}
```

Use in an Activity:

```kt
override fun onResume() {
    super.onResume()
    scr.start()
}

override fun onPause() {
    super.onPause()
    scr.stop()
}
```

Minimum version: **Android 10 (API 29)**

---

## 3. Blocking Recents-Screen Preview

### 3.1 Android 13+ — Official API

```kt
if (Build.VERSION.SDK_INT >= 33) {
    setRecentsScreenshotEnabled(false)
}
```

What this does:

* Hides your activity’s thumbnail in Recents

What it does NOT do:

* Does not block normal screenshots
* Does not stop screen recording

Great for protecting sensitive content without hurting UX.

---

### 3.2 Android 10–12 — Workarounds

#### 3.2.1 Activity Lifecycle Approach

```kt
override fun onPause() {
    window.addFlags(FLAG_SECURE)
}

override fun onResume() {
    window.clearFlags(FLAG_SECURE)
}
```

#### Result:

* Works when pressing **Home**
* Does *not* work reliably when pressing **Recents** (especially Samsung)

---

#### 3.2.2 Using `onWindowFocusChanged`

```kt
override fun onWindowFocusChanged(hasFocus: Boolean) {
    if (hasFocus) window.clearFlags(FLAG_SECURE)
    else window.addFlags(FLAG_SECURE)
}
```

This fails because:

* Dialog steals focus
* Keyboard steals focus
* Notifications steal focus
* Permission dialogs steal focus
* Multi-window mode
* System UI interactions

Your activity is still visible but loses focus → flicker and broken UX.

---

### 3.3 Android 10+ — Better Approach: `onTopResumedActivityChanged()`

```kt
override fun onTopResumedActivityChanged(isTopResumedActivity: Boolean) {
    super.onTopResumedActivityChanged(isTopResumedActivity)

    if (isTopResumedActivity) {
        window.clearFlags(WindowManager.LayoutParams.FLAG_SECURE)
    } else {
        window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)
    }
}
```

### Why it’s better:

* Not triggered by dialogs
* Not affected by keyboard
* Not affected by system UI
* Only fires when entering/exiting foreground

### Limitations:

* Only Android 10+
* Toggling `FLAG_SECURE` may still flicker
* Split-screen behavior varies per OEM

---

## 4. Per-Screen Security in Jetpack Compose

```kt
@Composable
fun SecureScreenProtection() {
    val activity = LocalActivity.current
    val allowScreenshot = LocalAllowScreenshot.current
    DisposableEffect(Unit) {
        if (!allowScreenshotForDev()) {
            activity?.window?.setFlags(
                WindowManager.LayoutParams.FLAG_SECURE,
                WindowManager.LayoutParams.FLAG_SECURE
            )
        }

        onDispose {
            if (allowScreenshot) {
                activity?.window?.clearFlags(WindowManager.LayoutParams.FLAG_SECURE)
            }
        }
    }
}

val LocalAllowScreenshot = compositionLocalOf<Boolean> { false }

private fun allowScreenshotForDev(): Boolean {
    return true // or BuildConfig.DEBUG
}
```

Apply at app level:

```kt
CompositionLocalProvider(
    LocalAllowScreenshot provides isScreenshotAllowed,
) {
    // YourApp()
}
```

This allows selective protection without harming UX or debugging.

---

## 5. Summary

| Goal                                | Best Approach                        |
| ----------------------------------- | ------------------------------------ |
| Block screenshots completely        | `FLAG_SECURE`                        |
| Block Recents preview (Android 13+) | `setRecentsScreenshotEnabled(false)` |
| Block Recents preview (Android 10+) | `onTopResumedActivityChanged()`      |
| Detect screenshots (Android 10–13)  | MediaStore Observer                  |
| Detect screenshots (Android 14+)    | `ScreenCaptureCallback`              |
| Per-screen protection               | Compose `SecureScreenProtection()`   |

---
