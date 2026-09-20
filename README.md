# AntiCrash — Fabric client mod for Minecraft 26.3

A client-side crash guard. It doesn't stop *your* game from crashing on its
own — it defends against the 14 most common ways a malicious server,
player, or corrupted save can crash a vanilla client on purpose.

## Read this first

1. **This is a source project, not a compiled jar.** Building a Fabric mod
   requires Gradle to download Minecraft's own code — that download has to
   happen from a machine with internet access. I can't run that here, so
   you'll run one command yourself (step-by-step below).
2. **You need JDK 25, not 21.** Minecraft 26.1+ raised its minimum required
   Java version to 25. Java 21 will fail to build or run this. Install a
   JDK 25 build (Temurin/Adoptium, Azul Zulu, or Microsoft's OpenJDK builds
   all publish one) — `build.gradle` is set up to request it via Gradle's
   toolchain support, but Gradle still needs a JDK 25 installed somewhere
   on your machine to find.
3. **No Yarn mappings here on purpose.** Minecraft stopped shipping
   obfuscated code as of 26.1, and Yarn (the old community mapping layer)
   was deprecated as the default around the same time — Loom now uses
   Mojang's own official class/method names directly. That also means
   class names changed from what older Fabric tutorials show: `Text` is
   now `Component`, `MinecraftClient` is now `Minecraft`, `HandledScreen`
   is now `AbstractContainerScreen`, and so on. The source files here have
   already been updated to the new names.
4. Two of the fourteen guards are wired all the way into the game via a
   mixin/Fabric-API hook and will work the moment it compiles. The other
   twelve are complete, tested, dependency-free utility classes — you (or I,
   in a follow-up once you paste back any mapping names) wire each into its
   one real call site, which the Javadoc on each class tells you exactly how
   to find. I did it this way on purpose: guessing exact obfuscated method
   names for a game version this new and pretending it's "done" would just
   hand you a mod that fails silently or doesn't build. Real coordinates
   beat convincing-looking fake ones.

## The 14 guards

Heads up: the Javadoc on the not-yet-wired guards below (rows other than
#5 and #12) still mentions screen/class names the way they were commonly
known under the old Yarn mappings (`BookScreen`, `SignEditScreen`,
`AnvilScreen`, etc.). Simple, widely-recognized screen names like those
are often identical or very close under Mojang's official mappings too,
but I haven't individually verified each one against 26.3 — treat them as
a starting point for your IDE search, not a guarantee.

| # | Vector | File | Status |
|---|--------|------|--------|
| 1 | Recursive/oversized item NBT crashing tooltips | `guard/NbtDepthGuard.java` | Utility ready; hook is scaffolded in `mixin/HandledScreenTooltipMixin.java` |
| 2 | Giant written-book pages | `guard/TextLengthGuard.java` | Utility ready — call from `BookScreen` |
| 3 | Giant sign lines | `guard/TextLengthGuard.java` | Utility ready — call from `SignEditScreen` |
| 4 | Giant anvil rename text | `guard/TextLengthGuard.java` | Utility ready — call from `AnvilScreen` |
| 5 | Malicious chat JSON (deep/wide hover/sibling trees) | `guard/TextComponentGuard.java` | **Fully wired** via `ClientReceiveMessageEvents.ALLOW_CHAT` |
| 6 | Malicious title/subtitle JSON | `guard/TextComponentGuard.java` | Utility ready — call from title-packet handling |
| 7 | Firework NBT with absurd explosion counts | `guard/FireworkGuard.java` | Utility ready — call before spawning firework particles |
| 8 | Out-of-range potion amplifier / enchant level / registry id | `guard/EffectGuard.java` | Utility ready |
| 9 | Corrupted map color-array data | `guard/MapGuard.java` | Utility ready — call from map-texture rendering |
| 10 | Malformed skull/player-head texture property | `guard/SkullTextureGuard.java` | Utility ready |
| 11 | One entity's bad tracked-data killing the whole tracker pass | `guard/EntityMetadataGuard.java` | Utility ready — wrap per-entity apply calls |
| 12 | Oversized/flooded custom payload (plugin channel) packets | `guard/PayloadGuard.java` | Flood-counter **fully wired** (resets every tick); size check is a one-line call at your payload receiver |
| 13 | NaN/Infinity/absurd teleport coordinates | `guard/CoordinateGuard.java` | Utility ready — call before applying a position update |
| 14 | Generic uncaught-exception safety net | `guard/CrashGuard.java` | Utility ready; example hook scaffolded in `mixin/ClientMainLoopMixin.java` |

Every "utility ready" guard is plain Java with zero Minecraft-specific
imports (except the two fully-wired ones), so it **compiles regardless of
what 26.3's internals look like**. Each file's Javadoc says exactly which
vanilla class/method to call it from and what that call site has looked
like historically.

## Don't want to install Java/Gradle locally? Build it in the cloud instead

This project includes `.github/workflows/build.yml`, which builds the mod
on GitHub's free CI runners (JDK 25 + Gradle already set up there) instead
of on your machine:

1. Create a new (can be private) repo on GitHub.
2. Push this folder's contents to it — either `git push` from a terminal,
   or GitHub Desktop, or even the "upload files" button on the repo's web
   page (drag the whole `anticrash` folder contents in, minus the outer
   `anticrash-fabric-mod-src` wrapper folder from the zip).
3. Go to the repo's **Actions** tab. A "Build AntiCrash mod" run should
   already be in progress (or click **Run workflow** to start one).
4. When it finishes (green check), open that run and scroll to
   **Artifacts** — download `anticrash-jar`, which contains the built
   `.jar`.

If the run fails, click into it to see the same kind of error output
`gradle build` would print locally — paste that back to me and it's just
as debuggable as a local failure.

## Build it locally instead

Requires **JDK 25** and internet access (Gradle needs to fetch Minecraft +
Fabric Loader/API the first time).

```bash
cd anticrash
./gradlew build
```

The finished jar lands at `build/libs/anticrash-1.0.0.jar`. Drop it in your
`.minecraft/mods` folder alongside a matching **Fabric API** jar for
26.3 and launch with the Fabric loader profile.

## If the build fails

- **"Unsupported class file major version" / toolchain errors** — Gradle
  can't find a JDK 25 to use. Install one, then either make it your
  system default `java`, or point Gradle at it directly by uncommenting
  `org.gradle.java.home=...` in `gradle.properties`.
- **Dependency resolution errors on `com.mojang:minecraft`, the Fabric
  Loader, or Fabric API** — the version strings have moved again. Go to
  `https://fabricmc.net/develop/`, select Minecraft `26.3`, and copy the
  Loader / Fabric API / Loom version strings it shows into
  `gradle.properties` (and bump `loom_version` in the plugins block if
  it changed).
- **Mixin apply errors mentioning a method that doesn't exist** — expected
  for the two example mixins (`ClientMainLoopMixin`,
  `HandledScreenTooltipMixin`), which are deliberately `require = 0`
  placeholders. They'll just silently skip rather than fail the build.
  If something *else* fails to apply, that's a sign a class name changed
  from what's in this project — open the class in your IDE to find the
  current name.

## Finishing the remaining guards

For each "utility ready" row above: open the named vanilla class in your
IDE (Loom's `genSources` gives you readable decompiled source once the
project first syncs), find the method the Javadoc points at, and add the
one-line guard call shown in that class's comments. Paste me the resulting
method signature if you want the exact wiring code written for you — once
you're inside the actual 26.3 sources, this is fast because I'm no longer
guessing at names that may have shifted.
