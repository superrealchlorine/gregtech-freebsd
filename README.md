# GregTech: New Horizons on FreeBSD

A little guide for running **GregTech: New Horizons (GTNH)** on **FreeBSD 15.x** using Prism Launcher, LWJGL3, and modern Java.

## Tested configuration

- FreeBSD 15.x
- Prism Launcher
- KDE Plasma 6 / KWin (Wayland)
- Mesa graphics drivers
- OpenJDK 25
- GTNH with lwjgl3ify 3.0.42

This guide documents the local compatibility adjustments required to get GTNH running on FreeBSD. It is not an official GTNH project.

## 1. Install dependencies

Install the required packages as root:

```sh
pkg install prismlauncher openjdk25 lwjgl3 glfw openal-soft sdl3
```

## 2. Import the GTNH instance

1. Download the Java 17–25 Prism/MultiMC instance archive from [GTNH Downloads](https://downloads.gtnewhorizons.com/).
2. Import it into Prism Launcher.
3. Close Prism Launcher completely before modifying instance files.

Set an instance variable for convenience:

```sh
INST="$HOME/.local/share/PrismLauncher/instances/GT-NH"
```

Adjust this path if your instance has a different name or location.

### A. Fix forgePatches resolution

The FreeBSD Prism Launcher build may fail to resolve the lwjgl3ify `forgePatches` Maven artifact automatically.

This is not a separate RetroFuturaBootstrap download. RetroFuturaBootstrap is bundled with lwjgl3ify; `forgePatches` is a separate artifact containing compatibility patches required by the launcher.

For lwjgl3ify 3.0.42, download:

**`lwjgl3ify-3.0.42-forgePatches.jar`**

Sources:

- [GitHub – lwjgl3ify 3.0.42 releases](https://github.com/GTNewHorizons/lwjgl3ify/releases/tag/3.0.42)
- [CurseForge – lwjgl3ify 3.0.42](https://www.curseforge.com/minecraft/mc-mods/lwjgl3ify/files/9376133)
- [CurseForge – Additional files](https://www.curseforge.com/minecraft/mc-mods/lwjgl3ify/files/9376133/additional-files)

The main CurseForge download is `lwjgl3ify-3.0.42.jar`. Make sure you download the separate `forgePatches` artifact instead.

Download directly from the GitHub release:

```sh
mkdir -p "$INST/libraries"

fetch -o "$INST/libraries/lwjgl3ify-3.0.42-forgePatches.jar" \
  "https://github.com/GTNewHorizons/lwjgl3ify/releases/download/3.0.42/lwjgl3ify-3.0.42-forgePatches.jar"
```

The local filename must match the library referenced by the Prism component patch:

```text
libraries/lwjgl3ify-3.0.42-forgePatches.jar
```

The relevant component is normally:

```text
patches/me.eigenraven.lwjgl3ify.forgepatches.json
```

Its library entry should reference:

```json
{
    "name": "com.github.GTNewHorizons:lwjgl3ify:3.0.42:forgePatches",
    "MMC-hint": "local"
}
```

If your instance already contains this component, you should not need to recreate it. Check that its version and local library reference match the downloaded artifact.

### B. Add lwjgl-glfw to the classpath

FreeBSD's LWJGL3 package uses the system GLFW library rather than bundling `lwjgl-glfw-natives-freebsd.jar`.

As a result, Prism Launcher may omit `lwjgl-glfw.jar` from the instance classpath.

Copy the system JAR into the instance:

```sh
mkdir -p "$INST/libraries" "$INST/patches"

cp /usr/local/share/java/classes/lwjgl3/lwjgl-glfw.jar \
   "$INST/libraries/lwjgl-glfw.jar"
```

Create a local Prism component definition:

```sh
cat > "$INST/patches/org.lwjgl.glfw.freebsd.json" << 'EOF'
{
    "formatVersion": 1,
    "libraries": [
        {
            "MMC-filename": "lwjgl-glfw.jar",
            "MMC-hint": "local",
            "name": "org.lwjgl:lwjgl-glfw:3.4.3"
        }
    ],
    "name": "LWJGL GLFW FreeBSD Fix",
    "uid": "org.lwjgl.glfw.freebsd",
    "version": "3.4.3"
}
EOF
```

Register the component in `mmc-pack.json`:

```sh
INST="$INST" python3 -c '
import json, os

p = os.path.join(os.environ["INST"], "mmc-pack.json")

with open(p, "r") as f:
    data = json.load(f)

if not any(c.get("uid") == "org.lwjgl.glfw.freebsd"
           for c in data.get("components", [])):
    data["components"].append({
        "cachedName": "LWJGL GLFW FreeBSD Fix",
        "cachedVersion": "3.4.3",
        "uid": "org.lwjgl.glfw.freebsd",
        "version": "3.4.3"
    })

with open(p, "w") as f:
    json.dump(data, f, indent=4)
'
```

## 3. Configure Prism Launcher

Reopen Prism Launcher and edit the instance settings.

These can also be configured globally if preferred.

### Java

**Java executable:**

```text
/usr/local/openjdk25/bin/java
```

**Java arguments:**

```text
-Dorg.lwjgl.librarypath=/usr/local/lib/lwjgl3:/usr/local/lib -Dorg.lwjgl.sdl.libname=/usr/local/lib/libSDL3.so
```

Do not pass `-Dos.name=Linux`.

LWJGL3 needs to retain its FreeBSD platform detection so it can use `org.lwjgl.system.freebsd.DynamicLinkLoader`.

### Native libraries

Under Tweaks → Native Libraries:

- Enable Use system GLFW:
  `/usr/local/lib/libglfw.so`
- Enable Use system OpenAL:
  `/usr/local/lib/libopenal.so`

### Wayland compatibility

For KDE Plasma 6 / KWin Wayland, set the Prism Launcher wrapper command to:

**AMD/Intel GPUs using Mesa:**

```sh
env -u WAYLAND_DISPLAY SDL_VIDEODRIVER=x11 __GLX_VENDOR_LIBRARY_NAME=mesa
```

**NVIDIA GPUs:**

```sh
env -u WAYLAND_DISPLAY SDL_VIDEODRIVER=x11
```

This forces SDL to use X11/XWayland instead of attempting to initialize its Wayland video backend.

The Mesa environment variable is intended for AMD/Intel GPUs using Mesa.

## 4. Fix LWJGLUtil platform detection

**Problem:** `Unknown platform: FREEBSD`

LWJGL3 3.4+ identifies FreeBSD as `Platform.FREEBSD`.

However, the version of `org.lwjglx.LWJGLUtil` in lwjgl3ify 3.0.42 does not handle that platform explicitly.

This results in:

```text
LinkageError: Unknown platform: FREEBSD
```

The workaround below modifies the `tableswitch` bytecode in `LWJGLUtil.class`, redirecting the default branch to `PLATFORM_LINUX`.

**Important:** This is a binary-specific workaround for the tested lwjgl3ify 3.0.42 JAR. Do not assume the byte sequence applies to other versions.

### Back up the original JAR

```sh
MOD="$INST/.minecraft/mods/lwjgl3ify-3.0.42.jar"

cp -n "$MOD" "$INST/lwjgl3ify-3.0.42.jar.bak"
```

### Apply the bytecode modification

```sh
INST="$INST" python3 -c '
import os, zipfile

inst = os.environ["INST"]

bak_path = os.path.join(inst, "lwjgl3ify-3.0.42.jar.bak")
mod_path = os.path.join(inst, ".minecraft/mods/lwjgl3ify-3.0.42.jar")

old_seq = bytes.fromhex(
    "2eaa0000000000003100000001000000030000001c000000230000002a"
)

new_seq = bytes.fromhex(
    "2eaa0000000000002300000001000000030000001c000000230000002a"
)

with zipfile.ZipFile(bak_path, "r") as zin, \
     zipfile.ZipFile(mod_path, "w") as zout:

    for item in zin.infolist():
        data = zin.read(item.filename)

        if item.filename == "org/lwjglx/LWJGLUtil.class":
            assert data.count(old_seq) == 1, \
                "Unexpected LWJGLUtil bytecode"

            data = data.replace(old_seq, new_seq, 1)

        zout.writestr(item, data)

print("Patched LWJGLUtil.class successfully.")
'
```

The original JAR is retained as:

```text
lwjgl3ify-3.0.42.jar.bak
```

If the bytecode assertion fails, stop rather than applying the modification to an unexpected binary.

## 5. Launch

Launch GTNH normally through Prism Launcher.

If everything is configured correctly, LWJGL should be able to load the system GLFW and OpenAL libraries while retaining FreeBSD platform detection.

## Limitations and acknowledgements

- This configuration is based on one tested FreeBSD setup, not a guarantee of compatibility with every GPU, desktop environment, or GTNH release.
- The bytecode workaround is tied to a specific lwjgl3ify version.
- Updating lwjgl3ify may require revisiting the classpath configuration and platform workaround.
- Prism Launcher updates or instance reimports may overwrite local component configuration.
- Keep backups of modified instance files.
- These changes are local to the instance and do not modify the upstream GTNH distribution.
- Graphics performance and rendering behavior may vary depending on the GPU and graphics stack.

## Upstream compatibility

The ideal long-term solution is native FreeBSD support in the relevant LWJGL compatibility code.

The bytecode workaround in this guide should be considered temporary. A proper upstream compatibility change would eliminate the need for users to modify the distributed JAR.

This guide is independently maintained and is not affiliated with or endorsed by the GTNH development team.

## Known issues on my setup

- The default "26.2 Minecraft bitmap rendering" results in black textures. Make sure to change it!
- Vulkan rendering results in degraded performance (softpipe/llvmpipe rendering).
