# Chrono Divide Mod SDK

<!-- vscode-markdown-toc -->
* [Introduction](#Introduction)
* [The Engine](#TheEngine)
	* [File and folder structure](#Fileandfolderstructure)
	* [Custom game extensions](#Customgameextensions)
		* [Game modes](#Gamemodes)
		* [Game rules](#Gamerules)
	* [File loading priority](#Fileloadingpriority)
	* [Rules and art merging](#Rulesandartmerging)
* [Modding guide](#Moddingguide)
	* [Development workflow](#Developmentworkflow)
	* [Preparing for release](#Preparingforrelease)
	* [Packaging and distribution](#Packaginganddistribution)
	* [Modding limitations](#Moddinglimitations)
	* [Breaking changes](#Breakingchanges)
	* [Unsupported engine features](#Unsupportedenginefeatures)

<!-- vscode-markdown-toc-config
	numbering=false
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

## <a name='Introduction'></a>Introduction

This repository contains the resources needed to get a new Chrono Divide mod started, or to easily adapt an existing RA2 mod, for full compatibility with the new engine. Basic knowledge of RA2 modding is required.

The Chrono Divide game engine is customizable to a high degree and is mostly compatible with original RA2 (v1.006) mods. Because it runs with the original RA2 MIX archives, it maintains backwards compatibility, aside from a few breaking changes, which will be explained later in this document.

**IMPORTANT**: Yuri's Revenge mods are NOT compatible.

## <a name='TheEngine'></a>The Engine

### <a name='Fileandfolderstructure'></a>File and folder structure

The engine stores all game assets in a virtual file system, backed up by a browser storage layer, such as OPFS or IndexedDB. Aside from a few exceptions, this file system behaves like an ordinary file system.

> Due to security restrictions, the web application has no access to the user's real file system and uses a private sandbox instead.

The internal storage can be accessed using a browser's dedicated dev tools or with the custom Chrono Divide GUI, found under "Options" -> "Storage". Below is a diagram of the basic folder structure:

```
📁 / (Root)
├─📁 cache                              - Various caches (e.g. voxel models)
├─📁 mods                               - Extension point for game modifications
│ └─📁 <modId>
│   ├─📁 replays
│   └─expand*.mix, ecache*.mix etc.
├─📁 music                              - Contents of theme.mix converted to MP3
├─📁 replays
├─📁 Taunts                             - Original RA2 Taunts
├─ra2.mix, multi.mix, language.mix      - Original RA2 mix archives
└─glsl.png, ra2ts_l.webm                - Assets converted from SHP/BIK
```

The `mods` folder is where all custom game modifications are stored, each with its own dedicated sub-folder. A mod folder may contain any combination of the following types of files: `*.mix`, `*.mmx`, `*.ini`, `*.csf`, `*.mpr`, `*.map`, `*.pkt`.

> MIX archives are only loaded if they match an original MIX file name or one of the following patterns: `expand##.mix`, `ecache##.mix`, `elocal##.mix`, where `#` represents a single digit.

### <a name='Customgameextensions'></a>Custom game extensions

The game engine uses a custom [ra2cd.mix](src/ra2cd.mix/) which contains mostly INI overrides and some new assets. This file is bundled with the game client and cannot be found in the virtual file system. Below is an overview of the custom files it contains and their purpose:

```
📁 ra2cd.mix
├─sidec01cd.mix     - custom icons for the command bar (allied version)
├─sidec02cd.mix     - custom icons for the command bar (soviet version)
├─artcd.ini         - art.ini overrides
├─mpmodescd.ini     - replacement for mpmodes.ini
├─ra2cd.pkt         - custom list of multiplayer maps
├─rulescd.ini       - rules.ini overrides
└─ui.ini            - replacement ui.ini, adding more buttons in the command bar
```

As a general rule, all custom file names defined by Chrono Divide, which don't exist in the original game, contain the `cd` suffix.

**IMPORTANT**: Some files are stand-in replacements to their original counterparts (`mpmodescd.ini`, `ui.ini`), while others are just extensions, adding or overriding specific values (`artcd.ini`, `rulescd.ini`)

#### <a name='Gamemodes'></a>Game modes

Chrono Divide implements the "Team Alliance" game mode from the later Yuri's Revenge (YR) expansion, and thus uses a custom [mpmodescd.ini](./src/ra2cd.mix/mpmodescd.ini), derived from the original `mpmodes.ini` and `mpmodesmd.ini` files.

**BREAKING CHANGE**: The engine properly reads the `AlliesAllowed` flag from `<mpmode>.ini`, just like YR does. The vanilla RA2 engine did not in fact read this correctly and was probably hardcoded. Because of this, the custom MIX also includes the fixed game mode INIs from YR (`<mpmode>md.ini`)

#### <a name='Gamerules'></a>Game rules

The custom [rulescd.ini](src/ra2cd.mix/rulescd.ini) and [artcd.ini](src/ra2cd.mix/artcd.ini) override various values from the base `rules.ini` and `art.ini`, respectively. These changes have a minimal impact on gameplay and are mostly bug fixes or patches that prevent various exploits of the original game. These changes are documented inline, within the INI files.

**BREAKING CHANGE**: The custom files also contain critical configuration required for the engine to function properly. These changes are marked accordingly and should be dealt with sensibly by modders.

> Although possible to hardcode most of the values within the engine just like vanilla RA2 did, thus avoiding some breaking changes, having them in the INI allows for further customization, which was not possible before.

For full documentation on the supported INI entries, as well as new additions, please refer to [RULES](./RULES.md) and [ART](./ART.md) documents.

### <a name='Fileloadingpriority'></a>File loading priority

On start-up, the game engine creates a single virtual folder by merging the mod folder, the root folder, as well as the contents of specific MIX archives.

Files are loaded by following a well-defined order of priorities. For example, if two files with the same name exist in two separate locations, only the one with the highest priority is loaded.

Below is the list of files loaded from the virtual folder, in order of decreasing priority (High -> Low):

- `*.ini`, `ra2.csf`, `*.mpr`, `*.map`, `*.pkt`
- `ecache99.mix` -> `ecache00.mix`
- `expand99.mix` -> `expand00.mix`
- `elocal99.mix` -> `elocal00.mix`
- `*.mmx`
- `ra2cd.mix` (hard-coded)
- `language.mix`,
- `audio.mix`, `audio.bag` ( + `audio.idx`), `cameo.mix`
- `ra2.mix`
- `cache.mix`, `load.mix`, `local.mix`, `neutral.mix`, `conquer.mix`, `generic.mix`, `isogen.mix`, `<theater>.mix`, `iso<theather>.mix`
- `multi.mix`
- `sidec01cd.mix`, `sidec02cd.mix`, `sidec01.mix`, `sidec02.mix`

### <a name='Rulesandartmerging'></a>Rules and art merging

To avoid duplicating the entire `rules.ini` or `art.ini` contents, the engine uses a separate set of [rulescd.ini](src/ra2cd.mix/rulescd.ini) and [artcd.ini](src/ra2cd.mix/artcd.ini) files. These only contain new values and overrides specific to Chrono Divide and are merged over the base versions when loading the game.

> The merge is done on a field-by-field basis. For example, defining a section with a single key will not overwrite the entire section from the base file, but only the specified key instead.

Modders can provide their own `rules.ini` or `art.ini` as before, and normally need not worry about the added `*cd.ini` variants. However, in rare cases of key conflicts, the custom `*cd.ini` variants take priority and the modder may have to modify them as well.

## <a name='Moddingguide'></a>Modding guide

### <a name='Developmentworkflow'></a>Development workflow

1. Open the Chrono Divide client and navigate to "Options" -> "Storage".
2. Open the `mods` folder and create a new folder for your mod.

	> NB: The folder name is used as an unique ID and must only consist of alphanumeric characters, dash (`-`) or underscore (`_`). You should choose a name that's short, yet readable.

3. Copy all your modded files to this folder. You can either drag and drop, or use the "Upload" button from the toolbar.

	> **NB**: Making any changes to the mod files requires a full reload of the game client.

4. Reload the game client and navigate to the "Mods" screen.

	> You can also leave the file browser open for easy access and open another game client in a different tab. Just make sure to always refresh after making changes.

5. Press "Load" to reload the game client using this mod.

	> Note that your mod is listed as `UNSUPPORTED`. This is expected during development or when importing RA2 mods that weren't designed or tested specifically for Chrono Divide. We will fix this later when preparing the mod for publishing.

6. Test and debug your changes.

	> The game engine will generally report INI mistakes as warnings in the browser's dev console (F12). It will also warn when using unsupported or incompatible engine features. If an error is sufficiently severe, the game will crash and print an error message in the console.

7. Rinse and repeat.

8. When you are happy with the changes, it's time to package and prepare the mod for distribution.

### <a name='Preparingforrelease'></a>Preparing for release

The game will generally allow importing any RA2 mod from an archive file via the "Import..." button from the "Mods" screen. However, it is not guaranteed that all mods will function correctly, due to subtle differences and backwards incompatible changes in the game engine, compared to the vanilla RA2 engine.

Mods that weren't specifically designed or tested for Chrono Divide, will thus show a warning when being imported and will also be listed in the mod selector with an `UNSUPPORTED` tag.

The game will only treat a mod as fully supported when the mod has a manifest file named `modcd.ini`. This is a file that contains various metadata about a mod, such as a human-readable name, description, author and version.

**IMPORTANT**: The manifest must be a standalone file and cannot be included in a MIX archive.

Below is a sample manifest, illustrating all supported fields:

```ini
; modcd.ini
[General]
; The mod ID will be used as folder name when installing the mod
; and may only contain alphanumeric characters, dash (-) or underscore (_).
; It should be unique, yet readable
ID=my-mod
Name=My Mod ; This is shown in the mod selector as well as the games list
Description=A nice description to show to the player
; The version field can have any format, but should ideally be updated whenever
; non-cosmetic changes are done to the mod.
; The version is shown in the games list and is relevant when players
; using a different version attempt to join
Version=1.0.0
Author=Modder Name
Website=https://my.mod.com ; A dedicated page or website for the mod
```

### <a name='Packaginganddistribution'></a>Packaging and distribution

The mod files can simply be archived using 7-Zip or any other software that is able to produce a compatible archive. Just make sure to include all relevant files using a flat structure (without any nested folders) and also to include the `modcd.ini` manifest.

The mod archive can be uploaded to a dedicated website, such as ModDB, from which players can freely download it and then install it in the Chrono Divide client, with the "Mods" -> "Import..." button.

> All archive formats supported by 7-Zip can be handled by the game client. E.g: `rar`, `tar`, `tar.gz`, `tar.bz2`, `tar.xz`, `zip`, `7z`

You can also apply on the game's [Discord](https://discord.gg/uavJ34JTWY) server to have your mod featured directly in the game client. The mod can be hosted on the Chrono Divide CDN and players will be able to easily install it with a single click from the "Mods" screen.

> A manual install method is also provided when hosting the files on a 3rd-party server, but the one-click install requires that the server sets correct CORS headers that allow the game client to download the mod files directly.

### <a name='Moddinglimitations'></a>Modding limitations

* The splash image that shows when loading the game (`glsl.shp`) can be customized, but it has to be provided as a standalone file, in PNG format (`glsl.png`). It will not be loaded from MIX files, due to limitations.

* The music tracks found in `theme.mix` are not currently customizable, nor is the `theme.ini` configuration.

* BIK videos are not converted when importing the mod and they cannot be used. It is however possible to override the menu video with a custom `ra2ts_l.webm` file. The format must be WebM (VP8) and can also be loaded from a MIX archive.

* The custom translation strings added by the Chrono Divide client cannot currently be changed from the CSF files.

### <a name='Breakingchanges'></a>Breaking changes

* HVA animations are currently not supported. HVA files are loaded, but only the first frame is actually used. As an alternative method for animating helicopter rotors, several `art.ini` fields have been introduced, offering a greater degree of freedom. Please refer to the documentation found in the sample [artcd.ini](src/ra2cd.mix/artcd.ini), under `Rotor*`.

* Multi-section VXL files must not have duplicate section names within the header. Having multiple sections with the same name will result in the engine loading only the last section that shares that name within the given VXL file.

* Most Z/depth sorting related fields are not parsed. The 3D engine generally tries to fit each sprite as well as possible within the 3D world and doesn't use the object sorting algorithm of the original game. As an exception, the `ZShapePointMove` field is used for buildings with `Refinery=yes` or `NukeSilo=yes` to prevent clipping of objects within the 3D box of the respective building.

* Building animations such as airport pad lights (`GAAIRC_A`) must have `Flat=yes` to prevent clipping of stationed aircraft.

* Repair depot animations `End` values are normalized and interpreted as an inclusive last frame index, instead of a frame count.

* Several errors which were ignored by the vanilla RA2 engine, such as setting a non-existent `Warhead` will now cause a fatal crash instead.

### <a name='Unsupportedenginefeatures'></a>Unsupported engine features

* &cross; Map triggers (aside from cosmetic ambient sfx) and, as direct consequence, singleplayer or multiplayer coop maps
* &cross; INI flags related to AI-controlled opponents found in `rules.ini` and `ai.ini`
* &cross; Localized or ambient damage inflicted by Particle systems or sprite Animations
* &cross; Crate Powerups that are unused in the original client (Invulnerability, IonStorm, Gas, Tiberium, Pod, Cloak, Darkness, Explosion, ICBM, Napalm, Squad)
* &cross; Veterancy flags on pre-placed map units
* &cross; Negative light/darkening on `AlphaImage`s (values lower than 127)
* &cross; Veteran abilities which are unused in the original client (e.g. RANGE, GUARD_AREA)
* &cross; `voxels.vpl`. Because voxels are rendered as 3D models, lighting is entirely managed by the 3D engine.
* &cross; Obsolete TS logic (tunnels, ice cracks etc.)

Please also check the list of supported [rules.ini](./RULES.md) and [art.ini](./ART.md) flags.
