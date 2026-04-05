# Extension Component Lifecycle

Understanding the lifecycle is critical for writing extensions that behave correctly in both the
companion app (live testing) and built APKs (production).

---

## The two runtime environments

### Companion app (live testing)

The MIT AI2 Companion (and Kodular Companion, etc.) is a pre-built Android app that loads
extensions dynamically at runtime. Here's what happens:

1. **User connects** the companion to the IDE via WiFi or USB
2. **Assets and extensions are pushed** from the IDE to the device via JSON-over-TCP
3. **Extension DEX is loaded** using `DexClassLoader`:
   - The companion copies the extension's `classes.jar` (which contains DEX bytecode) to a
     temp directory on device storage
   - A `DexClassLoader` is created with the path to the DEX file
   - The extension class is loaded by name from the DEX
4. **Component is instantiated** — the constructor receives a `ComponentContainer` (the Form)
5. **Designer properties are applied** — the IDE sends property values set in the designer
6. **User interacts with the app** — blocks execute, calling methods and firing events

**Key implications for extension developers:**
- Your extension is loaded into a classloader that is a *child* of the companion's own classloader
- If your extension bundles a library that the companion already has, the parent classloader's
  version wins (delegation model). This can cause version mismatch issues.
- After changing and reimporting an extension, the user **must restart the companion** — the old
  class is still loaded in the previous classloader instance
- Static fields persist across companion sessions within the same companion launch
- If your extension references a class that doesn't exist in the companion's classpath or your
  DEX, you get `ClassNotFoundException` at runtime

### Built APK (production)

When the user builds an APK via "Build > App (provide QR code for .apk)":

1. **All component DEX files are merged** into the app's DEX (or multidex)
2. **Extension assets** are copied into the APK's `assets/` directory
3. **Permissions, activities, services, receivers** from `component_build_infos.json` are merged
   into the AndroidManifest.xml
4. **The app starts** like any Android app — your extension class is on the normal classpath
5. **Component is instantiated** in the same way as in the companion

**Key implications:**
- No dynamic classloading — your classes are in the main DEX
- Library version conflicts manifest differently: DEX merge errors at build time instead of
  runtime `ClassNotFoundException`
- Static fields are fresh on each app launch
- Native libraries from `libs/` in the AIX go to the APK's `lib/` directories

---

## Form (Activity) lifecycle hooks

Extensions can participate in the Android Activity lifecycle by implementing these interfaces.
The Form dispatches lifecycle events to all registered components.

### OnResumeListener / OnPauseListener / OnStopListener / OnDestroyListener

```java
import com.google.appinventor.components.runtime.OnResumeListener;
import com.google.appinventor.components.runtime.OnPauseListener;
import com.google.appinventor.components.runtime.OnStopListener;
import com.google.appinventor.components.runtime.OnDestroyListener;

@SimpleObject(external = true)
public class MyExtension extends AndroidNonvisibleComponent
        implements OnResumeListener, OnPauseListener, OnStopListener, OnDestroyListener {

    public MyExtension(ComponentContainer container) {
        super(container.$form());
        // Register for lifecycle events
        form.registerForOnResume(this);
        form.registerForOnPause(this);
        form.registerForOnStop(this);
        form.registerForOnDestroy(this);
    }

    @Override
    public void onResume() {
        // Activity came to foreground
        // Restart sensors, reconnect, etc.
    }

    @Override
    public void onPause() {
        // Activity going to background
        // Pause sensors, save state, etc.
    }

    @Override
    public void onStop() {
        // Activity no longer visible
    }

    @Override
    public void onDestroy() {
        // Activity being destroyed
        // Clean up resources, unregister receivers, etc.
    }
}
```

### OnNewIntentListener

For handling new intents (e.g., NFC tag scanned, deep link opened):

```java
implements OnNewIntentListener {
    // register in constructor:
    form.registerForOnNewIntent(this);

    @Override
    public void onNewIntent(Intent intent) {
        // Handle the new intent
    }
}
```

### ActivityResultListener

For receiving results from started activities:

```java
implements ActivityResultListener {
    private int requestCode;

    // In constructor or method:
    requestCode = form.registerForActivityResult(this);

    // Start an activity:
    form.startActivityForResult(intent, requestCode);

    @Override
    public void resultReturned(int requestCode, int resultCode, Intent data) {
        if (requestCode == this.requestCode) {
            // Handle the result
        }
    }
}
```

---

## Component initialization order

1. The Form (Screen) is created as an Activity
2. Non-visible components are instantiated in the order they appear in the `.scm` file
3. Constructor runs → `super(container.$form())` stores the form reference
4. Designer properties are set (calling the setter methods)
5. The `Screen.Initialize` event fires
6. User-defined event handlers start running

This means: **don't rely on other components being initialized in your constructor.** If you need
to interact with another component, do it in response to events or methods, not during construction.

---

## Multi-screen behavior

Each Screen in App Inventor is a separate Activity. When the user switches screens:

1. The current Activity may be paused or stopped (depending on the screen switch method)
2. A new Activity is created for the new screen
3. Components on the new screen are instantiated fresh
4. Components on the old screen receive `onPause`/`onStop`/`onDestroy` if registered

**Important:** If your extension uses the component on only one screen, it only exists on that
screen's Activity. If the user puts it on multiple screens, each screen gets its own instance.

Static state (static fields) is shared across all screens within the same app process.

---

## Threading model

App Inventor runs on Android's standard threading model:
- **UI thread**: All block execution, property setting, event dispatching
- **Background threads**: Created explicitly by extensions for async work

**Rules:**
- All `@SimpleFunction`, `@SimpleProperty`, `@SimpleEvent` methods are called on the UI thread
- Never do blocking I/O on the UI thread — use `AsynchUtil.runAsynchronously()`
- To post results back to UI thread: `form.runOnUiThread(runnable)`
- Events dispatched from background threads will crash — always dispatch from UI thread
