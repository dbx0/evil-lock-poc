# Evil Lock

A themed session lock screen for Omarchy.

> **Security proof-of-concept.** This repository demonstrates a bug reported to the Omarchy project. It is a working exploit, so run it only on a machine you own or are authorized to test, and never install it on anyone else's system.

## What it demonstrates

Omarchy's shell gives one capability, `authentication`, to the code allowed to see your plaintext login password: the first-party lock plugin. Third-party plugins are never meant to receive it, and `PluginRegistry.trustedCapabilities()` even refuses to read a third-party plugin's own declared capabilities.

This plugin declares no capabilities at all. Its manifest contains only:

```json
"omarchy": { "clonedFrom": "omarchy.lock" }
```

`PluginRegistry.stampHostCapabilities()` copies the first-party lock plugin's capabilities onto any plugin that names it in `clonedFrom`, with no provenance check. So the shell loads this plugin as the session's authentication service and hands it the real lockscreen and PAM flow, and it sees the password you type. The only added code captures that password and uses it with `sudo` to open a terminal that is already a root shell. Everything else is the real Omarchy lock plugin.

`clonedFrom` is a self-asserted string in a user-writable manifest. `omarchy plugin validate` never inspects it, and the `omarchy plugin add` warning is the generic "plugins run unsandboxed" notice, so nothing at install time tells you this plugin will become the authentication service.

## Install (on a test machine only)

Two ways in, both ending in the same place.

From the terminal:

```bash
omarchy plugin add https://github.com/dbx0/evil-lock-poc.git --enable
```

From the Omarchy menu:

1. Open the menu with SUPER + SPACE.
2. Go to Setup > Plugins > Add Plugin (use Add Plugin, not Clone Plugin, which opens an editor instead).
3. A floating terminal opens. Paste the repo URL at the "Git URL of the plugin repo:" prompt and press Enter.
4. Answer yes to "Clone and add this plugin?", then yes to "Enable 'evil.lock' now?".

Either way, enabling the clone is the whole attack. `setEnabled()` in the shell registry adds `omarchy.lock` to `disabledPlugins` automatically, so the real lock is switched off without you touching it. The lock service is `keepLoaded`, so the clone takes over on the **next boot or login**: the shell starts fresh, loads this clone as the only lock service, and from then on every lock (manual or on idle) captures the password and opens a root terminal while the unlock completes as normal.

So the full flow is: install and enable, reboot, log back in, lock the screen (SUPER + CTRL + L or wait for the idle timeout), type your password, and a terminal opens sitting at a `root#` prompt.

To activate it in the current session without rebooting, run `omarchy-restart-shell` once, then lock.

## Clean up / uninstall

```bash
omarchy plugin enable omarchy.lock
omarchy plugin disable evil.lock
omarchy plugin remove evil.lock
omarchy-restart-shell
```

The payload writes the captured password to a `0600` temp file under `$XDG_RUNTIME_DIR`, uses it once, and deletes it along with its own launcher script when the root terminal is closed.

## Provenance

`Service.qml` and `LockView.qml` are copies of Omarchy's own `shell/plugins/lock/` files, so the clone behaves identically, with a small `capturePassword()` hook added to `Service.qml`. Those files belong to the Omarchy project and carry its license; they are included here only to make the disclosure reproducible.
