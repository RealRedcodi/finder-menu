# Repository audit

- **Date:** 2026-10-11
- **Commit audited:** `299ccaf` (v1.4.0, build 7), identical to upstream `nikgrbn/finder-menu` `main`
- **Scope:** all Swift sources, shell scripts, Info.plists, entitlements, localizations, README, SECURITY.md, docs tooling
- **Limits:** done on Linux, so nothing was compiled or run. Findings marked *verify on a Mac* are from reading the code.

## Summary

The codebase is small (about 895 lines of Swift, 170 of shell), dependency-free and carefully written. The security design holds up: no high-severity issues were found. The findings below are edge-case bugs, UX surprises and project hygiene.

| # | Severity | Area | Finding |
|---|---|---|---|
| 1 | Medium | Templates | Dotfile templates may get odd duplicate names (`.env` → `.env 2.env`) *(verify on a Mac)* |
| 2 | Low | Go to Parent Folder | Moves an unrelated Finder window when no window shows the clicked folder |
| 3 | Low | Package contents | Inside "Show Package Contents", actions run in the folder *containing* the package |
| 4 | Low | Helper launch | 0.5 s timer can open the settings window during a slow cold launch from a menu action |
| 5 | Low | New File | Every non-"exists" write error is reported as "no permission?" |
| 6 | Low | `finder-menu` | `restart_extension` spins ~10 s and warns when Finder isn't running |
| 7 | Info | Helper | Multi-URL events stop at the first `terminal` / `openwith` |
| 8 | Info | Settings | "Version" label isn't localized |
| 9 | Info | `finder-menu terminal` | `mdfind` query interpolates the stored bundle id unescaped |
| 10 | Info | Project | No CI, no tests |
| 11 | Info | Fork | README/SECURITY point installs, releases and reports at upstream |

## Findings

### 1. Dotfile template duplicates *(Medium, verify on a Mac)*
`Sources/FinderMenu/main.swift` `unusedFilename(_:in:)` splits the name with `NSString.deletingPathExtension` / `pathExtension`. Apple documents the extension as whatever follows the final period, so for `.env` Foundation likely reports extension `env` and a base that is either `""` or `.env`, giving `" 2.env"` or `".env 2.env"` instead of `.env 2`. The README uses `.env` as its showcase template. Check with `(".env" as NSString).pathExtension` and `.deletingPathExtension`; if confirmed, treat a single leading dot as part of the base name.

### 2. Go to Parent Folder fallback *(Low)*
`goUp(from:)` looks for the Finder window whose target matches the clicked folder and, if none matches, retargets the **front** Finder window. Right-clicking the Desktop (target `~/Desktop`, no window) or any non-window context therefore sends whatever window is in front to the parent of the clicked folder. Suggest doing nothing (or opening a new window) when no window matches.

### 3. Package contents *(Low)*
`folder(for:)` maps a package to its parent folder. That is right for a *selected* `.app`, but when browsing inside a package via Show Package Contents, the extension's `targetedURL` is the package itself, so New File, Open Terminal Here and Go to Parent Folder act on the folder that contains the package. Distinguishing "selected item" from "current folder" in the URL would fix it.

### 4. Launch timing race *(Low)*
`applicationDidFinishLaunching` shows the settings window if no URL arrived within 0.5 s. On a slow cold launch the URL can arrive later: the settings window appears, the action still runs, and `finish()` no longer quits because a window exists.

### 5. Misleading error *(Low)*
`createFile` maps every error other than "file exists" to "Couldn't create the file here (no permission?)". A name over 255 bytes, a read-only volume or a full disk all show that text.

### 6. `restart_extension` with Finder not running *(Low)*
`finder_copies` uses `pgrep -xU "$UID" Finder`; with Finder quit it returns nothing, the count is 0, the loop runs 5 times (~10 s) and prints the "didn't restart cleanly" warning.

### 7. Multi-URL Apple events *(Info)*
`application(_:open:)` returns after the first `terminal` or `openwith` URL, so later URLs in the same event are dropped. The extension only ever sends one, so this is theoretical.

### 8. Unlocalized label *(Info)*
`SettingsWindow.swift` builds `"Version " + …` without `NSLocalizedString`.

### 9. `mdfind` interpolation *(Info)*
`scripts/finder-menu terminal` puts the stored `Terminal` default into an `mdfind` query string without escaping. The value is user-written, so the impact is only a broken query.

### 10. No CI or tests *(Info)*
Nothing checks that the app builds. A GitHub Actions `macos-latest` job running `scripts/build.sh "$RUNNER_TEMP/out"` plus a localization key-parity check would catch most regressions. Unit tests would need the shared code split into a SwiftPM target.

### 11. Fork references *(Info)*
`README.md` (release badge, `brew install nikgrbn/tap/finder-menu`, clone URL) and `SECURITY.md` (advisory link) point to upstream. Fine if this fork tracks upstream; if the fork diverges, users following its README would install upstream instead. `LICENSE` must keep the upstream copyright line under MIT.

## Checked and fine

- **Link authentication:** `sentByOurExtension()` pins the sender to the bundled appex's cdhash via `kSecGuestAttributeAudit` + `cdhash H"…"` requirement and fails closed.
- **Injection:** AppleScript literals escape `\` and `"`; terminals launch through `NSWorkspace` arguments or URL components, never a shell; the `finder-menu terminal` osascript call escapes its input.
- **File creation:** `Data().write(…, options: .withoutOverwriting)` is atomic against races; names with `/`, `.`, `..` or only whitespace are rejected; templates are validated on read and write.
- **Allowlists:** `openwith` only accepts bundle ids in `Editors.swift`; template filenames never travel in URLs.
- **Sandbox:** the extension holds only `app-sandbox` plus a read-only shared-preference exception for `local.findermenu`.
- **Localization:** en, de, it, ru, uk each have all 39 keys, no extras, no duplicates, matching `%@` counts, and both Info.plists list all five.
- **Plists:** principal class `FinderMenuSync` matches `@objc(FinderMenuSync)`; versions match across both plists.
- **Scripts:** `set -euo pipefail` everywhere; `enable` no longer deletes the installed copy when re-enabling it; build output stays outside the repo.
- **README accuracy:** feature list, terminal order, editor list and the line-count claims match the code.
- **Docs tooling:** `mockups.html` loads no remote resources.
