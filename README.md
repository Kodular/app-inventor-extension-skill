# app-inventor-extension-skill

AI Skill to develop App Inventor Extensions.

This skill teaches AI agents (Claude, etc.) how to write, build, debug, and package App Inventor / Kodular extensions (`.aix` files) from scratch. It covers the full extension development stack — from annotations and component lifecycle to build toolchains and dependency management.

## What's inside

```
├── SKILL.md                        # Main skill document
├── references/
│   ├── annotations.md              # Complete annotation API reference
│   ├── aix-format.md               # AIX ZIP structure & components.json
│   ├── lifecycle.md                 # Companion vs built-app runtime, Form lifecycle
│   └── dependencies.md             # Dependency conflicts, ProGuard/R8, shading
└── README.md
```

## What it covers

- **Extension class skeleton** — correct annotations, constructor pattern, PascalCase naming, type mapping
- **Build toolchains** — [FAST](https://github.com/jewelshkjony/fast-cli) (recommended) and [Rush](https://github.com/shreyashsaitwal/rush-cli), project setup, build commands
- **Full annotation API** — `@SimpleFunction`, `@SimpleEvent`, `@SimpleProperty`, `@DesignerComponent`, `@DesignerProperty`, `@Options`, `@IsColor`, `@UsesPermissions`, `@UsesLibraries`, manifest annotations
- **AIX package format** — ZIP structure, `classes.jar` (DEX), `components.json`, `component_build_infos.json`, assets, native libs
- **Component lifecycle** — companion app DexClassLoader loading vs built APK, Form/Activity lifecycle hooks (`OnPause`, `OnResume`, `OnDestroy`, `ActivityResultListener`)
- **Dependency management** — what's already on AI2's classpath, version conflict diagnosis, ProGuard/R8 shading, AAR support
- **Common idioms** — async operations with event callbacks, runtime permissions, `YailList`, dropdown/helper blocks, multi-component extensions
- **Debugging** — `simple_components.json` introspection, companion vs APK differences, `ClassNotFoundException` diagnosis

## Installation

Download the `.skill` file from [Releases](https://github.com/Kodular/app-inventor-extension-skill/releases) and install it in your AI tool of choice.

Or clone this repo and point your tool at the repository root.

## Example prompts

Once installed, the skill triggers on prompts like:

- *"Build me an App Inventor extension that fetches weather data from an API"*
- *"I'm getting ClassNotFoundException in the companion but the APK works fine"*
- *"Write a Kodular extension in Kotlin that wraps the Android BLE scanner"*
- *"What's inside an AIX file?"*
- *"How do I add OkHttp as a dependency without conflicting with AI2's version?"*

## Related projects

- [FAST CLI](https://github.com/jewelshkjony/fast-cli) — Feature-rich AppInventor Source Terminal
- [Rush CLI](https://github.com/shreyashsaitwal/rush-cli) — Modern extension builder for MIT AI2
- [MIT App Inventor](https://github.com/mit-cml/appinventor-sources) — App Inventor source code
- [Kodular](https://www.kodular.io/) — No-code Android app builder (App Inventor fork)

## License

MIT
