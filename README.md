# upkeep

Interactive Linux updates in one terminal session: the system package manager, AUR, Flatpak and Nix, then cached-package cleanup and reboot/service-restart checks. A single Python 3.9+ file with no dependencies.

Made with AI assistance; review it before using it.

## Install

```bash
git clone https://github.com/maia-kitty/upkeep.git
ln -s "$PWD/upkeep/upkeep" ~/.local/bin/upkeep
```

## Usage

```bash
upkeep                    # check each source, ask before upgrading it
upkeep --check            # check only; install nothing
upkeep --refresh          # refresh cached metadata first (may ask for sudo)
upkeep --clean            # only cached-package cleanup; add --check to preview
upkeep --maintenance      # only reboot/service checks; add --check to just report
upkeep --backend none     # skip the system manager: only Flatpak and Nix
```

Skip parts with `--no-system`, `--no-aur`, `--no-flatpak`, `--no-nix`, `--no-fwupd`, `--no-cleanup` or `--no-maintenance`. `--verbose` shows the commands and full output, and `--timeout` sets the seconds per check (default 180). A failed check skips that source, the others still run, and the exit status is nonzero.

Run it as your normal user: only system commands go through `sudo` or `doas`. Upgrades and cleanups ask `[Y/n]`, a reboot asks `[y/N]`, and the package managers keep their own prompts. Ctrl+C goes to the running package manager; upkeep waits for it to stop, then cancels the session.

## Sources

| Source | When | Upgrade |
| --- | --- | --- |
| pacman | Arch, CachyOS, Manjaro, EndeavourOS | `pacman -Syu`, checked with `checkupdates` from `pacman-contrib` (never `pacman -Sy`) |
| APT | Debian, Ubuntu, Mint, Pop!_OS | `apt-get upgrade` |
| DNF | Fedora, RHEL and clones | `dnf upgrade` |
| Zypper | openSUSE | `zypper update`; `zypper dup` on Tumbleweed/Slowroll |
| XBPS | Void | `xbps-install -Su`; run again if XBPS updated itself |
| APK | Alpine | `apk upgrade` |
| AUR | Arch, with `paru` or `yay` | `<helper> -Sua`; skipped if the system wasn't updated in the same run (including `--no-system`) |
| Flatpak | when installed | `flatpak update` |
| Nix | a user profile exists | `nix profile upgrade --all` (Nix 2.22+), or `nix-env --upgrade` for legacy profiles |
| Firmware | `fwupdmgr` is installed | `fwupdmgr update`; fwupd asks for authorisation through polkit and offers its own reboot |

The system manager comes from `/etc/os-release`; pick one with `--backend`. APT, Zypper, APK, legacy Nix and fwupd check against cached metadata unless you pass `--refresh`. For firmware between interactive runs, enable `fwupd-refresh.timer`: until the LVFS metadata has been downloaded once, fwupd reports no updates at all. The Nix profile defaults to `~/.nix-profile`, else `$XDG_STATE_HOME/nix/profiles/profile`; change it with `--nix-profile`. Modern Nix only upgrades packages from unlocked flake references.

Not covered: immutable systems (ostree), NixOS rebuilds, Home Manager, root Nix profiles.

## Cleanup

After the updates, each cleanup gets its own prompt:

| Manager | Cleanup |
| --- | --- |
| pacman | `paccache` keeps 3 versions and drops archives of uninstalled packages (`pacman -Sc` without `pacman-contrib`). Also lists orphaned packages to pick for `pacman -Rns`. |
| APT / DNF / Zypper | `apt-get autoclean` / `dnf clean packages` / `zypper clean` |
| XBPS / APK | `xbps-remove -OO` / `apk cache clean` |
| AUR | `<helper> -Sc --aur`, with the helper's own prompts |
| Nix | `nix-store --gc`; keeps every profile generation |
| Flatpak | `flatpak uninstall --unused` for each installation; Flatpak lists the refs and asks again |

## After updates

After a system update, upkeep:

- lists `.pacnew`/`.pacsave` files to merge (Arch, with `pacdiff`)
- checks for reboot markers and a running kernel that was removed or replaced
- finds services that need a restart with `needrestart`, `checkservices`, `needs-restarting` or Void's `xcheckrestart`, and restarts the ones you pick by number

Display managers and session services are never restarted; they're flagged for a logout or reboot. Plain `--check` skips the service scan, because it needs sudo.

## Status bars

`upkeep --json` is read-only: no sudo, refreshes, cleanup or installs. It prints a report (`"version": 1`) and writes it atomically to `~/.cache/upkeep/status.json`. A check it can't count shows as unknown, never as zero. `detected` lists every source found on the machine, including ones switched off with `--no-*`, so a bar can offer toggles for exactly those. Interactive runs touch `last-session` in the same folder when they finish, so a bar knows when to recheck.

`upkeep --launch` opens the updater (`--refresh --wait`) in a detached Foot window titled "Upkeep", or in your default terminal through `xdg-terminal-exec`, and passes on any `--no-*` flags. Reloading the bar can't interrupt a transaction.
