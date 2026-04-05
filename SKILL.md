---
name: appinventor-extension
description: >
  Build App Inventor / Kodular extensions (.aix files) from scratch. Use this skill whenever the user
  wants to create, debug, fix, or understand an App Inventor extension — including writing Java or
  Kotlin component code, setting up a Rush or FAST project, understanding the AIX package format,
  handling external dependencies, dealing with ProGuard/R8, or troubleshooting companion-vs-built-app
  differences. Also trigger when the user mentions "extension", "AIX", ".aix", "SimpleFunction",
  "SimpleEvent", "SimpleProperty", "DesignerComponent", "AndroidNonvisibleComponent", "Rush", "FAST",
  "fast-cli", "rush-cli", "App Inventor component", "Kodular extension", "helper blocks", "dropdown
  blocks", or asks about the extension lifecycle, simple_components.json, or the annotation processor
  pipeline. If someone asks you to "make an extension that does X" for App Inventor or Kodular, this
  is the skill to use.
---

# App Inventor Extension Development

This skill enables an AI agent to write, build, debug, and package App Inventor extensions (AIX files).
It covers the full stack: annotation-driven component API, build toolchains (Rush, FAST), AIX internals,
companion vs. built-app lifecycle, dependency management, and common idioms.

## Quick orientation

An App Inventor extension is a Java or Kotlin class that extends `AndroidNonvisibleComponent` (or, for
visible extensions on supported forks, `AndroidViewComponent`), annotated with `@SimpleObject(external = true)`
and `@DesignerComponent(category = ComponentCategory.EXTENSION, ...)`. The annotation processor generates
metadata (`simple_components.json`) that the IDE and runtime consume. The build toolchain compiles the
source, runs d8/R8 to produce DEX bytecode, and packages everything into an AIX (a ZIP with a specific
directory layout).

**Before writing any extension code, read `references/annotations.md`** for the full annotation API.
**Before dealing with build or packaging issues, read `references/aix-format.md`** for AIX internals.
**Before dealing with dependencies, read `references/dependencies.md`**.

---

## Extension class skeleton

### Non-visible extension (most common)

```java
package com.example.myextension;

import com.google.appinventor.components.annotations.*;
import com.google.appinventor.components.common.ComponentCategory;
import com.google.appinventor.components.runtime.*;
import com.google.appinventor.components.runtime.util.*;

@DesignerComponent(
    version = 1,
    description = "Short user-facing description shown in the designer",
    category = ComponentCategory.EXTENSION,
    nonVisible = true,
    iconName = "images/extension.png"  // 16x16 PNG icon, optional
)
@SimpleObject(external = true)
public class MyExtension extends AndroidNonvisibleComponent {

    public MyExtension(ComponentContainer container) {
        super(container.$form());
        // 'form' is the current Form (Activity)
        // container.$context() gives you the Android Context
    }

    @SimpleFunction(description = "Returns the sum of two numbers.")
    public int Add(int a, int b) {
        return a + b;
    }

    @SimpleProperty(description = "The current greeting message.")
    public String Greeting() {
        return greeting;
    }

    @DesignerProperty(
        editorType = PropertyTypeConstants.PROPERTY_TYPE_STRING,
        defaultValue = "Hello"
    )
    @SimpleProperty
    public void Greeting(String value) {
        this.greeting = value;
    }

    @SimpleEvent(description = "Fired when the operation completes.")
    public void OperationComplete(String result) {
        EventDispatcher.dispatchEvent(this, "OperationComplete", result);
    }

    private String greeting = "Hello";
}
```

Key rules the agent must follow when generating extension code:

1. **The class must have `@SimpleObject(external = true)`**. Omitting `external = true` means the
   annotation processor treats it as a built-in component, not an extension.

2. **The constructor takes `ComponentContainer container`**, not `Form`. Call `super(container.$form())`.

3. **Method names are PascalCase** — they become block names in the IDE. `Add`, `GetValue`, not
   `add`, `getValue`.

4. **Event methods must call `EventDispatcher.dispatchEvent()`** with `this`, the exact method name
   as a string, and the arguments in the same order as the method signature.

5. **Property getters and setters share the same PascalCase name.** The getter returns a value, the
   setter is void. Put `@DesignerProperty` on the setter if you want the property editable in the
   designer panel.

