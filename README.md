# SirenSetting Limit Adjuster: Build 3889 patch

A patched version of the SirenSetting Limit Adjuster ASI plugin that works on GTA V game build 3889 under FiveM, including 32 siren lights per siren setting.

This is not the original plugin and not an official release. The original plugin is closed source and all credit for it belongs to its author. This file is a binary patch of that plugin, made for my own use and shared as is.

## Problem

On FiveM with game build 3889 the original plugin fails in two stages:

1. FiveM refuses to load it:
   `this ASI plugin does not claim to support game build 3889`
   The plugin has no `FX_ASI_BUILD 3889` resource.
2. After adding that resource the plugin loads, but its log shows:
   `Buffer hook application failed!`
   The ID and index hooks apply, but the buffer hook does not. The game is left half patched and siren lights stop working.

Note: when a server enforces an older game build (for example `sv_enforceGameBuild 3095`), current FiveM versions still run the 3889 executable. The plugin has to support the executable, not the enforced build number.

## Cause

The buffer hook searches 19 code patterns in the game and writes new field offsets there to get 32 instead of 20 lights. In build 3889 five of these patterns no longer match, and the code around many of the others was recompiled. The large siren update function, the per vehicle siren state structure and its constructor changed.

## What was changed

1. Added the `FX_ASI_BUILD 3889` resource.
2. Replaced the original buffer hook initialization with new code in an added PE section (`.s32`). The new code:
   - locates six anchors by pattern scan (the siren update function, two getter functions, the allocation site, the state constructor and the bone index function),
   - compares every patch site with the expected bytes from build 3889 and aborts without writing anything if a single byte differs,
   - then applies all patches:
     - state structure size `0x188` to `0x248`,
     - replacement constructor with four arrays of 32 entries and shifted tail fields,
     - displacement changes for the arrays, bit masks and float table in the update function and the two getters,
     - loop bound and light count constants changed from 20 to 32,
     - bone lookup through the plugin tables,
     - a per frame copy of the siren setting with 32 entries, freed at function exit.
3. Restored the plugin critical section and the bone function pointer, which the original initializer set.

The ID and index handling of the original plugin is unchanged.

## Installation

1. Close FiveM.
2. Back up your current `SirenSetting_Limit_Adjuster.asi` (if you have one).
3. Copy `SirenSetting_Limit_Adjuster.asi` from this repository into `FiveM.app\plugins`.
4. Start FiveM and check `SirenSettings.log` (see below).

Players on a server also need the plugin to see more than 20 lights on other vehicles.

To revert, close FiveM and put your backup back.

## Verification

`SirenSettings.log` in the GTA directory should contain:

```
Found ID patterns.
ID hooks applied.
Found index patterns.Index hooks applied.
Buffer hooks applied.
```

If the check fails (for example after a game update), the log shows `Buffer hook application failed!`, nothing is written to the game and the plugin does not apply the 32 light changes.

## File

- `SirenSetting_Limit_Adjuster.asi`
  - size: 55296 bytes
  - SHA-256: `05E8723A1EF31E7365013C1ECC633F6ABC8E32FF3B2B7464EB4861D5ABEAB3FC`

## Test results

Tested on one machine, FiveM, game build 3889:

- Vehicles with up to 20 lights work as before.
- Vehicles whose siren setting ID is above 255 map to the correct setting.
- A siren setting with more than 20 lights (for example `NPS 2023 Tahoe PPV w/ push bumper`) shows lights beyond number 20.
- No crash during normal driving or when entering and leaving vehicles.

## Known limitations

- Only the main siren update function was ported. The loops that clear light bits for damaged or missing lights were left at 20 lights. A damaged light above number 20 may not switch off.
- Tested on a single machine and a single game build. Other FiveM or game updates can change the code again.
- Writing into game code carries a crash risk. The patch is checked by exact byte comparison, but it is not a replacement for a proper rebuild from source.
- The siren setting is copied once per vehicle and frame. A small performance cost is possible with many vehicles.

## Credits

- Original plugin: SirenSetting Limit Adjuster by its author (source not public).
- Analysis and patch: written with the help of an AI coding assistant and tested by hand in game.

## Disclaimer

Use at your own risk. This is a binary modification of a third party plugin and is provided without warranty. If you are the author of the plugin and want this removed or merged, please open an issue.
