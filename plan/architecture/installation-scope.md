# Installation Scope: Per-User and Machine-Wide

## 1. Purpose

PackageX provides one cross-platform installation-scope policy while allowing each native installer adapter to implement it using the target operating system's real installation model.

- **Per-user:** installed for and available to the account performing installation. It should not require administrator/root privileges unless a separately declared operation requires them.
- **Machine-wide:** installed in a machine-managed location and intended to be available to all local accounts. It normally requires administrator/root authorization.
- **Choice:** an installer may offer both scopes when the selected format and every requested feature support both. The developer selects the default choice.

Scope is not the same as elevation. Scope expresses who receives the application. `elevate` controls whether privileged execution is permitted or required to perform selected operations.

## 2. .px policy

The resolved package policy is:

~~~toml
[package]
scope = "user"                  # "user" | "machine" | "choice"
scope_default = "user"          # required only when scope = "choice"
elevate = "auto"                # true | false | "auto"
~~~

Rules:

- `user` installs only for the invoking account.
- `machine` installs to a shared machine location and performs only machine-scoped registrations.
- `choice` exposes both options only when the selected target-format capability declares both choices. `scope_default` preselects one; it does not silently force that selection.
- The default scope is `user` when omitted.
- A developer may lock scope by selecting `user` or `machine`.
- Scope changes during an update are a migration, not a routine update. A per-user install must not silently become machine-wide, or vice versa.

### Scope and elevation are separate

| Scope | Elevation | Validity |
|---|---|---|
| user | false | Normal per-user install |
| user | auto | No elevation unless a separately requested operation needs it; such a conflict is diagnosed |
| user | true | Elevated execution may be used, but files and registrations remain per-user |
| machine | auto | Request native administrator/root authorization when required |
| machine | true | Explicit machine-wide privileged install |
| machine | false | Invalid when any machine-level write/registration requires privilege; fail during `packx check` |
| choice | auto | Prompt for scope, then resolve privileges for the chosen scope |
| choice | true/false | Validate each offered choice; do not offer one that contradicts the fixed privilege policy |

A failed or denied machine-wide authorization must not silently fall back to per-user. The installer reports the required privilege and exits without partial writes.

## 3. Path and state resolution

The Engine resolves logical tokens using selected scope. The SDK receives resolved logical destinations and target capabilities; it must not infer scope from a hard-coded path string.

| Purpose | Per-user examples | Machine-wide examples |
|---|---|---|
| Windows app files | `%LOCALAPPDATA%\\Programs\\<app>` or developer-selected user path | `%ProgramFiles%\\<app>` |
| Windows app support/updater data | `%LOCALAPPDATA%\\<publisher>\\<app>` | `%ProgramData%\\<publisher>\\<app>` |
| macOS app bundle | `~/Applications/<app>.app` | `/Applications/<app>.app` |
| macOS shared support | `~/Library/Application Support/<app>` | `/Library/Application Support/<app>` |
| Linux app files | `~/.local/opt/<app>` or AppImage location selected by user | `/opt/<app>`, `/usr/local/<app>` or native package-manager paths |
| Linux app data | `$XDG_DATA_HOME/<app>` (normally `~/.local/share/<app>`) | `/var/lib/<app>` |
| User scheduler | Current account scheduler/config | Not used for machine updates |
| Machine scheduler | Not used | Machine task/daemon/timer with protected configuration |

These are examples, not universal path substitutions. The target adapter and format conventions take precedence. App-owned mutable data, logs, caches, preferences and user documents remain in user-writable data locations even when the executable is installed machine-wide, unless the developer separately defines shared data.

Machine-wide installation must protect updater binaries, configuration, runtime reference data and staging directories from modification by ordinary users. An elevated updater must never load update instructions or executables from a user-writable directory.

## 4. Native format capability matrix

A cross-platform `scope` field does not make every format support both scopes. The Engine requests scope support from the SDK capability handshake. Unsupported combinations fail before artifact emission unless the developer explicitly selected an alternative output format.

| Platform / format | Per-user | Machine-wide | Native behavior and restriction |
|---|---|---|---|
| Windows custom EXE | Supported candidate | Supported candidate | PackageX helper can choose user or common paths; machine scope requires administrator authorization and protected updater/uninstaller locations. |
| Windows MSI | Supported | Supported | Use Windows Installer installation context. A dual-purpose MSI must be authored/tested for both contexts using the supported `ALLUSERS` / `MSIINSTALLPERUSER` model. |
| Windows MSIX | Per-user registration | Enterprise provisioning path | Package staging is machine-managed but registration is per-user. Provisioning for all users is a distinct administrator/IT deployment operation, not equivalent to a conventional per-machine MSI install. |
| macOS DMG | Supported | Supported with authorized copy to `/Applications` | Drag-copy to `~/Applications` needs no admin; copying to `/Applications` requires appropriate authorization. A DMG does not automatically execute an embedded helper on drag-and-drop. |
| macOS PKG | Conditional | Supported | Use package install domains/choices where authored and supported. `LocalSystem` installation generally requires administrator authorization; `CurrentUserHomeDirectory` is a per-user domain. Verify the actual package's declared domains and scripts. |
| Linux DEB | Not native | Supported | `dpkg`/APT package installation is system-managed. Do not claim a conventional per-user DEB installation. |
| Linux RPM | Not native | Supported | RPM/DNF/Zypper package installation is system-managed. Do not claim a conventional per-user RPM installation. |
| Linux AppImage | Supported | Conditional | The usual model is a user-owned executable image. An admin may place it in a common directory, but that is not equivalent to native package-manager registration. |
| Linux Flatpak | Supported | Supported | Flatpak has explicit per-user and system installations; use its native `--user` / `--system` semantics and repository scope. |

