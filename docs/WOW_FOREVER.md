# WoW Forever support (version 27.4)

This document records why Masque: Raeli was updated in November 2026, what was changed, and how the changes were checked.

## Goal

Make Masque: Raeli load and work in **World of Warcraft: Forever** without being flagged as out of date or incompatible, while:

- keeping the existing Retail and Classic support working,
- keeping the visual look of all four skins unchanged,
- bringing the skin definitions up to date with the current Masque skin API.

## Background: what WoW Forever is

WoW Forever is Blizzard's "Classic Plus" game.

| Property | Value |
| --- | --- |
| Client version | 1.60.1 |
| TOC interface number | `16001` |
| Game type | Its own project, internally named "Camelot" (`WOW_PROJECT_CAMELOT`). It is **not** Classic Era. |
| Addon API | The modern Mainline API (based on Retail 12.x), not the Classic Era API |
| Flavor-specific TOC suffixes | `_Camelot.toc` (and reportedly `_Forever.toc`). `_Classic.toc` does **not** apply. |
| Typical install folder | `World of Warcraft\_classic_beta_\Interface\AddOns` |

Because Forever is a separate game type, a TOC that only lists Classic Era interface numbers (such as `11503`) is not loaded as up to date in Forever. The addon has to list `16001` itself.

How these facts were checked:

- Masque's own TOC lists `16001` on its `## Interface:` line.
- The BigWigs packager, which builds this addon's releases, maps every `16xxx` interface number to the "forever" game type and writes a `_Camelot.toc` for it.
- The remaining details (the project constant and the possible rename of the suffix to `_Forever`) come from community guides, not Blizzard documentation.

## State of the addon before the update

- **TOC:** one file, `Masque_Raeli.toc`, with a multi-value line `## Interface: 110002, 40400, 11503` (Retail 11.0.2, Cataclysm Classic, Classic Era from 2024) and a single file entry, `Masque_Raeli.lua`.
- **Packaging:** `.pkgmeta` has `enable-toc-creation: yes`, so the packager splits the multi-value Interface line into one TOC per game version at release time.
- **Skins:** `Masque_Raeli.lua` gets Masque through `LibStub('Masque')` and registers four skins with `MSQ:AddSkin()`:
  - Raeli - Square Edge
  - Raeli - Square Inset (uses Square Edge as its template)
  - Raeli - Ring Edge
  - Raeli - Ring Inset (uses Ring Edge as its template)
- **Masque API version:** the skins declared `Masque_Version = 90001`, the old name for the API version key.

## What changed

### 1. TOC: Forever interface number

`16001` was added to the existing Interface line:

```
## Interface: 110002, 40400, 11503, 16001
```

A multi-value line was chosen over a separate `_Camelot.toc` because:

- it matches how the addon was already set up,
- Masque itself uses the same approach,
- the packager already generates a `_Camelot.toc` from it at release time,
- there is only one file list to maintain.

The file list was not changed.

### 2. Masque API version

Current Masque (12.1.8, November 2026) uses `API_VERSION = 120001`. It still accepts the old `Masque_Version` key as a fallback, but `API_VERSION` is the current name.

Both Edge skins now declare `API_VERSION = 120001` instead of `Masque_Version = 90001`. The Inset skins inherit it through their templates.

### 3. Pet autocast settings migrated

Masque 12.1.8 removed the `AutoCastable` and `AutoCastShine` skin settings. Raeli still used them, so current Masque was silently ignoring Raeli's autocast values.

Masque now uses these settings instead:

| New setting | Replaces | Used for |
| --- | --- | --- |
| `AutoCast_Frame` | `AutoCastShine` | Size and position of the autocast frame |
| `AutoCast_Corners` | `AutoCastable` | The autocast corner border |

Each new setting has two parts:

- **Top-level values** are used on clients with the modern animated autocast effect (Retail and Forever).
- **A `Classic` sub-table** is used on clients with the classic autocast border.

