# Annotations Reference

This document is the complete reference for all annotations used in App Inventor extension
development. The annotation processor reads these at compile time and generates
`simple_components.json` and `simple_component_build_infos.json`.

All annotations live in `com.google.appinventor.components.annotations`.

---

## @SimpleObject

Marks a class for processing by the annotation system.

```java
@SimpleObject(external = true)
```

- **`external`** (boolean, default `false`): Must be `true` for extensions. When `false`, the
  class is treated as a built-in component (only valid inside the AI2 source tree).

Without this annotation, no other annotations on the class will be processed.

---

## @DesignerComponent

Metadata for the component as it appears in the App Inventor designer.

```java
@DesignerComponent(
    version = 1,
    description = "HTML description shown in the designer tooltip and docs",
    category = ComponentCategory.EXTENSION,
    nonVisible = true,
    iconName = "images/extension.png",
    helpUrl = "https://example.com/docs",
    androidMinSdk = 21
)
```

**Elements:**

| Element | Type | Required | Notes |
|---------|------|----------|-------|
| `version` | `int` | Yes | Increment on every user-visible change |
| `description` | `String` | Yes | HTML allowed. Shown in designer tooltip and generated docs |
| `category` | `ComponentCategory` | Yes | Must be `ComponentCategory.EXTENSION` for extensions |
| `nonVisible` | `boolean` | No (default `false`) | `true` for non-visible components (most extensions) |
| `iconName` | `String` | No | Path to 16x16 PNG icon, relative to assets |
| `helpUrl` | `String` | No | URL to external documentation |
| `androidMinSdk` | `int` | No | Minimum Android SDK version |

---

## @SimpleFunction

Exposes a method as a method block in the blocks editor.

```java
@SimpleFunction(description = "Returns the sum of two numbers.")
public int Add(int a, int b) {
    return a + b;
}
```

**Elements:**

| Element | Type | Default | Notes |
|---------|------|---------|-------|
| `description` | `String` | `""` | Tooltip text in blocks editor |
| `userVisible` | `boolean` | `true` | Set `false` to hide from blocks editor |

**Rules:**
- Method name becomes the block name — use PascalCase
- `void` methods become statement blocks (top/bottom connectors)
- Non-void methods become value blocks (plug on the left)
- No method overloading — each name must be unique
- Parameters can be any supported type (see type mapping in SKILL.md)

---

## @SimpleEvent

Exposes an event that can be handled with an event block.

```java
@SimpleEvent(description = "Triggered when data is received.")
public void DataReceived(String data, int statusCode) {
    EventDispatcher.dispatchEvent(this, "DataReceived", data, statusCode);
}
```

**Critical:** The event method body must call `EventDispatcher.dispatchEvent()` with:
1. `this` — the component instance
2. The method name as a string — **must match exactly**, case-sensitive
3. The arguments in the same order as the method parameters

**Elements:**

| Element | Type | Default | Notes |
|---------|------|---------|-------|
| `description` | `String` | `""` | Tooltip text |
| `userVisible` | `boolean` | `true` | Set `false` to hide |

Events always return `void`.

---

## @SimpleProperty

Exposes a property with getter and/or setter blocks.

```java
// Getter
@SimpleProperty(description = "The current value.")
public int Value() {
    return this.value;
}

// Setter
@SimpleProperty
public void Value(int value) {
    this.value = value;
}
```

**Elements:**

| Element | Type | Default | Notes |
|---------|------|---------|-------|
| `description` | `String` | `""` | Put on either getter or setter, not both |
| `userVisible` | `boolean` | `true` | Set `false` to hide from blocks |
| `category` | `PropertyCategory` | — | Deprecated, no longer used |

**Rules:**
- Getter and setter share the same PascalCase name
- Getter returns a value, setter takes one parameter and returns void
- You can have getter-only (read-only) or setter-only (write-only) properties
- The `description` should be placed on one of them, not both

---

## @DesignerProperty

Makes a property editable in the designer's properties panel (right side of IDE).

```java
@DesignerProperty(
    editorType = PropertyTypeConstants.PROPERTY_TYPE_STRING,
    defaultValue = "Hello"
)
@SimpleProperty
public void Greeting(String value) {
    this.greeting = value;
}
```

**Elements:**