### Important MSIX distinction

MSIX package deployment separates machine staging from user registration. Standard MSIX app installation is per-user. An administrator or enterprise deployment system can provision a package for all users, after which each user registers it. PackageX must report this as `machine_provisioned` or equivalent native deployment behavior, not mislabel it as the same install context as MSI.

### Important macOS distinction

A `.app` can be placed under `~/Applications` for one user or `/Applications` for all users. A `.pkg` may support installation domains such as `CurrentUserHomeDirectory` and `LocalSystem`, but the package must be authored to support the chosen domain. A format label alone is not proof of per-user support.

### Linux distinction

DEB/RPM are OS-level package-manager operations, normally requiring root and registering system files/state. Per-user installs should use a format that genuinely supports them, such as Flatpak `--user` or a user-owned AppImage/portable directory. Do not unpack a DEB/RPM manually into a home directory and call it a native per-user installation.

## 5. Capability handshake and compile-time validation

For each output target, SDK capabilities must declare:

- Supported scope values: `user`, `machine` and optionally `choice`.
- Whether the native format distinguishes staging, provisioning and per-user registration.
- Required privilege for each operation.
- Whether custom installer helper execution is supported.
- Whether an Electron app-only runtime restoration plan is supported.
- Native data, shortcut, service, updater and uninstaller registration scope.
- Format restrictions that prevent a requested scope or operation.

`packx check` validates every selected target and step. If a build requests `choice` but a format can only install system-wide, the build fails rather than silently forcing machine scope. If a multi-format build includes formats with different scope capabilities, the build report names the discrepancy.

A package may expose only the intersection of selected target capabilities as a shared scope choice, or the Engine may require separate target-specific builds. It must not imply that one installer behaves identically across all formats.

## 6. Updater, uninstaller and service implications

- Per-user applications register per-user shortcuts, PATH entries, schedulers and uninstall records.
- Machine-wide applications register shared shortcuts and machine-level uninstall metadata where supported.
- Per-user update schedulers run as that user and write only to that user's protected install.
- Machine-wide schedulers/services run under a protected system/service identity, use machine-protected configuration and write only to machine-owned locations.
- A machine updater must verify feed signatures and artifact digests before elevated writes. It must not execute user-writable custom scripts as administrator.
- Uninstall scope follows the original installation record. An ordinary user cannot remove a machine-wide install without appropriate authorization.
- User data must not be deleted merely because machine-wide binaries are removed.

## 7. Electron interaction

Electron runtime files follow the application's install scope and live inside that application's installed generation. They are not placed into a system-wide shared Electron cache.

- Per-user app: runtime is installed under the user's application root with the app.
- Machine-wide app: runtime is installed under the protected machine application root with the app.
- During updates, the installed generation retains its required runtime while one next runtime is staged for the next release. The updater commits app + runtime together after the running app exits and retains the old generation until the new one passes health checks.
- The Engine and SDK must still honor native format restrictions: app-only runtime restoration cannot mutate signed MSIX packages or sealed macOS app bundles after signing.

## 8. Verification references

- [Windows Installer ALLUSERS property](https://learn.microsoft.com/en-us/windows/win32/msi/allusers)
- [Windows Installer installation context](https://learn.microsoft.com/en-us/windows/win32/msi/installation-context)
- [MSIX deployment architecture](https://learn.microsoft.com/en-us/windows/apps/windows-app-sdk/deployment-architecture)
- [MSIX packaged app behavior](https://learn.microsoft.com/en-us/windows/msix/desktop/desktop-to-uwp-behind-the-scenes)
- [Apple application file locations](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPFileSystem/Articles/WhereToPutFiles.html)
- [macOS installer domains](https://manpagez.com/man/8/installer/)
- [Flatpak system and user installation](https://docs.flatpak.org/en/latest/using-flatpak.html)
- [Debian Policy: maintainer scripts](https://www.debian.org/doc/debian-policy/ch-maintainerscripts)

## 9. Acceptance criteria

- [ ] .px exposes explicit per-user, machine-wide and optional scope-choice policies.
- [ ] Scope remains separate from elevation and install directory.
- [ ] Per-user is the default; machine-wide is explicit.
- [ ] A denied machine authorization never silently degrades to user scope.
- [ ] The SDK declares native scope capability for every target format.
- [ ] MSI is tested in both installation contexts.
- [ ] MSIX staging/provisioning/registration are represented distinctly.
- [ ] macOS user/system install locations and PKG domains are validated per package.
- [ ] DEB/RPM are not incorrectly represented as native per-user installs.
- [ ] Flatpak user/system and AppImage user/common placement are distinguished.
- [ ] Scheduler, updater, uninstaller, PATH, service and data ownership follow selected scope.
- [ ] Multi-format builds report scope capability differences without silent downgrades.
