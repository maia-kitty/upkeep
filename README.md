# upkeep

Interactive updates across Linux package managers in one terminal session: the system package manager, AUR helpers, Flatpak and Nix, followed by cached-package cleanup and reboot/service-restart checks. A single Python file with no dependencies.

Made with AI assistance; review it before using it.

## Install

```bash
git clone https://github.com/maia-kitty/upkeep.git
ln -s "$PWD/upkeep/upkeep" ~/.local/bin/upkeep
```

## Usage


Requires Python 3.9+, no other dependencies. Run:

```bash
upkeep                    # Check each source, ask before upgrading it
upkeep --check            # Check only; no package installations
upkeep --check --refresh  # Also refresh cached metadata/channels
upkeep --backend none     # Only AUR-independent sources: Flatpak and Nix
upkeep --clean            # Only offer cached-package cleanup
upkeep --clean --check    # Preview cleanup without removing anything
```

It selects the system manager from `/etc/os-release`: pacman for Arch/CachyOS, APT for Debian/Ubuntu, DNF for Fedora/RHEL, Zypper for openSUSE, XBPS for Void, and APK for Alpine. Arch checks require `checkupdates` from `pacman-contrib`; they use a temporary database to avoid a partial system upgrade. Tumbleweed/Slowroll use `zypper dup`. Void may need a second run when XBPS updates itself. Immutable and declaratively managed operating systems need their own system update workflows; NixOS system rebuilds are not included. Use `--backend none` on those systems or choose a supported backend explicitly.

It also checks AUR packages with `paru` or `yay` on Arch, Flatpak when installed, and Nix user packages. Nix profiles with `manifest.json` use `nix profile upgrade --all --dry-run --refresh` (Nix 2.22+); legacy profiles with `manifest.nix` use `nix-env --upgrade --dry-run`. The default profile is `~/.nix-profile`, falling back to `$XDG_STATE_HOME/nix/profiles/profile` (normally `~/.local/state/nix/profiles/profile`). Choose another with `--nix-profile /path/to/profile`. Modern Nix only upgrades packages installed from unlocked flake references; pinned packages stay pinned. Legacy Nix checks use the current user's channels; `--refresh` updates those channels first. Home Manager, project flake locks, root profiles, and the Nix daemon are managed separately. See the [Nix profile upgrade manual](https://nix.dev/manual/nix/2.35/command-ref/new-cli/nix3-profile-upgrade.html).

APT, Zypper, APK, and legacy Nix checks use cached metadata unless `--refresh` is supplied. Other checks can contact repositories and populate caches, but do not install packages. Refreshing system metadata may require a password. Run the helper as your normal user: only system refresh/upgrade commands use `sudo` or `doas`; Nix and AUR commands run as you. Package managers retain their own transaction prompts. Failed checks skip that source, continue checking the others, and return a nonzero exit status; check timeouts default to 180 seconds and can be changed with `--timeout`. Use `--no-nix`, `--no-aur`, or `--no-flatpak` to skip sources.

Installation prompts default to yes (`[Y/n]`): press Enter to install updates. Reboot prompts default to no. The terminal interface uses your terminal's own color palette, clear sections, package/version columns, and compact Nix flake summaries. `--verbose` shows the underlying commands and full check output. Color is disabled when output is piped, `NO_COLOR` is set, or `TERM=dumb`.

After an interactive system update session, upkeep checks for reboot markers and a running kernel that has been removed or replaced on disk. It uses an installed `needrestart` (batch/list mode), `checkservices` (with reloads, config processing, and restarts disabled), or standalone `needs-restarting` to find services that need restarting. On runit, `xcheckrestart` from Void's `xtools-minimal` package is another fallback: upkeep links outdated process IDs to enabled runit services through their supervisors, and reports other outdated processes for manual restart or logout. Scans may request your sudo/doas password. Choose numbered services, `0`, or `all` to restart them, or press Enter to skip; systemd service definitions are reloaded before selected services restart. Display managers and core session services are excluded and flagged for logout/reboot. On other init systems, `needrestart` results can use OpenRC, runit, or SysV restart commands when available. A recommended reboot always gets a separate confirmation prompt. If no detector is installed or scanning fails, the helper reports that limitation.

Run `upkeep --maintenance` to do these checks without checking packages, or `upkeep --maintenance --check` to report findings without restarting services or rebooting. Ordinary `--check` only checks reboot markers/kernel files; it does not run privileged service scans. Use `--no-maintenance` to skip all reboot/service checks. The reboot hints are best effort; retained kernel files, containers, or an unavailable detector can limit detection. See [needrestart's batch mode documentation](https://github.com/liske/needrestart/blob/master/README.batch.md) for the service and kernel findings it reports.

The **Cached packages** step runs after package checks/upgrades and before reboot/service checks. Each cleanup gets a separate `[Y/n]` prompt. `--check` only previews supported cleanup operations or prints the proposed command when the manager has no dry run. `--no-cleanup` skips this step; `--clean` runs it without package updates or reboot/service actions. Failed previews skip that cleanup, and failures do not block the remaining managers.

| Manager | Cleanup | What is removed or retained |
| --- | --- | --- |
| pacman | `paccache -r -k 3`, then `paccache -r -u -k 0` | Keep 3 cached versions; remove all archives of uninstalled packages. Without `paccache`, fall back to `pacman -Sc`, which follows pacman's cache policy and also offers to remove unused sync databases. |
| APT | `apt-get autoclean` | Remove obsolete downloaded packages no longer available from repositories. |
| DNF | `dnf clean packages` | Remove downloaded package archives; retain repository metadata. |
| Zypper | `zypper clean` | Remove downloaded package archives; retain repository metadata. |
| XBPS | `xbps-remove -OO` | Remove outdated archives and archives of uninstalled packages. |
| APK | `apk cache clean` | Remove obsolete or no-longer-needed archives from the configured cache. |
| paru / yay | `<helper> -Sc --aur` | Clean AUR build caches and sources with the helper's own prompts; system cache cleanup stays separate. |
| Nix | `nix-store --gc` | Collect unreachable store objects while preserving all profile generations and rollback roots. Runs as your user, including when no supported user profile exists. |
| Flatpak | `flatpak repair` for each discovered installation | Prune unreferenced store objects and verify the installation. Damaged objects may be downloaded again; installed applications and runtimes are retained. User, default system, and custom installations containing installed refs are supported. |

Nix cleanup uses `--gc --print-dead` for a read-only preview and never passes generation-deletion flags; see the [Nix garbage collection manual](https://nix.dev/manual/nix/2.21/command-ref/nix-collect-garbage). Flatpak uses `repair --dry-run` before offering the cleanup; see the [Flatpak repair reference](https://manpages.debian.org/unstable/flatpak/flatpak-repair.1.en.html). Cache removal can require downloading archives or rebuilding AUR packages again in future.

This helper does not enable a tray app or scheduled checks or remove installed package orphans automatically.

## For status bars

`upkeep --json` is strictly read-only: it checks packages and reboot hints, never invokes sudo/doas, refreshes system indexes or legacy Nix channels, cleans caches, restarts services, or installs packages. It prints a report (`"version": 1`) and also writes it atomically to `$XDG_CACHE_HOME/upkeep/status.json` (normally `~/.cache/upkeep/status.json`). Interactive runs touch `last-session` in the same folder when they finish, so a bar can recheck afterwards. Failed or unrecognized checks report an error or unknown count instead of zero.

`upkeep --launch` opens the interactive updater in a detached Foot window titled "Upkeep" (`--refresh --wait`), so closing or reloading the bar cannot interrupt a package transaction.
