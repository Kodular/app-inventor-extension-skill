# Dependency Management

Dependency management is the single most common source of bugs, build failures, and
frustration in App Inventor extension development. This document explains why and how to
handle it.

---

## The core problem

App Inventor's runtime (both the companion app and the build server) already includes a set of
libraries on its classpath. These are compiled into the companion APK and linked during app
builds. When your extension bundles a library that overlaps with one already present, you get
one of three failure modes:

1. **DEX merge failure at build time** — two copies of the same class exist
2. **`ClassNotFoundException` or `NoSuchMethodError` at runtime** — the wrong version of a
   class is loaded (e.g., companion uses v1.0 but your code calls a v2.0 method)
3. **Silent misbehavior** — the method exists in both versions but behaves differently

### Libraries already in App Inventor's classpath

This list varies by fork and version. For MIT App Inventor (as of nb197+):

- `appcompat` and related AndroidX libraries
- `androidx.core`
- `com.google.android.material`
- `gson` (com.google.gson)
- `json.org` (org.json)
- `kxml2` (org.kxml2)
- `commons-io`
- `guava` (selected parts)
- `okhttp` (selected version)
- `fasterxml.jackson` (selected parts)
- `appinventor-components` (annotations, runtime, common)

**For Kodular**, additional libraries may be present (e.g., Firebase, AdMob, etc.).

### How to check what's provided

**FAST** maintains a `provided-deps.xml` file in `$FAST_HOME/lib/` that lists all provided
dependencies. Dependencies listed here are available at compile time but not bundled into the AIX.

**Rush** ships its own set of provided JARs in its `deps/` directory.

You can also inspect the companion APK:

```bash
# Extract classes from companion APK
apktool d MITAI2Companion.apk -o companion_extracted/
# Or use dex2jar / jadx to see what classes are present
```

---

## Dependency strategies

### Strategy 1: Use only provided libraries (safest)

If your extension only needs libraries already on AI2's classpath, don't bundle anything. Use
the classes directly — they'll be available at runtime.

```java
// This works because gson is already on AI2's classpath
import com.google.gson.Gson;
import com.google.gson.JsonObject;

@SimpleFunction
public String ToJson(String key, String value) {
    JsonObject obj = new JsonObject();
    obj.addProperty(key, value);
    return new Gson().toJson(obj);
}
```

No `@UsesLibraries`, no JARs in `deps/`. Clean and conflict-free.

### Strategy 2: Bundle a library not in AI2 (straightforward)

If you need a library AI2 doesn't include (e.g., a specific SDK), bundle it.

**With FAST (Gradle resolver):**
```yaml
# fast.yml
deps:
  - com.squareup.retrofit2:retrofit:2.9.0
```

**With Rush or FAST (manual):**
1. Download the JAR (and its transitive deps)
2. Place in `deps/` folder
3. Declare with `@UsesLibraries`:
```java
@UsesLibraries(libraries = "retrofit-2.9.0.jar, okhttp-4.12.0.jar, okio-3.6.0.jar")
```

**Watch out for transitive dependencies** — Retrofit depends on OkHttp, which AI2 might already
include. You need to check versions.

### Strategy 3: Shade / relocate conflicting packages (advanced)

When you absolutely must use a different version of a library that AI2 already includes, use
ProGuard/R8 to relocate the package:

```
# proguard-rules.pro
-repackageclasses 'com.example.myextension.shaded'
-keep class com.example.myextension.** { *; }
```

Or, if using a build tool that supports shading (like the Gradle Shadow plugin in a pre-build
step), relocate the conflicting library's package:

```
com.google.gson -> com.example.myextension.shaded.gson
```

This is fragile and complex. Avoid it if at all possible.

### Strategy 4: Use AAR libraries

FAST supports AAR (Android Archive) libraries, which can include resources, native code, and
manifest entries. To use AARs:

1. Place the `.aar` file in `deps/`
2. FAST extracts classes, resources, and JNI libs during the build
3. ProGuard/R8 rules from the AAR are applied automatically when using `-r` or `-s`

Rush (stable) does not support AARs directly — you need to extract the `classes.jar` from the
AAR manually.

---

## Handling version conflicts between extensions

When a user imports two extensions that bundle different versions of the same library, they
get a DEX merge conflict. This is a common user complaint.

**As an extension developer, you can:**

1. **Minimize what you bundle** — only include what's truly needed
2. **Use ProGuard/R8 aggressively** — strip unused classes so the conflict surface is smaller
3. **Shade your dependencies** — relocate packages so they don't conflict
4. **Document your dependencies** — tell users what libraries your extension bundles

**There is a community tool called Conflixer** that helps users resolve conflicts between
extensions by identifying and removing duplicate classes.

---

## ProGuard / R8 configuration for extensions

### When to use ProGuard/R8

- Your extension bundles external libraries (reduces AIX size and conflict surface)
- You want to strip unused code
- You need to obfuscate code

### Basic ProGuard rules for extensions

```pro
# Keep the extension class itself
-keep public class com.example.myextension.MyExtension {
    public *;
}

# Keep all classes with App Inventor annotations
-keepclassmembers class * {
    @com.google.appinventor.components.annotations.SimpleFunction *;
    @com.google.appinventor.components.annotations.SimpleEvent *;
    @com.google.appinventor.components.annotations.SimpleProperty *;
}

# Keep enum classes used as @Options
-keep enum com.example.myextension.** { *; }

# Keep classes referenced in AndroidManifest.xml
-keep class com.example.myextension.MyService { *; }
-keep class com.example.myextension.MyReceiver { *; }

# Don't warn about AI2 runtime classes (they're provided, not bundled)
-dontwarn com.google.appinventor.**
```

### FAST-specific ProGuard notes

FAST auto-keeps manifest-declared classes when `-m` flag is passed. With `-r` flag, ProGuard
is applied; with `-s` flag, R8 shrinker is used (generally produces smaller output).

FAST also supports applying ProGuard/R8 rules from runtime AARs automatically.

---

## Native libraries (.so files)

If your extension includes JNI / native code:

1. Compile `.so` files for target ABIs (armeabi-v7a, arm64-v8a, x86_64)
2. Place them in `libs/<ABI>/` in your project
3. They end up in the AIX under `com.example.myextension/libs/<ABI>/`
4. At runtime:
   - **Companion**: copies `.so` to a temp directory and loads via `System.load()`
   - **Built APK**: `.so` files go into APK's `lib/<ABI>/` and load normally

```java
// Loading native library in extension
static {
    System.loadLibrary("mynativelib");
}

// Declare the native method
public native String nativeProcess(String input);
```

---

## Common dependency mistakes

1. **Bundling `appinventor-components.jar`** — Never do this. It's always provided.

2. **Bundling `gson.jar` when AI2 already has it** — Use the provided version. If you need a
   newer version, shade it.

3. **Using `implementation` scope for everything** — Only `implementation` (or just dropping JARs
   in `deps/`) should be for libraries NOT in AI2's classpath. For AI2-provided libs, they should
   be compile-only.

4. **Forgetting transitive dependencies** — If you bundle `library-a.jar` which depends on
   `library-b.jar`, you need both (unless `library-b` is already provided by AI2).

5. **Not testing on the companion** — Build-time success doesn't mean runtime success. The
   companion's classpath is different from the build-time classpath.
