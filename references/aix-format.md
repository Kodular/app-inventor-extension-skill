# AIX Package Format

The `.aix` file is a ZIP archive with a specific directory layout. Understanding this format is
essential for debugging import failures, inspecting built extensions, and understanding what the
build toolchain produces.

---

## Directory structure

```
com.example.myextension.aix                    ← ZIP file
└── com.example.myextension/                   ← top-level dir = fully qualified package name
    ├── classes.jar                             ← DEX bytecode (NOT a Java JAR despite the name)
    ├── components.json                         ← array of component descriptors
    ├── component_build_infos.json              ← array of build metadata
    ├── AndroidRuntime.jar                      ← optional: additional runtime support DEX
    ├── extension.properties                    ← optional: key-value metadata
    ├── assets/
    │   ├── FQCN/                               ← subdirectory named after component's FQCN
    │   │   ├── component.png                   ← icon shown in designer palette
    │   │   └── custom_asset.dat                ← any bundled file declared with @UsesAssets
    │   └── ...
    ├── libs/                                   ← optional: native shared libraries
    │   ├── armeabi-v7a/
    │   │   └── libmynative.so
    │   ├── arm64-v8a/
    │   │   └── libmynative.so
    │   └── x86_64/
    │       └── libmynative.so
    └── proguard.txt                            ← optional: consumer ProGuard rules
```

---

## classes.jar

Despite the `.jar` extension, this file contains **DEX bytecode** (Dalvik Executable), not
Java bytecode. This is a historical naming convention.

The build toolchain:
1. Compiles `.java`/`.kt` → `.class` files (javac / kotlinc)
2. Runs d8 or R8 to convert `.class` → `.dex`
3. Packages the `.dex` as `classes.jar`

When debugging, you can inspect the DEX contents:

```bash
# Unzip the AIX
unzip com.example.myextension.aix -d aix_contents/

# List classes in the DEX
# Using dexdump (from Android SDK build-tools)
dexdump -f aix_contents/com.example.myextension/classes.jar

# Or using jadx for decompilation
jadx aix_contents/com.example.myextension/classes.jar -d decompiled/
```

---

## components.json

An array of component descriptor objects. Each object describes one component class and its
blocks (properties, methods, events). This is the same format as `simple_components.json`
generated during the build.

```json
[
  {
    "type": "com.example.myextension.MyExtension",
    "name": "MyExtension",
    "external": "true",
    "version": "1",
    "categoryString": "EXTENSION",
    "helpString": "Description shown in designer...",
    "helpUrl": "",
    "showOnPalette": "true",
    "nonVisible": "true",
    "iconName": "aiwebres/com.example.myextension/icon.png",
    "properties": [
      {
        "name": "Greeting",
        "editorType": "string",
        "defaultValue": "Hello",
        "editorArgs": []
      }
    ],
    "blockProperties": [
      {
        "name": "Greeting",
        "description": "The current greeting message.",
        "type": "text",
        "rw": "read-write",
        "deprecated": "false"
      }
    ],
    "events": [
      {
        "name": "OperationComplete",
        "description": "Fired when the operation completes.",
        "deprecated": "false",
        "params": [
          { "name": "result", "type": "text" }
        ]
      }
    ],
    "methods": [
      {
        "name": "Add",
        "description": "Returns the sum of two numbers.",
        "deprecated": "false",
        "params": [
          { "name": "a", "type": "number" },
          { "name": "b", "type": "number" }
        ],
        "returnType": "number"
      }
    ]
  }
]
```

### Debugging with components.json

Common issues to look for:

- **Missing method/event/property**: The annotation was not processed. Check that the class has
  `@SimpleObject(external = true)` and each member has the correct annotation.
- **Wrong type**: The type mapping wasn't what you expected. For example, `long` maps to `number`.
- **`"external": "false"`**: You forgot `external = true` in `@SimpleObject`.
- **`"showOnPalette": "false"`**: Something in the annotations is preventing the component from
  showing.

---

## component_build_infos.json

Contains build-time metadata for each component: permissions, libraries, assets, manifest entries.

```json
[
  {
    "type": "com.example.myextension.MyExtension",
    "permissions": [
      "android.permission.INTERNET"
    ],
    "libraries": [],
    "assets": [],
    "activities": [],
    "broadcastReceivers": [],
    "services": [],
    "contentProviders": []
  }
]
```

When you use `@UsesPermissions`, `@UsesLibraries`, etc., they end up here. At app build time,
App Inventor reads this file and merges the entries into the app's AndroidManifest.xml and build
configuration.

---

## AndroidRuntime.jar

Some extensions include an additional `AndroidRuntime.jar` alongside `classes.jar`. This is used
for extensions that need runtime support code that should be loaded separately. FAST supports
generating an optimized AndroidRuntime.jar with the `-z` flag for hard ZIP compression.

---

## Multi-component AIX

If your extension contains multiple `@SimpleObject(external = true)` classes, they all appear in
`components.json` as separate entries. The `classes.jar` contains all of their DEX bytecode. The
user imports the AIX once and gets all components.

---

## Inspecting an AIX manually

```bash
# Rename to .zip and extract (or just use unzip directly)
cp MyExtension.aix MyExtension.zip
unzip MyExtension.zip -d aix_extracted/

# Check the structure
find aix_extracted/ -type f

# Pretty-print the component descriptor
cat aix_extracted/com.example.myextension/components.json | python3 -m json.tool

# Check the build infos
cat aix_extracted/com.example.myextension/component_build_infos.json | python3 -m json.tool

# Verify the DEX is valid
file aix_extracted/com.example.myextension/classes.jar
# Should show: "Dalvik dex file version 035" or similar
```

---

## Common AIX issues

1. **Wrong top-level directory name**: Must exactly match the package name of the main extension
   class. If it doesn't match, App Inventor won't find the component.

2. **Java JAR instead of DEX**: If `classes.jar` contains Java bytecode instead of DEX, the
   build tool didn't run d8/R8. The companion and APK builder expect DEX.

3. **Missing components.json**: Without this file, the IDE doesn't know what blocks to show.
   Rebuild with a properly configured annotation processor.

4. **Asset paths wrong**: Assets must be in the `assets/FQCN/` subdirectory where FQCN is the
   fully qualified class name of the component that declared them with `@UsesAssets`.
