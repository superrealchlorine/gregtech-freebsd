# GT: New Horizons on FreeBSD

A community-maintained guide for running **GregTech: New Horizons (GT:NH)** on **FreeBSD 15.x** using Prism Launcher, LWJGL3, and Java 25.

Tested configuration:
- FreeBSD 15.x
- Prism Launcher
- KDE Plasma 6 / KWin (Wayland)
- Mesa graphics drivers
- OpenJDK 25
- GTNH with lwjgl3ify 3.0.42

This guide documents the local compatibility adjustments required to get the game running. It is not an official GTNH project.

## 1. Install dependencies

```sh
pkg install prismlauncher openjdk25 lwjgl3 glfw openal-soft sdl3
```

## 2. Import the GTNH instance

1. Download the Java 17+ Prism/MultiMC instance archive for GTNH.
2. Import it into Prism Launcher.
3. Close Prism Launcher completely before modifying instance files.

Set an instance variable for convenience:

```sh
INST="$HOME/.local/share/PrismLauncher/instances/GT-NH"
```

Adjust this path if your instance has a different name or location.

### A. Fix RetroFuturaBootstrap (forgePatches) resolution

The FreeBSD Prism Launcher build may fail to resolve the `forgePatches` Maven artifact automatically.

Copy the matching `gtnh-rfb` JAR into the instance's libraries directory.

For example, for version 3.0.42:

```sh
mkdir -p "$INST/libraries"
```

Place the matching `gtnh-rfb-3.0.42.jar` inside:

```text
$INST/libraries/
```

Use the version corresponding to your installed lwjgl3ify configuration.

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

```sh
env -u WAYLAND_DISPLAY SDL_VIDEODRIVER=x11 __GLX_VENDOR_LIBRARY_NAME=mesa
```

This forces SDL to use X11/XWayland instead of attempting to initialize its Wayland video backend.

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

## Notes and limitations

- This configuration is based on one tested FreeBSD setup, not a guarantee of compatibility with every GPU, desktop environment, or GTNH release.
- The bytecode workaround is tied to a specific lwjgl3ify version.
- Updating lwjgl3ify may require revisiting the classpath configuration and platform workaround.
- Prism Launcher updates or instance reimports may overwrite local component configuration.
- Keep backups of modified instance files.
- These changes are local to the instance and do not modify the upstream GTNH distribution.

## Upstream compatibility

The ideal long-term solution is native FreeBSD support in the relevant LWJGL compatibility code.

The bytecode workaround in this guide should be considered temporary. A proper upstream compatibility change would eliminate the need for users to modify the distributed JAR.

This guide is independently maintained and is not affiliated with or endorsed by the GTNH development team.
