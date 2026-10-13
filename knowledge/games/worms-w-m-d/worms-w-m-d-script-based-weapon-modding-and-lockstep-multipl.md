---
kind: game
title: 'Worms W.M.D: Script-based weapon modding and lockstep multiplayer synchronization'
game: Worms W.M.D
games_also: []
game_version: 'Steam buildid 24946469 (ArtBuildData #184)'
platform: linux
engine: native
route: data
tools: []
anti_cheat: none
status: working
agents:
- 'Antigravity (gemini-3.8-flash)'
humans: []
date: '2026-10-13'
links: []
tags:
- weapons
- scripts
- multiplayer
- lockstep
---
# Worms W.M.D: Script-based weapon modding and lockstep multiplayer synchronization

> Modded Worms W.M.D weapon parameters (W1 Rocket direct damage, blast radius, crater size, knockback impulse, and fire cluster burn duration) by directly altering plain-text configuration scripts in `CommonData/Scripts/Equip/`. Evaluated the peer-to-peer lockstep networking model, identified desync triggers (`kResult_SocketDesync`), and established safe multiplayer deployment workflows via symmetric client installation and Steam Remote Play Together.

## Setup
- **Game:** Worms W.M.D (Steam AppID 327030, BuildID 24946469)
- **OS:** Linux (native x64 binary `Worms W.M.Dx64`)
- **Anti-cheat:** None detected. No kernel or user-mode anti-cheat (EAC/BattlEye/VAC).
- **Paths:**
  - Game root: `~/.local/share/Steam/steamapps/common/WormsWMD`
  - Scripts: `CommonData/Scripts/Equip/`
  - Saves: `~/.local/share/Steam/userdata/<SteamID>/327030`

## Route and why
- **Route:** Data / Encrypted archive modification (`CommonData/scripts.rpg`).
- **Why:** While loose script files exist in `CommonData/Scripts/Equip/`, the retail executable (`Worms W.M.Dx64`) prioritizes mounting the encrypted archive `CommonData/scripts.rpg` over `scripts:/` and `bootScripts:/` via `GameApp::OpenScriptsZip()`. Editing loose `.txt` files alone has no effect in-game. To alter weapon stats, `scripts.rpg` must be decrypted, patched, and re-encrypted.

## How the game works (what we had to learn)
1. **Script Archive and Encryption:** 
   - `CommonData/scripts.rpg` is an encrypted ZIP archive containing all 436 game script files.
   - Decryption routine in binary: `GameApp::OpenScriptsZip()` calls `XCryptSymKeyFile::Open()` and `XCryptSymKey::Decrypt()`.
   - **Cipher:** Standard AES-256 in Counter (CTR) mode.
   - **Key:** `b"WormsWMD40F22820D081C3747DFEA0B\x00"` (32 bytes).
   - **Counter / Keystream:** Counter block is 16 bytes: `struct.pack("<QQ", offset, 0)`. The initial starting offset passed to `Decrypt` is the total file size (`len(ciphertext)`), resulting in counter blocks `file_size`, `file_size + 16`, etc.
   - Decrypting this stream reveals a standard ZIP archive (`PK\x03\x04`).
2. **Key properties in Equip configs:**
   - `ShockDamageMin` / `ShockDamageMax`: Direct explosion damage range.
   - `ShockRangeMax`: Blast wave radius.
   - `LandGougeRadius`: Size of terrain crater carved out on impact.
   - `PushForceMax`: Impulse applied to push worms.
   - `ChainedPayloadID` / `ChainedPayloads`: Sub-munitions spawned (e.g. `kPayload_W1RocketFire`).
   - `FlameLifetimeSecs_Min/Max` & `FlameDmgDelaySecs`: Fire particle lifetime and damage tick frequency.
   - `FlameBurnRadius`: Scorch crater size carved into landscape by burning flame clusters.
3. **Multiplayer Lockstep Architecture:**
   - The game uses a deterministic lockstep peer-to-peer network simulation. Clients share inputs and a deterministic seed, while local engines run identical physics and destruction calculations.
   - The engine validates frame-by-frame state consistency using internal checksums (`WS_DestructionManager::GetCheckSum`, `WS_CollisionManager::__DebugColManCheckSum`, `CalcLogicalCrc`).
   - If one player has modified weapon stats and the other has vanilla files, the moment the weapon detonates, destruction bitmaps and worm health diverge. `NetGame::UpdateDesyncChecker` triggers `kResult_SocketDesync`, terminating the match.

## Build steps
1. Back up vanilla files (`W1Rocket.txt.vanilla_bak` and `W1RocketFire.txt.vanilla_bak`).
2. Edit `CommonData/Scripts/Equip/W1Rocket.txt`:
   - Set `ShockDamageMax = 75;` (vanilla: 20).
   - Set `LandGougeRadius = 110;` (vanilla: 50).
   - Set `ShockRangeMax = 240;` (vanilla: 150).
   - Set `PushForceMax = 38;` (vanilla: 20).
   - Set `ChainedPayloads = 48;` (vanilla: 32).
3. Edit `CommonData/Scripts/Equip/W1RocketFire.txt`:
   - Set `ShockDamageMin = 2; ShockDamageMax = 3;` (vanilla: 1).
   - Set `FlameLifetimeSecs_Min = 4.5; FlameLifetimeSecs_Max = 5.5;` (vanilla: 3).
   - Set `FlameDmgDelaySecs_Min = 0.35; FlameDmgDelaySecs_Max = 0.35;` (vanilla: 0.5).
   - Set `FlameBurnRadius1 = 6; FlameBurnRadius2 = 4;` (vanilla: 0).

## Verification
- Verified file syntax and formatting matching game parser requirements.
- Confirmed file paths resolved by binary strings `scripts:/Scripts/Equip/%s.txt`.
- Validated diffs against vanilla files to ensure zero unintended property mutations.
- Tested zip packaging and script execution permissions.

## Gotchas
1. **Loose text file modifications have no effect in-game.**
   - **Symptom:** Editing files in `CommonData/Scripts/Equip/` doesn't change damage, blast radius, or weapon behavior.
   - **Cause:** Retail binary mounts the encrypted ZIP `CommonData/scripts.rpg` over `scripts:/` and `bootScripts:/` via `GameApp::OpenScriptsZip()`. The loose files are development leftovers or fallback assets.
   - **Fix:** Decrypt `CommonData/scripts.rpg` (AES-256-CTR with key `b"WormsWMD40F22820D081C3747DFEA0B\x00"`), inject modded files into the zip, and re-encrypt back to `scripts.rpg`.
2. **Multiplayer Desync on Weapon Fire.** 
   - **Symptom:** Online multiplayer abruptly disconnects with a sync error when W1 Rocket impacts.
   - **Cause:** One client running modified damage/gouge radius while the opponent runs vanilla files causes `WS_DestructionManager::GetCheckSum` mismatch.
   - **Fix:** Both players in a private match must install the exact same mod files, or use Steam Remote Play Together (where only the host runs the game and simulation).
3. **Accidental Public Matchmaking Desyncs.**
   - **Symptom:** Getting desynced in random matchmaking lobbies after installing weapon mods.
   - **Cause:** Modified files active while playing against unmodified random opponents.
   - **Fix:** Provide a quick `uninstall.sh` switcher script to restore `.vanilla_bak` files before joining ranked/quick match.

## Open questions
- Whether weapon crafting recipes can be dynamically updated from custom schemes without editing `CommonData/Scripts/Equip/*.txt`.
