# Vodka Launcher — "can't detect Mocktail" on NixOS

Paste the section below to Claude on the main PC (where the source lives).

Environment where this was diagnosed:
- NixOS, Linux 6.18.64
- Flatpak 1.18.4, `space.bigrat.mocktail` 1.0.4 from flathub (installed in BOTH user and system scope)
- `vodka-launcher` installed via Nix (`nix run github:AriCuteGirl/vodka-launcher`), run as an AppImage

---

## Prompt for Claude

Vodka Launcher fails on NixOS with "can't detect Mocktail" even though `space.bigrat.mocktail` 1.0.4 is installed from Flathub.

**Root cause (already diagnosed and confirmed, don't re-investigate):**

linuxdeploy's `AppRun.wrapped` exports `LD_LIBRARY_PATH=$APPDIR/usr/lib/:...` for the whole process. When the launcher spawns the host `flatpak` binary, the child inherits that var. On NixOS `flatpak` resolves its deps via `DT_RUNPATH`, and `LD_LIBRARY_PATH` takes precedence over `DT_RUNPATH` — so flatpak loads the AppImage's bundled glib/crypto/curl and then dies on the first library the bundle doesn't ship:

```
flatpak: error while loading shared libraries: libz.so.1: cannot open shared object file
```

flatpak exits non-zero with no parseable stdout, so the detection code concludes Mocktail isn't installed. Verified directly:

```
LD_LIBRARY_PATH set   -> flatpak: error loading libz.so.1
LD_LIBRARY_PATH unset -> space.bigrat.mocktail	1.0.4
```

It works on normal distros only because their `/usr/lib` happens to supply the missing libs. NixOS has no `/usr/lib`. Bundling the missing libs is NOT the fix — it cascades (`libz.so.1`, then `libcom_err.so.2`, ...). The env var is the bug.

**What I want you to do:**

1. Add one helper for spawning host commands that clears the AppImage-injected env, and route **every** host subprocess spawn through it — not just the detection probe. It must also cover `flatpak run ... space.bigrat.mocktail`, the updater spawn (`--command=/app/lib/mocktail/mocktail_updater`), and any other `Command::new` that execs a host binary:

```rust
cmd.env_remove("LD_LIBRARY_PATH")
   .env_remove("GIO_MODULE_DIR")
   .env_remove("GTK_PATH")
   .env_remove("GI_TYPELIB_PATH");
```

Find all host-command spawn sites and convert them. Note the distinction: env vars meant *for* Mocktail (`MOCKTAIL_*`) must still be passed through — only the AppImage's own loader/GTK vars get stripped.

2. Fix the misleading errors. Right now both "Flatpak is not installed, so Vodka can't find Mocktail" and "Mocktail is not installed" fire on *any* flatpak failure, including this one. Distinguish three cases: flatpak binary genuinely absent (ENOENT on spawn) vs. flatpak ran but exited non-zero (surface its exit code and stderr) vs. flatpak succeeded and Mocktail simply wasn't in the list.

3. Handle duplicate rows. Mocktail can be installed in both user and system scope, so `flatpak list --app --columns=application,version` returns the same app twice. Make sure detection and version parsing dedupe rather than tripping over the second row.

Show me the diff before applying.

---

## Appendix: how to reproduce on a NixOS box

```bash
D=~/.cache/appimage-run/*/  # extracted Vodka AppDir

# fails
env LD_LIBRARY_PATH="$D/usr/lib/" flatpak list --app --columns=application,version

# works
env -u LD_LIBRARY_PATH flatpak list --app --columns=application,version
```

## Appendix: local stopgap (not a real fix)

A shim at `~/.cache/appimage-run/<sha>/usr/bin/flatpak` that strips the var before exec'ing the real
binary. Works because `AppRun` puts `$APPDIR/usr/bin` first on `PATH`.

```bash
#!/usr/bin/env bash
exec env -u LD_LIBRARY_PATH -u GIO_MODULE_DIR -u GTK_PATH -u GI_TYPELIB_PATH /run/current-system/sw/bin/flatpak "$@"
```

Caveats: wiped by re-extraction, and bypassed entirely by `--appimage-extract-and-run` (that flag
extracts fresh to a temp dir), so the launcher must be started as plain `vodka-launcher`.