6. **Overloading is not supported.** Each block name must be unique.

---

## Build toolchains

There are two modern CLI tools for building extensions outside the App Inventor source tree. Both
produce AIX files and support Java and Kotlin.

### FAST (Feature-rich AppInventor Source Terminal) — recommended

FAST is actively maintained (v5.x as of 2026), supports Gradle and Maven dependency resolution,
AAR libraries, R8 shrinking, AIDL compilation, Kotlin, and ProGuard. It is the most feature-rich
option currently available.

```bash
# Install (Linux/macOS)
curl https://raw.githubusercontent.com/jewelshkjony/fast-cli/main/scripts/install.sh -fsSL | sh

# Create project
fast create MyExtension
# Prompts for: package name, author, language (Java/Kotlin)

# Build
cd MyExtension
fast build         # debug build
fast build -r      # release build with ProGuard
fast build -s      # release build with R8 shrinker
fast build -d      # dex with R8 dexer (smaller classes.jar)
fast build -z      # optimize AndroidRuntime.jar with hard ZIP compression
```

**Project structure (FAST):**
```
MyExtension/
├── src/
│   └── com/example/myextension/
│       └── MyExtension.java
├── deps/                    # local JAR/AAR dependencies
├── assets/                  # bundled assets (accessible at runtime)
├── AndroidManifest.xml      # optional, for activities/services/receivers
├── proguard-rules.pro       # optional ProGuard/R8 rules
├── fast.yml                 # project config
└── out/                     # built AIX output
```

**fast.yml** key fields:
```yaml
name: MyExtension
description: My awesome extension
min_sdk: 21
# Dependencies (premium feature):
deps:
  - com.squareup.okhttp3:okhttp:4.12.0
  - androidx.core:core:1.12.0
```

### Rush

Rush (by Shreyash Saitwal) was the pioneer modern extension builder. Latest stable is v1.2.5. The
2.0 branch is in development with Maven dependency resolution.

```bash
# Install (Linux/macOS)
curl https://raw.githubusercontent.com/shreyashsaitwal/rush-cli/main/scripts/install/install.sh -fsSL | sh

# Create and build
rush create MyExtension
cd MyExtension
rush build
```

**Project structure (Rush):**
```
MyExtension/
├── src/
│   └── com/example/myextension/
│       └── MyExtension.java
├── deps/                    # local JAR dependencies
├── assets/
├── rush.yml                 # project config
└── out/                     # built AIX output
```

### Which to use?

For an AI agent generating extensions, FAST is preferred because it has active dependency resolution
(Gradle/Maven), R8 support, and handles more edge cases. Rush is solid for simple extensions without
complex dependencies.

---

## Component lifecycle

Read `references/lifecycle.md` for the full lifecycle details including companion vs. built app
differences, Form/Activity lifecycle hooks, and extension classloading.

**Summary of the two runtime modes:**

| Aspect | Companion app | Built APK |
|--------|--------------|-----------|
| Extension loading | DexClassLoader loads classes.jar from pushed assets at runtime | Classes merged into app's DEX at build time |
| Asset access | Assets pushed over WiFi to device storage | Assets bundled in APK's assets/ directory |
| Restart required | Yes, after importing/changing extension | No (clean install) |
| Debugging | Logcat + companion console | Logcat |
| Native libs | Loaded from temp directory | Loaded from APK's lib/ directory |

---

## Annotations reference

Read `references/annotations.md` for the complete annotation API. Here is a brief summary:

- **`@SimpleObject(external = true)`** — marks the class as an extension component
- **`@DesignerComponent(...)`** — metadata for the IDE: version, description, category, icon
- **`@SimpleFunction(description = "...")`** — exposes a method as a block
- **`@SimpleEvent(description = "...")`** — exposes an event block
- **`@SimpleProperty(description = "...")`** — exposes a property block (getter or setter)
- **`@DesignerProperty(editorType = ..., defaultValue = "...")`** — makes a property editable in the designer panel
- **`@UsesPermissions(permissionNames = "...")`** — declares Android permissions
- **`@UsesLibraries(libraries = "...")`** — declares bundled JAR libraries
- **`@UsesAssets(fileNames = "...")`** — declares bundled assets
- **`@UsesActivities(activities = {...})`** — declares activities for the manifest
- **`@UsesBroadcastReceivers(receivers = {...})`** — declares broadcast receivers
- **`@UsesServices(services = {...})`** — declares services
- **`@UsesContentProviders(providers = {...})`** — declares content providers

