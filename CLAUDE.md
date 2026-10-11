# CLAUDE.md

Guidance for Claude Code (and other agents) working in this repository.

## Repo and ownership

- This is **RealRedcodi/finder-menu**, a fork of [nikgrbn/finder-menu](https://github.com/nikgrbn/finder-menu). The fork owner is **Redcodi**.
- Upstream is still referenced on purpose in `LICENSE` (MIT copyright notice, keep it), and currently also in `README.md` (release badge, Homebrew tap `nikgrbn/tap`, clone URL) and `SECURITY.md` (advisory link). Those README/SECURITY links install and report to **upstream**, not this fork. Only change them if Redcodi asks.
- To sync with upstream: `git fetch https://github.com/nikgrbn/finder-menu.git main` then merge. Last synced at upstream `299ccaf` (v1.4.0).
- Agent work goes on a feature branch, never straight to `main`.

## What it is

A macOS Finder right-click menu add-on: New File… (with saved templates), Open Terminal Here, Open Folder With ▸, Copy Path, Go to Parent Folder. Swift 5, AppKit, no Xcode project, no SwiftPM, no third-party dependencies, no network code. Targets macOS 13+. Version lives in `Resources/App-Info.plist` and `Resources/Extension-Info.plist` (`CFBundleShortVersionString` and `CFBundleVersion`, bump both files together).

## Architecture

Two binaries in one bundle, `Finder Menu.app`:

| Part | Path | Notes |
|---|---|---|
| Finder Sync extension | `Sources/FinderMenuExtension/FinderSync.swift` | Sandboxed (`Resources/Extension.entitlements`). Builds the menu, copies paths, and forwards every other action to the helper as a `findermenu://<action>?path=…&app=…&template=…` URL. Principal class is `@objc(FinderMenuSync)`, matching `NSExtensionPrincipalClass`. |
| Helper app | `Sources/FinderMenu/main.swift` | Not sandboxed, `LSUIElement`. Handles the URLs (newfile, terminal, openwith, up, settings), then quits. Opened directly, it shows the settings window and acts like a regular app. |
| Settings window | `Sources/FinderMenu/SettingsWindow.swift` | Programmatic AppKit (stack views, no nibs). |
| Terminal list | `Sources/FinderMenu/Terminals.swift` | Bundle id + launch method (`openFolder`, URL scheme, or CLI args). Order = auto-pick priority. |
| Shared (compiled into both) | `Sources/Shared/` | `Settings.swift` (UserDefaults domain `local.findermenu`, menu item ids), `Templates.swift` (New File templates, filename validation), `Editors.swift` (Open Folder With allowlist). |

Settings: the helper writes the `local.findermenu` defaults domain; the extension reads it through the `shared-preference.read-only` entitlement exception. Keys: `Hide.<item-id>`, `Terminal`, `HideOnOtherDrives`, `FileTemplates` (array of `{id, filename}`).

### Security invariants (do not weaken)

- The helper accepts `findermenu://` links only when the Apple event sender's code signature matches the **cdhash of the bundled extension** (`sentByOurExtension()` in `main.swift`). Fails closed.
- Paths must be absolute and exist; packages resolve to their parent folder (`folder(for:)`).
- `openwith` only accepts bundle ids from `editors` (or none, which shows a picker the user drives).
- Template file names never travel in the URL; only the template id does.
- New files use `.withoutOverwriting` (O_EXCL). Never add an overwrite path.
- AppleScript strings go through the `lit()` escaper in `goUp`. Never interpolate raw paths into AppleScript or a shell.
- No networking code. The README advertises "network access: none".

## Build, install, run

macOS only, needs Xcode Command Line Tools (`xcrun swiftc`, `codesign`). Nothing builds or runs on Linux, so in a cloud/Linux session you can only read and edit; say so instead of claiming a build passed.

```bash
./install.sh                         # build to a temp dir, copy to ~/Applications, enable extension
scripts/build.sh <out-dir>           # compile + ad-hoc sign only
scripts/finder-menu enable [app]     # install/enable (no arg = Homebrew layout)
scripts/finder-menu disable          # same as ./uninstall.sh
scripts/finder-menu terminal [name|auto]
scripts/finder-menu settings
swift scripts/make-icon.swift        # regenerate Resources/AppIcon.icns and docs/icon.png
swift scripts/make-mockups.swift     # render docs/mockups/mockups.html to docs/images/*.png
```

`build.sh` compiles each target with a single `swiftc` call over `Sources/<Target>/*.swift Sources/Shared/*.swift`, so new files are picked up automatically. Build output must stay outside the repo (`install.sh` uses `mktemp -d`; `build/` is gitignored) so macOS never registers a second copy of the extension.

There are no tests and no CI. Verify changes by building and right-clicking in Finder on a Mac.

## Localization

- `Resources/<lang>.lproj/Localizable.strings` and `InfoPlist.strings`; languages: en, de, it, ru, uk. `build.sh` copies every `.lproj` into both the app and the appex.
- Keys are the English strings passed to `NSLocalizedString` (plus the `menuItems` titles in `Settings.swift`, which are localized through a variable). Every key must exist in every language, with matching `%@` counts.
- Adding a language also means adding it to `CFBundleLocalizations` in **both** Info.plists and to the README Languages list.

## Conventions

- Code style: small files, top-level `let` tables, comments that explain *why* (Finder quirks, sandbox limits). Match that density.
- Commit messages: short imperative sentence describing the user-visible effect (see `git log`).
- README "About 900 lines of Swift and 170 lines of shell" is a real count (`Sources/` and `install.sh`, `uninstall.sh`, `scripts/finder-menu`, `scripts/build.sh`); update it if a change moves it noticeably.
- README images are rendered mockups, not screenshots. Edit `docs/mockups/mockups.html` and re-render.

## Audit

The latest repository audit and its open findings are in `docs/AUDIT.md`.