Raeli's original values were moved into the `Classic` sub-tables. The top-level values were set to match Masque's own default skins (`AutoCast_Corners.SetAllPoints = true`; frame size left to Masque's default), so the modern autocast effect looks exactly as it did before.

| Skin | `AutoCast_Frame.Classic` | `AutoCast_Corners.Classic` |
| --- | --- | --- |
| Square Edge / Inset | 34 x 34, offset (1, -1) | `UI-AutoCastableOverlay`, 65 x 65, offset (0.5, -0.5) |
| Ring Edge / Inset | 30 x 30, offset (1, -1) | Masque's default classic texture, 54 x 54, offset (0.5, -0.5) |

Every other skin setting Raeli uses was compared with Masque's current list of skin layers and is still valid. No textures, sizes or colours were changed.

### 4. Client API: `GetAddOnMetadata`

The only game function the addon calls is `GetAddOnMetadata`, used to show the addon version in Masque's skin info. On clients built on the modern Mainline API, including Forever, the global function no longer exists; it lives at `C_AddOns.GetAddOnMetadata`.

The code now:

1. uses `C_AddOns.GetAddOnMetadata` when it exists,
2. falls back to the old global on older clients,
3. leaves the version empty instead of raising an error if neither exists.

### 5. Version and changelog

- `## Version:` in the TOC is now `27.4` instead of the `@project-version@` placeholder. The placeholder is only filled in by the release packager, so a manual install (copying the folder) used to show the literal text `@project-version@` in Masque's skin info.
- `CHANGELOG.md` has a 27.4 entry.

## How it was tested

- `Masque_Raeli.lua` was loaded in a Lua runtime with a stand-in for Masque's `AddSkin` (including template inheritance) under three conditions:
  - only `C_AddOns.GetAddOnMetadata` available (Forever and Retail),
  - only the old global `GetAddOnMetadata` available (older clients),
  - neither available.
  
  In all three cases the four skins registered, reported `API_VERSION = 120001`, carried the migrated autocast values, and no longer had the old `AutoCastable`/`AutoCastShine` keys.
- The addon was installed in the WoW Forever client (running under Bottles on Linux) and confirmed to load and work.

## Installing for testing

1. Download the repository (for example the `master` branch as a ZIP) and extract it.
2. Rename the extracted folder to exactly **`Masque_Raeli`**. Texture paths are hardcoded to `Interface\AddOns\Masque_Raeli\...`, and the folder name must match the TOC name.
3. Copy the folder into the Forever AddOns directory so that `Masque_Raeli.toc` sits directly inside `AddOns/Masque_Raeli/`.
4. Make sure a current Masque (12.1.x) is installed in the same AddOns directory.
5. In game, open `/msq` and go to Skins; the four Raeli skins should be listed.

AddOns directory locations:

| Setup | Path |
| --- | --- |
| Windows | `C:\Program Files (x86)\World of Warcraft\_classic_beta_\Interface\AddOns\` |
| Linux, Bottles (Flatpak) | `~/.var/app/com.usebottles.bottles/data/bottles/bottles/<BottleName>/drive_c/Program Files (x86)/World of Warcraft/_classic_beta_/Interface/AddOns/` |
| Linux, Bottles (distro package) | `~/.local/share/bottles/bottles/<BottleName>/drive_c/Program Files (x86)/World of Warcraft/_classic_beta_/Interface/AddOns/` |

If the game was installed somewhere other than the default, this finds the game-version folders inside Bottles:

```bash
find ~/.var/app/com.usebottles.bottles ~/.local/share/bottles -type d -path "*World of Warcraft/_*_" 2>/dev/null
```

## Known follow-ups

- The Retail and Classic interface numbers (`110002`, `40400`, `11503`) are still from 2024, so those clients may show the addon as out of date. Masque's current list is `11509, 20506, 30405, 38002, 40402, 50504, 120105`.
- The Forever TOC suffix may be renamed from `_Camelot` to `_Forever`. This only matters if a separate flavor-specific TOC is added later; the multi-value Interface line is not affected.
- The `TODO` at the top of `Masque_Raeli.lua` about Masque's cooldown pulse support is unchanged.

## References

- Masque source: <https://github.com/SFX-WoW/Masque>
- BigWigs packager: <https://github.com/BigWigsMods/packager>
- Example Forever port of another addon: <https://github.com/Nevcairiel/Quartz/pull/110>
- Community Forever addon guides: <https://forevermeta.me/en/guides/wow-forever-addons-interface>, <https://wowforeverguides.com/addons/developers>