---

## Common idioms and patterns

### Async operations with event callbacks

Extensions cannot return futures or use async/await. The pattern is: start work on a background
thread, then dispatch an event on the UI thread when done.

```java
@SimpleFunction(description = "Fetch data from a URL asynchronously.")
public void FetchUrl(final String url) {
    AsynchUtil.runAsynchronously(new Runnable() {
        @Override
        public void run() {
            try {
                // ... perform network I/O ...
                final String result = doHttpGet(url);
                // Post result back to UI thread
                form.runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        GotResponse(result);
                    }
                });
            } catch (final Exception e) {
                form.runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        ErrorOccurred(e.getMessage());
                    }
                });
            }
        }
    });
}

@SimpleEvent(description = "Fired when the URL fetch completes successfully.")
public void GotResponse(String responseContent) {
    EventDispatcher.dispatchEvent(this, "GotResponse", responseContent);
}

@SimpleEvent(description = "Fired when an error occurs.")
public void ErrorOccurred(String errorMessage) {
    EventDispatcher.dispatchEvent(this, "ErrorOccurred", errorMessage);
}
```

### Requesting runtime permissions (Android 6+)

```java
@SimpleFunction(description = "Request camera permission.")
public void RequestCameraPermission() {
    form.askPermission(android.Manifest.permission.CAMERA,
        new PermissionResultHandler() {
            @Override
            public void HandlePermissionResponse(String permission, boolean granted) {
                PermissionResult(granted);
            }
        });
}

@SimpleEvent
public void PermissionResult(boolean granted) {
    EventDispatcher.dispatchEvent(this, "PermissionResult", granted);
}
```

### Accessing the Activity and Context

```java
// In constructor or any method:
Activity activity = (Activity) form;              // the current Activity
Context context = form;                           // Form implements Context
// or: container.$context()  (in the constructor, from the ComponentContainer)
```

### Helper / dropdown blocks

To create dropdown menus (option lists) in the blocks editor:

```java
@SimpleProperty(description = "Set the mode.")
public void Mode(@Options(MyMode.class) int mode) {
    this.currentMode = mode;
}

@SimpleProperty
public @Options(MyMode.class) int Mode() {
    return currentMode;
}
```

Where `MyMode` is an enum in `com.google.appinventor.components.common` implementing `OptionList`.
For extensions, you define your own enum:

```java
public enum MyMode implements OptionList<Integer> {
    Fast(1),
    Normal(2),
    Slow(3);

    private final int value;
    MyMode(int value) { this.value = value; }

    public Integer toUnderlyingValue() { return value; }
}
```

### Multi-component extensions

A single AIX can contain multiple components. Each class gets its own `@DesignerComponent` and
`@SimpleObject(external = true)`. They all live in the same package, and the AIX bundles them
together. The user imports the AIX once and gets all components.

### Working with YailList

App Inventor's list type is `YailList`. When a `@SimpleFunction` accepts or returns a `java.util.List`
or `YailList`, it appears as a list socket in the blocks editor.

```java
@SimpleFunction(description = "Sort a list of numbers.")
public YailList SortNumbers(YailList numbers) {
    List<Object> items = new ArrayList<>();
    for (Object item : numbers.toArray()) {
        items.add(item);
    }
    Collections.sort(items, (a, b) ->
        Double.compare(Double.parseDouble(a.toString()),
                       Double.parseDouble(b.toString())));
    return YailList.makeList(items);
}
```

### Type mapping

| App Inventor type | Java type |
|-------------------|-----------|
| number | `int`, `float`, `double`, `long` |
| text | `String` |
| boolean | `boolean` |
| list | `YailList` or `java.util.List` |
| dictionary | `YailDictionary` |
| component | `Component` |
| color | `int` (use `@IsColor` annotation) |
| any | `Object` |

---

## Dependency management

This is the most common pain point. Read `references/dependencies.md` for the full guide.