| Element | Type | Notes |
|---------|------|-------|
| `editorType` | `String` | One of the `PropertyTypeConstants` values |
| `defaultValue` | `String` | Default value shown in designer (always a string) |
| `alwaysSend` | `boolean` | If `true`, value is always sent even if unchanged |

**Common editor types** (from `PropertyTypeConstants`):

| Constant | Shows as |
|----------|----------|
| `PROPERTY_TYPE_STRING` | Text field |
| `PROPERTY_TYPE_INTEGER` | Integer field |
| `PROPERTY_TYPE_FLOAT` | Float field |
| `PROPERTY_TYPE_BOOLEAN` | Checkbox |
| `PROPERTY_TYPE_COLOR` | Color picker |
| `PROPERTY_TYPE_TEXTAREA` | Multi-line text area |
| `PROPERTY_TYPE_ASSET` | Asset chooser (file picker) |
| `PROPERTY_TYPE_COMPONENT` | Component selector |
| `PROPERTY_TYPE_VISIBILITY` | Visibility options |
| `PROPERTY_TYPE_CHOICES` | Dropdown (needs custom implementation) |
| `PROPERTY_TYPE_NON_NEGATIVE_INTEGER` | Non-negative integer field |
| `PROPERTY_TYPE_NON_NEGATIVE_FLOAT` | Non-negative float field |

---

## @IsColor

Marks an `int` parameter or return value as a color value, enabling the color block socket.

```java
@SimpleProperty
public void BackgroundColor(@IsColor int color) {
    this.bgColor = color;
}
```

---

## @Options

Associates a parameter or return value with an OptionList enum, creating dropdown helper blocks.

```java
@SimpleProperty
public void Direction(@Options(Direction.class) int dir) { ... }
```

The enum must implement `OptionList<T>`:

```java
public enum Direction implements OptionList<Integer> {
    North(0), South(1), East(2), West(3);
    private final int value;
    Direction(int v) { this.value = v; }
    public Integer toUnderlyingValue() { return value; }
}
```

---

## Build info annotations

These annotations declare what the extension needs from the Android system. They end up in
`component_build_infos.json` and are merged into the app's AndroidManifest.xml at build time.

### @UsesPermissions

```java
@UsesPermissions(permissionNames = "android.permission.INTERNET, android.permission.ACCESS_FINE_LOCATION")
```

Or with `@UsesBroadcastReceivers`-style array syntax:

```java
@UsesPermissions(value = {
    @PermissionConstraint(name = "android.permission.CAMERA", maxSdkVersion = 30)
})
```

### @UsesLibraries

Declares JAR files that must be included. The JARs go in the `deps/` folder.

```java
@UsesLibraries(libraries = "okhttp-4.12.0.jar, okio-3.6.0.jar")
```

### @UsesAssets

Declares asset files bundled with the extension.

```java
@UsesAssets(fileNames = "model.tflite, labels.txt")
```

### @UsesActivities

```java
@UsesActivities(activities = {
    @ActivityElement(
        name = "com.example.myextension.MyActivity",
        configChanges = "orientation|screenSize",
        screenOrientation = "portrait"
    )
})
```

### @UsesBroadcastReceivers

```java
@UsesBroadcastReceivers(receivers = {
    @ReceiverElement(
        name = "com.example.myextension.MyReceiver",
        intentFilters = {
            @IntentFilterElement(actionElements = {
                @ActionElement(name = "android.intent.action.BOOT_COMPLETED")
            })
        }
    )
})
```

### @UsesServices

```java
@UsesServices(services = {
    @ServiceElement(
        name = "com.example.myextension.MyService",
        exported = "false"
    )
})
```

### @UsesContentProviders

```java
@UsesContentProviders(providers = {
    @ProviderElement(
        name = "com.example.myextension.MyProvider",
        authorities = "com.example.myextension.provider",
        exported = "false",
        grantUriPermissions = "true"
    )
})
```

---

## Annotation processor pipeline

At build time, the annotation processor (part of the AI2 components JAR, or the Rush/FAST
processor) scans all `@SimpleObject` classes and:

1. Validates annotations (e.g., checks that event methods return void)
2. Generates `simple_components.json` — the complete component descriptor
3. Generates `simple_component_build_infos.json` — permissions, libs, assets, manifest entries
4. For extensions: packages these into the AIX alongside the compiled DEX

If you see annotations that don't produce the expected blocks, check the processor output.
Both Rush and FAST log annotation processing results during the build.