**The core problem:** App Inventor's runtime already includes certain libraries (e.g., specific
versions of `appcompat`, `core`, `gson`, `okhttp`). If your extension bundles a different version
of the same library, you get one of: DEX merge conflicts at build time, `ClassNotFoundException`
at runtime, or subtle version-mismatch bugs.

**Key rules:**

1. **Never bundle libraries already on App Inventor's classpath.** Common ones include:
   `appcompat`, `core`, `annotations`, `gson`, `json`, `kxml2`. The build tools
   (Rush/FAST) provide these as compile-only dependencies.

2. **Use `compileOnly` / `provided` scope** for anything already in AI2's runtime. Only bundle
   libraries that AI2 does not include.

3. **When using FAST with Gradle/Maven resolution**, mark AI2-provided deps as `provided` in
   `fast.yml` or use FAST's built-in provided-deps list.

4. **ProGuard/R8 is your friend** for trimming bundled dependencies to only the classes you
   actually use, reducing AIX size and conflict surface.

5. **If two extensions bundle different versions of the same library**, the user gets a conflict.
   Use ProGuard to relocate (shade) your dependency's package if needed.

---

## AIX package format

Read `references/aix-format.md` for the detailed structure. Brief overview:

```
com.example.myextension.aix          (ZIP file)
└── com.example.myextension/
    ├── classes.jar                   (DEX bytecode, despite .jar extension)
    ├── components.json               (component descriptor array)
    ├── component_build_infos.json    (permissions, libraries, assets metadata)
    ├── AndroidRuntime.jar            (optional: extension's runtime support code)
    ├── assets/
    │   ├── extension.png             (component icon)
    │   └── ... (any bundled assets)
    ├── libs/                         (optional: native .so libraries)
    │   ├── armeabi-v7a/
    │   ├── arm64-v8a/
    │   └── x86/
    └── proguard.txt                  (optional: consumer ProGuard rules)
```

### simple_components.json (for debugging)

This file is generated by the annotation processor and describes every component, property, method,
and event. It is the single source of truth for what the IDE and runtime know about the extension.
When debugging "my block doesn't appear" issues, inspect this file.

A component entry looks like:

```json
{
  "type": "com.example.myextension.MyExtension",
  "name": "MyExtension",
  "external": "true",
  "version": "1",
  "categoryString": "EXTENSION",
  "helpString": "Short user-facing description...",
  "showOnPalette": "true",
  "nonVisible": "true",
  "iconName": "images/extension.png",
  "properties": [ ... ],
  "blockProperties": [ ... ],
  "events": [ ... ],
  "methods": [ ... ]
}
```

Each method entry includes name, description, parameter names/types, return type, and whether
it is deprecated. If something is missing here, it won't appear in the blocks editor.

---

## Debugging tips

1. **Block doesn't appear in IDE after import** — Check `components.json` inside the AIX (unzip it).
   Verify the annotation processor ran correctly. Common cause: missing `@SimpleObject(external = true)`.

2. **`ClassNotFoundException` in companion** — The companion loads extension DEX via
   `DexClassLoader`. If you changed the package name or class name without reimporting, the old
   metadata still points to the old class. Reimport the AIX and restart the companion.

3. **Works in companion but crashes in built APK (or vice versa)** — Usually a dependency version
   mismatch. The companion has its own classpath; the built APK merges everything. Check for
   duplicate classes.

4. **`NoSuchMethodError` at runtime** — You're compiling against a newer API than what's on the
   device. Check `min_sdk` in your project config.

5. **ProGuard strips your code** — Add keep rules for your extension class and any classes
   referenced by reflection. FAST auto-keeps classes declared in AndroidManifest.xml when `-m`
   is passed.

6. **Extension works on one fork but not another** — Different forks (MIT AI2, Kodular, Niotron,
   AppyBuilder) ship different runtime library versions. Test on each target.

---

## Loading extensions into App Inventor

1. In the App Inventor designer, look under the component palette for "Extension"
2. Click "Import extension"
3. Either upload the `.aix` file from your computer or provide a URL
4. The extension appears under "Extension" in the palette
5. Drag it into the viewer (it shows as a non-visible component below the phone mockup)
6. **Restart the companion** after importing or updating an extension

When exporting a project as `.aia`, all imported extensions are bundled inside the project file.
Other users who open the `.aia` do not need to import the extensions separately.
