# The DayZ Bible - Internals Reference for DayZ 1.29+

> **What this is:** a structured reference built only from the UnknownCheats thread *DayZ Reversal, Structs and Offsets* (thread 104269, 438 pages, scraped 2026-10-03). It covers 1.29 and newer. Every value cites the post(s) it comes from (`#NNNN` = post number in that thread). The raw material is in `DayZ_1.29_Consolidated_Dataset.md`.
>
> **What this is not:** a verified SDK. **Nothing here was tested against a live binary by the author of this document.** Status tags describe how well the *thread* corroborates a value, not whether it works on your build.
>
> **Out of scope:** anti-cheat evasion, ban evasion and server-side exploits (duping, teleporting other players' items). Where the thread discusses these, this wiki only notes that the discussion exists.

---

## Status legend

| Tag | Meaning |
|---|---|
| **[Corroborated]** | Two or more independent posters report the same value for the same build window, and at least one reports it working. |
| **[Single-source]** | One poster reports it for a 1.29 build. No contradiction found, no independent confirmation. |
| **[Unverified]** | Reported without evidence of working, or from an automated dumper that also produced obvious junk in the same run. |
| **[Conflicting]** | Posters disagree for the same build window. All values are listed. |
| **[Needs Update for 1.29+]** | Only 1.28-or-earlier data exists in the thread, or the source turned out to be a 1.28 binary. |
| **[1.30-exp]** | Applies to 1.30 experimental only. |

Unless stated otherwise, **"module RVA"** means an offset from the `DayZ_x64.exe` image base. **"Member offset"** means an offset from an object pointer. `[x]` means "dereference x".

---

## Table of Contents

1. [Version & Build Reference](#1-version--build-reference)
   1. [Build timeline](#11-build-timeline)
   2. [Module RVAs per build](#12-module-rvas-per-build)
   3. [1.28 -> 1.29 member-offset deltas](#13-128--129-member-offset-deltas)
   4. [Provenance warnings (mislabelled dumps)](#14-provenance-warnings-mislabelled-dumps)
2. [Engine Primitives](#2-engine-primitives)
   1. [ArmaString](#21-armastring)
   2. [AutoArray / pointer tables](#22-autoarray--pointer-tables)
   3. [Weak pointers & the 0xA8 identity offset](#23-weak-pointers--the-0xa8-identity-offset)
   4. [RTTI inspection trick](#24-rtti-inspection-trick)
3. [External vs Internal: Capability Matrix](#3-external-vs-internal-capability-matrix)
4. [World](#4-world)
   1. [World member map](#41-world-member-map)
   2. [Entity tables (near / far / slow / item / bullet)](#42-entity-tables)
   3. [Resolving the local player](#43-resolving-the-local-player)
5. [Entity / EntityAI](#5-entity--entityai)
   1. [Entity member map](#51-entity-member-map)
   2. [VisualState & transforms](#52-visualstate--transforms)
   3. [EntityType (class info & names)](#53-entitytype-class-info--names)
   4. [Classifying entities & loot](#54-classifying-entities--loot)
6. [Camera, Projection & World-to-Screen](#6-camera-projection--world-to-screen)
   1. [Camera member map](#61-camera-member-map)
   2. [World-to-screen](#62-world-to-screen)
   3. [FOV / zoom](#63-fov--zoom)
   4. [Camera manager (freecam-relevant structures)](#64-camera-manager-freecam-relevant-structures)
7. [Skeleton & Bones](#7-skeleton--bones)
   1. [Pointer chain](#71-pointer-chain)
   2. [Bone matrix layout & world transform](#72-bone-matrix-layout--world-transform)
   3. [Bone indices - players](#73-bone-indices--players)
   4. [Bone indices - infected](#74-bone-indices--infected)
   5. [Skeleton drawing bone pairs](#75-skeleton-drawing-bone-pairs)
8. [Network, Scoreboard & Player Identity](#8-network-scoreboard--player-identity)
9. [Inventory, Cargo & Items](#9-inventory-cargo--items)
10. [Weapons, Magazines & Ammunition](#10-weapons-magazines--ammunition)
    1. [Weapon / chamber](#101-weapon--chamber)
    2. [Magazine & item quantity](#102-magazine--item-quantity)
    3. [AmmoType config layout](#103-ammotype-config-layout)
    4. [Bullet manipulation / "Magic Bullet"](#104-bullet-manipulation--magic-bullet)
       1. [Validation bypass: InitSpeed boost](#1041-validation-bypass-initspeed-boost)
       2. [Bullet TTL manipulation / "Recycled Bullets"](#1042-bullet-ttl-manipulation--recycled-bullets)
11. [Environment: Time, Eye Accommodation, Grass, Weather](#11-environment-time-eye-accommodation-grass-weather)
    1. [Weather manipulation](#111-weather-manipulation)
12. [Player State, Health & Stats](#12-player-state-health--stats)
13. [Raycasting & Visibility](#13-raycasting--visibility)
    1. [Internal: engine raycast](#131-internal-engine-raycast)
    2. [External: rebuilt physics scene](#132-external-rebuilt-physics-scene)
14. [Enfusion Script VM & Config Access](#14-enfusion-script-vm--config-access)
15. [Signatures](#15-signatures)
    1. [Module globals](#151-module-globals)
    2. [Member-offset signatures](#152-member-offset-signatures)
    3. [Function signatures](#153-function-signatures)
16. [Function RVAs](#16-function-rvas)
17. [1.30 Experimental Preview](#17-130-experimental-preview)
18. [Conflict Register & Open Questions](#18-conflict-register--open-questions)
19. [Source Index](#19-source-index)
20. [Vehicles](#20-vehicles)
    1. [Vehicle member map](#201-vehicle-member-map)
    2. [Wheel physics](#202-wheel-physics)
    3. [Vehicle fluids & parts](#203-vehicle-fluids--parts)
21. [Complete Feature Catalog](#21-complete-feature-catalog)
    1. [External features](#211-external-features)
    2. [Internal-only features](#212-internal-only-features)
    3. [Server-side / impossible features](#213-server-side--impossible-features)

---

## 1. Version & Build Reference

### 1.1 Build timeline

| Date (2026) | Event (as stated in thread) | Posts |
|---|---|---|
| 19 Feb | First requests for 1.29 **experimental** `1.29.162270` dumps | #8257, #8260 |
| 4 Apr | Scanner dump for 1.29 exp `1.29.162270` | #8298, #8300 |
| **8 Apr** | **1.29 goes stable** ("like today we got updated to 1.29") | #8309 |
| 8 Apr | Stable build string `1.29.162510` | #8328, #8393 |
| 9 Apr | DayZ added to Game Pass. The Microsoft/Xbox PC build has **different** offsets. | #8347, #8464-#8465 |
| ~3 Jun | Patch shifted module RVAs ("an update like 3 days ago which changed a few like world base") | #8500 |
| 14 Jul | Build string `1.29.0.163047` (Steam), 313-offset dump | #8573 |
| 15 Jul | Module RVAs shift again | #8576, #8577 |
| ~12 Aug | Module RVAs shift again (latest Steam values in the thread) | #8617, #8639 |
| 17 Sep | **1.30 experimental** dumps appear | #8698 |

### 1.2 Module RVAs per build

Member offsets largely survived the 1.29 stable patches. The **module globals** changed with every patch.

| Global | 1.28 stable | 1.29 exp `162270` / stable `162510` | 1.29 Jun-Jul (`1.29.0.163047`) | 1.29 mid-Jul | **1.29 Aug->Oct (latest)** |
|---|---|---|---|---|---|
| World | `0xF4B050` | `0x4263FE8` **[Corroborated]** #8311 #8351 #8380 #8393 | `0x4264028` **[Corroborated]** #8488 #8501 #8573 | `0x4264058` **[Corroborated]** #8576 #8577 | `0x4262FE8` **[Corroborated]** #8617 #8639 #8659 #8675 #8717 |
| NetworkManager | `0xF5E190` | `0x100FBD0` **[Corroborated]** #8341 #8393 | `0x100FC10` **[Corroborated]** #8488 #8529 #8573 | `0x100FC40` [Single-source] #8577 | `0x100EBD0` **[Corroborated]** #8624 #8639 #8682 #8717 #8722 |
| Tick (game speed scalar) | `0xF193C8` | `0xFF4958` **[Corroborated]** #8268 #8298 #8393 | `0xFF4998` **[Corroborated]** #8532 #8573 | `0xFF49C8` **[Corroborated]** #8577 #8609 | `0xFF3958` [Single-source] #8617, also #8717 |
| Landscape | `0xF4B268` (#8280) | [Needs Update for 1.29+] | `0x42672D0` [Single-source] #8501 | - | `0x42662E0` [Single-source] #8617 #8717 |
| ScopeFovCtx | - | - | `0x4264920` #8573 | `0x4264950` #8577 | `0x42638E0` #8617 |
| FOV_Context / gFilterInternalObj | - | `0x1008CE0` #8393 #8394 | `0x1008CE0` #8501 #8573 | - | `0xF72388` [Unverified] #8617 |
| FOV base | - | `0x100A7D8` [Single-source] #8393 | - | - | - |
| DLC map manager | `0xF29D90` | `0x1008028` [Single-source] #8311 #8393 | `0x1008028` ("untested") #8501 | - | - |
| ScriptContext | `0xF19348` | - | `0xF19398` [Unverified] #8565 | - | - |
| gStatisticsScene (physics scene) | - | `0x42648E0` [Single-source] #8394 #8439 | - | - | [Needs Update for 1.29+] |
| Free debug camera instance | `0xF29EF8` (#8222) | - | `0xFEAB30` #8573 | `0xFEAB60` #8578 | `0xFE9AF0` #8617 |
| Config roots | - | - | - | - | ConfigRoot `0x4262BD0`, ConfigWrapper `0x4262BF0`, MissionRoot `0x4262C80`, MissionWrapper `0x4262CA0` [Single-source] #8644 |

> [!] Expect these globals to move with every patch. Prefer signature scanning (S15) to hard-coded RVAs. The thread repeatedly warns that a broken World pointer, rather than a broken member offset, is the usual cause of "nothing works after update" (#8517).

### 1.3 1.28 -> 1.29 member-offset deltas

These changes arrived with 1.29 (exp then stable) and held through the later 1.29 patches.

| Field | 1.28 | 1.29+ | Status | Posts |
|---|---|---|---|---|
| DayZPlayer -> Skeleton | `0x7E8` | `0x7E0` | [Corroborated] | #8311 #8380 #8393 #8396 #8654 |
| DayZInfected -> Skeleton | `0x678` | `0x670` | [Corroborated] | #8311 #8380 #8393 #8654 |
| Skeleton -> AnimClass | `0xB0` | `0x118` | [Corroborated] (one dissent: `0x110`, #8396) | #8311 #8380 #8393 #8654 #8675 |
| AnimClass -> bone matrix array | `0xBF0` | `0xBE8` | [Corroborated] | #8311 #8380 #8393 #8652 #8654 |
| Entity -> Inventory | `0x658` | `0x650` | [Corroborated] | #8311 #8366 #8393 #8650 |
| Entity -> NetworkID | `0x6E4` | `0x6DC` | [Corroborated] | #8311 #8380 #8393 #8587 #8722 |
| EntityType -> TypeName / config name | `0xA8` | `0xD0` | [Corroborated] | #8311 #8393 #8658 #8675 |
| EntityType -> ModelName (p3d path) | `0x88` | `0xB0` | [Corroborated] | #8311 #8438 #8645 |
| EntityType -> ObjectName | `0x70` | `0x98` (MZivert) **or** `0x70` (later dumps) | [Conflicting] | #8311 vs #8617 #8643 |
| EntityType -> CleanName (display name) | `0x4F0` | `0x518` | [Corroborated] | #8311 #8364 #8393 #8410 #8658 |
| World -> EyeAccom | `0x2974` | `0x296C` | [Corroborated] | #8339 #8354 #8380 #8393 |
| World -> time scale / hour | `0x2978` | `0x2970` | [Corroborated] (changed ~22 Apr) | #8405 #8416 #8573 |
| Magazine ammo count | `0x6B4` | `0x6AC` (-8) | [Corroborated] | #8388 #8393 #8738 |
| AmmoType -> initSpeed | `0x364` | `0x38C` | [Corroborated] | #8311 #8380 #8392 #8393 |
| Entity -> IsDead | `0xE0` / `0xE2` | `0xE2` | [Corroborated] | #8380 #8393 #8435 #8675 |

### 1.4 Provenance warnings (mislabelled dumps)

Several widely quoted posts look like 1.29 data but are not.

- **#8303 (hejmeddig123xd; quoted in #8304)** is titled "DayZ 1.29 Enfusion Engine - Full SDK Offsets / Updated for 1.29 stable compatibility". It uses World `0xF4B050`, which is the **1.28** stable RVA, and was posted on 7 Apr, the day *before* 1.29 went stable. Every module RVA and function RVA in it should be treated as **[Needs Update for 1.29+]**. Its *struct layouts* (damage system, input controller, weather, grass) never got 1.29 confirmation in the thread.
- **#8299 (livelys)** "IDA verified" block is also World `0xF4B050`, Network `0xF5E190`, so it is **1.28**.
- **#8461** dumper output contains many `0x0` and nonsensical entries (e.g. `World::EyeAccom = 0x17C`, `isDead = 0xBE`) and is **[Unverified]**.
- **#8466** labelled "Verified Offsets Summary" is explicitly for the **Xbox/Microsoft** build ("xbox offsets... verify the offsets yourself"). Do not use it for Steam.
- **#8643** is a Microsoft-Store build dump with deltas against a Steam "Loader". Same caution applies.
- Automated **"UPDATER" dumps** (#8573, #8578, #8617, #8698) resolve signatures blindly. Some entries are clearly wrong in the same run (e.g. `Animation::MatrixArray -> 0x88` and `Ammo::AirFriction -> 0x90` in #8617, contradicted by #8652 and #8654). Use their **module globals**, which match independent reports, and treat odd member offsets in them with suspicion.

---

## 2. Engine Primitives

### 2.1 ArmaString

Length-prefixed string object. Name fields store a **pointer** to one of these.

| Member | Offset | Type | Status | Posts |
|---|---|---|---|---|
| length | `+0x08` | `int32` | [Corroborated] | #8281 #8377 #8393 #8710 |
| chars | `+0x10` | `char[length]` | [Corroborated] | #8281 #8377 #8658 #8722 |

```cpp
// Read a name field: field holds ArmaString*
std::string ReadArmaString(uintptr_t strObj) {
    int32_t len = Read<int32_t>(strObj + 0x8);
    return ReadString(strObj + 0x10, len);
}
```

A frequent bug (#8678): reading the length at `+0x0` and the text at `+0x4` returns garbage. Use `+0x8` / `+0x10`.

### 2.2 AutoArray / pointer tables

Generic `{ T* data; int32 count; int32 alloc; }` layout (#8537):

```
+0x00  T*      data
+0x08  uint32  count
+0x0C  uint32  allocated
```

The **slow** and **item** tables use a different, slotted layout. See [S4.2](#42-entity-tables).

### 2.3 Weak pointers & the 0xA8 identity offset

The local-player link in World points to a *list/weak-pointer object*. `+0x8` holds a pointer that sits **0xA8 bytes into** the entity, so subtract `0xA8` to get the entity base (#8299 decompile note, #8340, #8510, #8620). The camera manager's active node is also described as a weak-pointer slot (#8611).

### 2.4 RTTI inspection trick

Read the MSVC RTTI behind an object's vtable to name unknown pointers (#8661):

```python
vtable   = read_ptr(obj)
col      = read_ptr(vtable - 0x8)           # CompleteObjectLocator
sig      = read_i32(col + 0x0)              # must be 1 on x64
td_rva   = read_i32(col + 0xC)
self_rva = read_i32(col + 0x14)
image    = col - self_rva
name     = read_cstr(image + td_rva + 0x14) # ".?AVClassName@@" -> strip suffix
```

You can sweep an object's offsets and print every one that yields valid RTTI to map its children.

---

## 3. External vs Internal: Capability Matrix

"External" means a separate process reading and writing game memory (kernel driver, hypervisor or DMA card; several posters use DMA). "Internal" means code running inside `DayZ_x64.exe`.

| Capability | External | Internal | Notes / sources |
|---|---|---|---|
| Read world, entity tables, positions, names, inventory | [x] | [x] | All offset chains in this wiki are pure reads. |
| Bone positions / skeleton | [x] | [x] | Same chain both ways (#8396: "im external but the chain is the same for internal"). |
| World-to-screen | [x] (reimplement math) | [x] (math, or call engine `GetScreenPosRelative`-style functions) | #8477 asks about using the camera class's own projection function internally. |
| Player names / Steam IDs (scoreboard) | [x] | [x] | S8 |
| Writing scalar engine state (eye accommodation, grass, time scale, third-person/crosshair flags) | [x] | [x] | Simple writes. Third-person write shown internally (#8441) and used externally (#8442). |
| Writing bullet position / visual state | [x] (needs high write frequency) | [x] | #8684-#8687, #8726. Range limits are reported (S10). |
| Freecam | [x] via data writes only (#8611), or vtable swap (#8470) | [x] | #8611 argues the data-only approach avoids patching the image. |
| Calling engine functions (raycast, `SetObjectMaterial`, config/script calls) | [NO] (no code execution) | [x] | #8394, #8439, #8453, #8458 |
| Visibility check | [!] Possible only by **rebuilding** physics geometry from memory and raycasting yourself (e.g. Intel Embree) | [x] engine raycast | External: #8494, #8496, #8518, #8703-#8707. Internal: S13.1 |
| Chams / material swap | [!] none shown | [x] via `SetObjectMaterial` | #8453, #8489 |
| Script VM / `FindClassByName` | [NO] | [x] | #8408, #8428, #8432 |
| Mouse-style aimbot | [x] via OS mouse input | [x] | #8498 |
| View-angle (vector) aimbot | **[Conflicting]**. #8498: "no way to do it externally unless you are doing shellcode". #8504: aim-override fields can be written externally, "No shellcode needed". | [x] | Input-controller layout in #8303 is 1.28 data -> [Needs Update for 1.29+] |
| Reading other players' **health** | [!] GameVariables may contain networked values (S12.1) - [Needs Update for 1.29+] | [!] Same | #8367: authoritative health is server-side. GameVariables blood/health readable in older builds; needs 1.29 verification. Item *Quality* at `entity + 0x194` always works. |
| **Bleeding detection** | [x] via `bleedingeffects` in GameVariables | [x] | #499, #852, #4346. Read the GameVariables table and check `bleedingeffects > 0`. Works externally. (S12.2) |
| **Unconscious detection** | [x] via `shock >= blood - 550` formula | [x] | #2571, #2742. Read blood and shock from GameVariables. (S12.3) |
| Other players' stance | [NO] (reported not networked) | [NO] | #8285, #8289 |

---

## 4. World

`World* world = Read<uintptr_t>(module_base + WORLD_RVA);` For the RVA per build, see S1.2.

### 4.1 World member map

| Member | Offset | Type | Status | Posts |
|---|---|---|---|---|
| BulletTable | `0xE00` | AutoArray of bullet entity ptrs | [Corroborated] | #8380 #8393 #8444 #8537 #8573 |
| BulletCount | `0xE08` | `uint32` | [Corroborated] | #8393 #8537 #8573 |
| Grass (offline / single-player) | `0xBF0` | - | [Corroborated] | #8393 #8573 #8717 |
| Grass (online) / "NoGrass" | `0xC00` | float (write `0` for no grass) | [Corroborated] | #8380 #8393 #8573 #8717 |
| Camera | `0x1B8` | `Camera*` | [Corroborated] | #8311 #8380 #8393 #8573 #8675 |
| NearEntList / size | `0xF48` / `0xF50` | ptr / `int32` | [Corroborated] | #8383 #8393 #8573 #8661 |
| FarEntList / size | `0x1090` / `0x1098` | ptr / `int32` | [Corroborated] | #8383 #8393 #8573 #8661 |
| SlowEntList / capacity | `0x2010` / `0x2018` | slotted table | [Corroborated] | #8393 #8573 #8717 |
| ItemList / capacity / size | `0x2060` / `0x2068` / `0x2070` | slotted table | [Corroborated] (see S4.2) | #8393 #8573 #8630 #8717 |
| Second item hash | `0x2080` / `0x2088` | "drops / inventoryItem" | [Single-source] | #8501 |
| Slow entity live count | `0x2020` | `int32` | [Single-source] | #8501 |
| LocalPlayer link | `0x2960` | weak-ptr list | **[Conflicting]** with `0x2958` | S4.3 |
| PlayerOn | `0x2968` | - | [Corroborated] | #8311 #8573 |
| EyeAccom | `0x296C` | float | [Corroborated] | #8339 #8393 #8573 |
| Hour / TimeScale | `0x2970` | float | [Corroborated] | #8405 #8416 #8573 |
| Day | `0x2974` | - | [Corroborated] | #8470 #8573 |
| DayTime | `0x2978` | - | [Corroborated] | #8501 #8573 |
| WeatherController | `0x7460` | ptr | [Corroborated] (from Jun 2026 build) | #8485 #8501 #8573 #8717 |

> **Table naming inversion.** Some posters label `0xF48` as "Far" and `0x1090` as "Near" (#8329, #8344, #8438). The majority, and every signature-based dump, use **`0xF48` = Near, `0x1090` = Far** (#8383 #8393 #8463 #8573 #8661). The near table often holds a single entry (yourself) when nobody is close (#8661). Far covers entities beyond ~100 m (#8667).

### 4.2 Entity tables

**Near / Far**: a plain pointer array (#8661).

```cpp
uintptr_t list  = Read<uintptr_t>(world + 0xF48);   // or 0x1090
int32_t   count = Read<int32_t>(world + 0xF50);     // or 0x1098
for (int i = 0; i < count; ++i) {
    uintptr_t ent = Read<uintptr_t>(list + i * 0x8);
}
```

**Slow / Item**: a slotted hash-style table with **0x18-byte** slots (#8393: `entry_size 0x18`, `entry_ptr 0x8`; #8630 worked example).

```
table + 0x00  slot*   data
table + 0x08  int32   capacity   (iterate this many slots)
table + 0x10  int32   size       (valid entries)
slot  + 0x00  uint32  state      (1 = valid; 0 / 2 = skip)
slot  + 0x08  Entity* entity
```

```python
# #8630 (leonardgilmore) - works for World+0x2060 (items) and World+0x2010 (slow)
data = read_ptr(world + off); cap = read_i32(world + off + 0x8)
for i in range(cap):
    slot = data + i * 0x18
    if read_u32(slot) != 1: continue
    items.append(read_ptr(slot + 0x8))
```

> [Conflicting] Some configs put the item-table count at `+0x8` (#8393 `slow_table_size 0x2018`, #8602 `ItemTable + 0x08`). #8630 found `+0x8` is **capacity** and `+0x10` is **size**. #8717's working 1.29 config agrees (`ItemTableCountAlloc 0x2068`, `ItemTableCount 0x2070`).

**What lives where** (as reported):
- Near/Far: players, infected, animals, vehicles (#8667).
- Slow: buildings and points of interest within ~1 km, including heli crashes and convoys (class `house` / `DayZBuilding`), plus **corpses** (#8449 #8454 #8590 #8599 #8661 #8435).
- Item: loose loot. Parse the slow table as well to see everything (#8667). One poster used `Entity+0x78 != 0` to skip picked-up items (#8502). [Single-source]

**Bullet table** (`World+0xE00`) is an AutoArray of bullet entities. Each bullet has the normal Entity layout (VisualState at `+0x1C8`, position `+0x2C`) (#8491 #8537 #8684). A bullet table that always reads empty was reported on a custom server (#8536). Unresolved.

### 4.3 Resolving the local player

```cpp
uintptr_t lp_list = Read<uintptr_t>(world + 0x2960);  // or 0x2958, see conflict
uintptr_t ident   = Read<uintptr_t>(lp_list + 0x8);
uintptr_t player  = ident ? ident - 0xA8 : 0;
```

**[Conflicting] `0x2960` vs `0x2958`:**
- `0x2960` works: #8351 (1.29 stable, Apr), #8393, #8510 (Jun), #8573 dump, #8622 (Aug), #8675 (Sept).
- `0x2958`: dumpers #8268 #8461 #8576. A decompiled getter on the Aug build reads `*(*(a1 + 0x2958) + 8) - 0xA8` (#8620). #8678 (Sept) says `0x2958` fixed their chain.
- Both values appear in working Aug-Sept code. The thread doesn't settle which getter (or which `this`) each refers to. **Verify against your build.**

Alternative chain listed in configs: `0x2960 -> +0x8 -> +0x5C8 -> +0x28` (#8470 #8472). [Unverified] There is also a "player-controlled entity" offset `0x430` (#8675). [Single-source]

---

## 5. Entity / EntityAI

### 5.1 Entity member map

| Member | Offset | Type | Status | Posts |
|---|---|---|---|---|
| vtable | `0x0` | - | - | #8661 |
| *(flag used for filtering)* | `0x0` (u16) | - | [Unverified] (1.28 era) | #8274 |
| "valid / not picked up" check | `0x78` | ptr | [Single-source] | #8502 |
| Owner | `0xA0` | ptr | [Unverified] (dumper) | #8573 |
| Identity sub-object | `+0xA8` | - | [Corroborated] | S2.3 |
| IsDead | `0xE2` | `bool` | [Corroborated] | #8380 #8393 #8435 #8573 #8675 |
| FutureVisualState | `0x120` | `VisualState*` | [Corroborated] | #8311 #8573 #8637 #8675 |
| EntityDead | `0x15D` | byte | [Unverified] (dumper) | #8311 #8573 |
| EntityType | `0x180` | `EntityType*` | [Corroborated] | #8380 #8393 #8410 #8573 #8658 |
| Quality (items: 0 Pristine -> 4 Ruined) | `0x194` | `int32` | [Corroborated] | #8374 #8393 #8675 |
| VisualState (render) | `0x1C8` | `VisualState*` | [Corroborated] | #8380 #8393 #8573 #8675 |
| LodShape | `0x200` | ptr | [Corroborated] | #8481 #8573 |
| SortObject | `0x228` | ptr | [Unverified] | #8573 |
| SprintFlag | `0x3AD` | byte | [Unverified] (dumper) | #8501 #8573 |
| Inventory | `0x650` | `Inventory*` | [Corroborated] | #8366 #8393 #8410 #8650 |
| Stamina | `0x6A4` | - | [Unverified] (dumper) | #8501 #8573 |
| NetworkID | `0x6DC` | `uint32` | [Corroborated] | #8380 #8393 #8587 #8722 |
| Player::StatsContainer | `0x6F0` | ptr | [Unverified] | #8573 #8643 |
| Player::DamageManager | `0x700` | ptr | [Unverified] | #8573 #8643 |
| DayZPlayer::Skeleton | `0x7E0` | ptr | [Corroborated] | S7 |
| DayZInfected::Skeleton | `0x670` | ptr | [Corroborated] | S7 |
| Player::InputController | `0x7E8` | - | [Unverified] (dumper; #8720 unsure between `0x7E8` and `0x828`) | #8501 #8573 #8720 |
| Object material array / count | `0x558` / `0x560` | `Material**` / `int32` | [Single-source] | #8489 #8501 |
| Hidden-selection state | `0x528` | `byte[]` | [Single-source] | #8489 |
| InventoryItem quantity | `0x848` | `float` | [Corroborated] | #8690 #8738 |

### 5.2 VisualState & transforms

| Member | Offset | Status | Posts |
|---|---|---|---|
| Transform (3x4 world matrix, 12 floats) | `0x08` | [Corroborated] | #8396 #8401 #8557 #8573 |
| Direction / forward | `0x20` | [Corroborated] | #8393 #8438 |
| Position (`Vector3`) | `0x2C` | [Corroborated] | #8351 #8393 #8438 #8561 |
| Velocity | `0x54` | [Single-source] for 1.29 | #8438 |
| InverseTransform | `0xA4` | [Corroborated] (dumpers) | #8268 #8298 #8643 |

The transform is column-major 3x4. Elements `[0..8]` are the basis (right/up/forward) and `[9..11]` are the translation, i.e. the world position. The position at `+0x2C` equals `[9..11]` of the matrix that starts at `+0x08` (#8396 #8557).

```cpp
Vector3 pos = Read<Vector3>(Read<uintptr_t>(entity + 0x1C8) + 0x2C);
```

### 5.3 EntityType (class info & names)

`EntityType* t = Read<uintptr_t>(entity + 0x180);` Every name field is an `ArmaString*`.

| Member | Offset | Example | Status | Posts |
|---|---|---|---|---|
| ObjectName | `0x70` | - | [Conflicting]: `0x70` (#8617 #8643), `0x6C` (#8501), `0x98` (#8311) | |
| CleanNameInternal / ClassName | `0x98` | - | [Unverified] | #8501 #8470 |
| ModelName (p3d path) | `0xB0` | `dz\gear\consumables\9v_battery.p3d` | [Corroborated] | #8438 #8642 #8645 |
| TypeName / ConfigName / CategoryName | `0xD0` | `dayzplayer`, `dayzinfected`, `car`, `house`, `inventoryItem`, `clothing`, `Weapon` ... | [Corroborated] | #8393 #8565 #8645 #8658 #8660 |
| CleanName (display name) | `0x518` | `Bandage`, `Fence`, `Wooden Crate` | [Corroborated] | #8364 #8393 #8410 #8630 #8658 |

```text
type      = [entity + 0x180]
typename  = [type + 0xD0] + 0x10      // char*
cleanname = [type + 0x518] + 0x10     // char*
```
(#8658)

### 5.4 Classifying entities & loot

- Primary switch is on **TypeName (`+0xD0`)**: `dayzplayer`, `dayzinfected`, `dayzanimal`, `car`, `boat`, `clothing`, `itemoptics`, `Weapon`, `house`, `inventoryItem` (#8565 #8645 #8660).
- Secondary switch is on **ModelName (`+0xB0`)** substrings: `backpacks`, `food`, `drinks`, `ammunition`, `firearms`, `camping`, `furniture`, `melee`, `explosives`, `medical`, `containers`, `cooking` (#8565 #8645).
- In 1.29, entity type strings moved from generic names toward the **class names of survivors**, while `car` and animals stayed the same (#8315).
- **Heli crashes**: config `house` and TypeName one of `Wreck_UH1Y`, `Wreck_Mi8`, `Wreck_Mi17`, `Wreck_Helicopter*`, `Wreck_AH1Z`, `Wreck_Ka52`, `Wreck_Mi24` (#8454). An alternative is to match `dz\structures\wrecks\aircraft` in the model path (#8284).
- **Convoys / dynamic events**: `DayZBuilding` names ending `BRDM_DE` / `Ural_DE` (#8590). Alternatively filter on `wreck_decal` (#8599).
- **Bots vs players**: names like `Survivor (1)` belong to players without a custom name (#8548 #8553). A valid Steam ID is the simplest way to separate bots from humans (#8377).

---

## 6. Camera, Projection & World-to-Screen

### 6.1 Camera member map

`Camera* cam = Read<uintptr_t>(world + 0x1B8);`

| Member | Offset | Type | Status | Posts |
|---|---|---|---|---|
| vtable | `0x00` | - | - | #8470 |
| InvertedViewRight | `0x08` | `Vector3` | [Corroborated] | #8438 #8477 #8637 #8675 #8717 |
| InvertedViewUp | `0x14` | `Vector3` | [Corroborated] | same |
| InvertedViewForward | `0x20` | `Vector3` | [Corroborated] | same |
| InvertedViewTranslation (camera position) | `0x2C` | `Vector3` | [Corroborated] | same |
| Zoom vector | `0x38` | - | [Single-source] | #8675 |
| ViewportSize | `0x58` | `Vector2` | [Corroborated] (#8637 using `0x4` was called wrong in #8638) | #8477 #8717 |
| FOV / aspect-ratio factor | `0x6C` / `0x70` | float | [Conflicting]: #8407 says both are `0x70`, corrected to aspect `0x6C` in #8420 #8421. #8675 calls `0x70` "zoom factor". | |
| ProjectionD1 | `0xD0` | `Vector3` (use `.x`) | [Corroborated] | #8438 #8477 #8717 |
| ProjectionD2 | `0xDC` | `Vector3` (use `.y`) | [Corroborated] | #8438 #8477 #8717 |
| State flags (freecam detach) | `0x1A8` | bitmask | [Single-source] | #8470 |
| Update interval / throttle | `0x1C0` | float | [Single-source] | #8470 |

ReClass layout confirmed in-game (#8477):

```cpp
class CameraClass {
    char    pad_0000[0x8];
    Vector3 Right;          // 0x08
    Vector3 Up;             // 0x14
    Vector3 Forward;        // 0x20
    Vector3 Position;       // 0x2C
    char    pad_0038[0x20];
    Vector2 ViewPortSize;   // 0x58
    char    pad_0060[0x70];
    Vector3 ProjectionD1;   // 0xD0
    Vector3 ProjectionD2;   // 0xDC
};
```

> #8477 got garbage from this layout because their `GetCurrentCamera` returned the *weak object*, not the instance. Make sure you dereference the real `Camera*`.

### 6.2 World-to-screen

The formula agreed across #8477 and #8565 (both 1.29):

```cpp
bool W2S(const Vector3& world, Vector2& out, float W, float H) {
    Vector3 d = world - cam.Position;                 // +0x2C
    float x = Dot(d, cam.Right);                      // +0x08
    float y = Dot(d, cam.Up);                         // +0x14
    float z = Dot(d, cam.Forward);                    // +0x20
    if (z < 0.65f) return false;                      // behind / too close
    float nx = (x / cam.ProjectionD1.x) / z;          // +0xD0 .x
    float ny = (y / cam.ProjectionD2.y) / z;          // +0xDC .y
    out.x = (W * 0.5f) + nx * (W * 0.5f);
    out.y = (H * 0.5f) - ny * (H * 0.5f);
    return true;
}
```

Common mistakes reported:
- Using `ProjectionD2.x` instead of `.y`, or negating `z` (#8302 compares old and new code).
- Reading camera fields at the wrong offsets (`0x30/0x3C/0x48/0x54` in #8302 are not the 1.29 layout).
- **Scoped ESP drift.** Bones drift when zoomed through a scope (#8262 #8270). This is attributed to the scope changing the projection (#8271). Read the projection every frame. A dedicated `ScopeFovCtx` global exists in later dumps (S1.2). A complete fix was never posted. **[Open]**

### 6.3 FOV / zoom

Field names and values for 1.29.162510 come from #8393 (`camera_fov`), mirrored in #8470. **The control flow below is this wiki's reading of those field names, not posted code. [Unverified]**

```
ctx            = module + 0x1008CE0          // context_rva
if [ctx + 0x88] (indirect_active) ...
   cam_obj     = [ctx + 0x118]               // direct_ptr
else
   cam_obj     = [[ctx + 0x80] + 0x10]       // root_ptr -> root_nested
vertical_fov   = [cam_obj + 0x4C] * [cam_obj + 0x50]   // default product ~= 0.74176502
```

This is **[Single-source]** in structure. The 1.28-era version (#8259) has the same `0x4C x 0x50` product, reached via `GetCurrentFov` string xrefs. The Microsoft-build dump (#8643) independently lists `CameraObject::Fov 0x50`, `FovFactor 0x4C`, `CameraManager::ActiveTracker 0x80`, `TrackerObject 0x10`, `BlendType 0x88`, `BlendCamera 0x118`. These match the chain above, so the *member* offsets are **[Corroborated]** across Steam and Microsoft builds. The *module RVA* `0x1008CE0` is build-specific (S1.2).

### 6.4 Camera manager (freecam-relevant structures)

From #8643 (Microsoft build; member offsets match the Steam chain in S6.3):

| Object | Member | Offset |
|---|---|---|
| CameraObject | Owner | `0x68` |
| CameraObject | LocalTransform | `0x18` |
| CameraObject | FovFactor / Fov | `0x4C` / `0x50` |
| CameraManager | ActiveTracker (node) | `0x80` |
| CameraManager | BlendType | `0x88` |
| CameraManager | BlendCamera (default camera) | `0x118` |
| Tracker node | TrackerObject | `0x10` |

#8611 explains the camera pipeline. `World::Update` composes `finalMatrix = state.localTM * state.ownerTM`, and `World::Render` pushes it to the render camera. `GetState()` returns the *default camera* while a script interpolation is active, otherwise `*activeNode->object`. Pointing the active node's object at the manager's default camera (null owner, so its transform passes through unchanged) and writing its local transform gives a detached camera **using only heap writes**. The post gives no offsets. The table above plausibly maps to its `OFF_ACTIVE_NODE` / `OFF_DEFAULT_CAMERA` / `OFF_NODE_OBJECT` / `OFF_CAM_OWNER` / `OFF_CAM_TRANSFORM`, but **that mapping is this wiki's inference and is [Unverified]**.

An older approach swaps camera vtable index 2 (update) with index 6 (orientation-only), clears flags `0x4C` and sets `0x02` at `cam+0x1A8`, and writes `-1.0f` to `cam+0x1C0` (#8470). **[Single-source]** #8611 notes this modifies the mapped image's vtable.

---

## 7. Skeleton & Bones

### 7.1 Pointer chain

```text
skeleton  = [entity + 0x7E0]      // DayZPlayer   (1.28: 0x7E8)
          = [entity + 0x670]      // DayZInfected (1.28: 0x678)
anim      = [skeleton + 0x118]    // AnimClass    (1.28: 0xB0)
bones     = [anim + 0xBE8]        // matrix array (1.28: 0xBF0)
bone_i    =  bones + 0x54 + i * 0x30   // local position (Vector3)
```

**[Corroborated]**: #8311 #8380 #8393 #8401 #8654 #8675 #8717.

| Conflict | Values | Posts |
|---|---|---|
| Skeleton -> AnimClass | `0x118` (majority) vs `0x110` | #8396 |
| Bone base / stride | `+0x54`, stride `0x30` (majority) vs base `+0x54`, stride `0x60` | #8557 |
| Bone base | `+0x54` vs reading the full 48-byte matrix at `bones + i*0x30` and taking floats `[9..11]` (= `+0x24`) | #8396 #8729 |

The last two rows describe the same data in two ways. `+0x24` is the translation inside a 48-byte bone matrix. `+0x54` = `0x30 + 0x24` works if the array has one extra matrix before bone 0, or if indices are 1-based. Mixing them shifts bone indices by one. That is the most likely source of the off-by-one index sets in S7.3. **Pick one convention and the matching index table.**

Also reported: a remap table at `anim + 0x7E4` (`uint32[]`, `0xFFFFFFFF` = absent) that only covers indices 0-23 on infected. Arms must use raw indices (#8396). [Single-source] `m_anim_class 0x90` appears alongside `0x118` in #8626, and in dumpers as `Skeleton::AnimClass2 0x90`. #8651 used it and got nothing. **[Unverified]**

### 7.2 Bone matrix layout & world transform

Each bone matrix is 48 bytes: `[0..8]` 3x3 rotation (column-major) and `[9..11]` local translation (#8396). Bone data is in **model space**. Multiply by the entity's world matrix (`[entity + 0x1C8] + 0x08`), or every bone collapses onto the entity origin (#8557):

```cpp
Vector3 BoneWorld(uintptr_t entity, int idx, bool player) {
    uintptr_t skel  = Read<uintptr_t>(entity + (player ? 0x7E0 : 0x670));
    uintptr_t anim  = Read<uintptr_t>(skel + 0x118);
    uintptr_t bones = Read<uintptr_t>(anim + 0xBE8);
    Vector3   b     = Read<Vector3>(bones + 0x54 + idx * 0x30);
    float m[12]; ReadBuf(Read<uintptr_t>(entity + 0x1C8) + 0x08, m, sizeof m);
    return { m[0]*b.x + m[3]*b.y + m[6]*b.z + m[9],
             m[1]*b.x + m[4]*b.y + m[7]*b.z + m[10],
             m[2]*b.x + m[5]*b.y + m[8]*b.z + m[11] };
}
```

The origin sits at the feet, with the head about 1.7 m above (#8557). Cache `skel`, `anim` and `bones` per entity, since only the vectors change each frame (#8557).

### 7.3 Bone indices - players

The DayZPlayer rig has **148 bones** (#8396). Two families of index tables circulate, offset by one from each other:

**Set A** (used with `bones + 0x54 + i*0x30`): **[Corroborated]** #8380 #8401 #8615 #8675 #8681

| Bone | Index | Bone | Index |
|---|---|---|---|
| Head | 24 | Pelvis | 0 |
| Neck | 21 | L / R hip | 1 / 9 |
| Spine2 | 20 | L / R knee | 4 / 12 |
| Spine1 | 19 | L / R foot | 6 / 14 |
| L / R shoulder | 61 / 94 | L / R elbow | 63 / 97 |
| L / R hand | 65 / 99 | | |

**Set B** (#8387 "boneindexes have not changed", #8396 raw indices): head 24, neck 22, spine3 21, pelvis 1, L/R arm 62/95, L/R forearm 64/98, L/R hand 66/100, L/R upleg 2/10, L/R leg 5/13, L/R foot 7/15. #8396 adds: spine 18/20/21, knee twists 5/13 and arm twist 96 (skip these).

Notable extras: bone **49** ~= between the eyes or front of the head; 24 sits toward the back of the head (#8614 #8618). Eyes are **37 / 38** (#8729). An unrelated low-numbered set (pelvis 0, spine 9-12, arms 31-34 / 47-50) appears in #8557. That set is **[Conflicting]** with every other source.

### 7.4 Bone indices - infected

The DayZInfected rig has **92 bones** (#8396). There is no consensus:

| Source | Head | Neck | Spine2 | Spine1 | Pelvis | L/R shoulder | L/R elbow | L/R hand | L/R hip | L/R knee | L/R foot |
|---|---|---|---|---|---|---|---|---|---|---|---|
| #8401 (Apr) | 21 | 19 | 18 | 17 | 15 | 24 / 56 | 25 / 59 | 27 / 60 | 87 / 88 | 11 / 4 | 13 / 6 |
| #8615 (Aug) | 21 | 19 | 16 | 15 | 0 | 24 / 56 | 25 / 59 | 27 / 60 | 1 / 9 | 4 / 12 | 6 / 13 |
| #8675 (Sep) | 22 | 19 | - | - | 0 | 24 / 56 | 26 / 59 | 28 / 61 | - | 4 / 10 | 6 / 13 |
| #8396 (raw, Apr) | 23 | 19 (branch) | 18 | 16 | 1 | 25 / 57 | 27 / 60 | 30 / 63 | 2 / 9 | 4 / 11 | 7 / 14 |

**[Conflicting]** Several posters report broken infected skeletons whichever table they use (#8665 #8668). The recommended method is to render every bone index on an infected entity and map the indices by eye (#8545 #8668).

### 7.5 Skeleton drawing bone pairs

For rendering skeleton lines, use bone index pairs (#8359 #8681):

```cpp
// Player skeleton bone pairs (connect bone A to bone B)
static const int playerBones[][2] = {
    { 0, 4 }, { 4, 6 },             // spine
    { 0, 12 }, { 12, 14 },          // legs (one side)
    { 0, 21 }, { 21, 24 },          // spine to shoulder
    { 21, 61 }, { 61, 63 }, { 63, 65 },  // left arm
    { 21, 94 }, { 94, 97 }, { 97, 99 }   // right arm
};

// Zombie/Infected skeleton bone pairs
static const int zombieBones[][2] = {
    { 0, 4 }, { 4, 6 },             // spine
    { 0, 10 }, { 10, 13 },          // legs (one side)
    { 0, 19 }, { 19, 22 },          // spine
    { 19, 24 }, { 24, 26 }, { 26, 28 },  // left arm
    { 19, 56 }, { 56, 59 }, { 59, 61 }   // right arm
};
```

**Note:** These indices are version-dependent and may conflict with S7.3/S7.4 tables. If skeleton renders incorrectly, iterate all bone indices visually to find correct pairs for your build.

---

## 8. Network, Scoreboard & Player Identity

```text
nm        =  module + NETWORK_MANAGER_RVA      // an object, NOT a pointer (#8341 #8722)
client    = [nm + 0x50]
scoreboard= [client + 0x18]                    // PlayerIdentity*[]
count     = (int32)[client + 0x24]
for i < count:
    id    = [scoreboard + i*8]
    netid = (uint32)[id + 0x30]                // match to entity+0x6DC
    name  = ArmaString [id + 0xF8]
    steam = ArmaString [id + 0xA0]
```

**[Corroborated]**: #8587 #8624 #8722 #8676->#8682. Matches 1.28 except for the entity NetworkID (`0x6E4` -> `0x6DC`).

| Member | Offset | Status | Posts |
|---|---|---|---|
| NetworkManager -> NetworkClient | `0x50` | [Corroborated] | #8341 #8587 #8624 #8722 |
| NetworkClient -> Scoreboard | `0x18` | [Corroborated] | #8587 #8624 #8722 |
| NetworkClient -> PlayerCount | `0x24` | [Corroborated] | #8377 #8587 #8624 #8722 |
| NetworkClient -> "ClientIdSize" | `0x170` | **[Conflicting]**. Listed by dumpers, but #8377 says using it as the scoreboard size "is wrong". | #8311 #8377 #8393 |
| NetworkClient -> MissionHeader | `0x28` | [Single-source] | #8437 |
| NetworkClient -> ThirdPerson flag | `0x9C` | [Corroborated] (written as `!enabled`) | #8441 #8573 #8617 |
| NetworkClient -> Crosshair flag | `0xA0` | [Corroborated] | #8441 #8573 #8617 |
| MissionHeader -> is_third_person_disabled / is_crosshair_disabled | `0x74` / `0x78` | [Single-source] alternative to `0x9C`/`0xA0` | #8437 |
| NetworkClient -> ServerName | `0x308` | [Corroborated] (earlier values: `0x360` #8311 #8388, `0x338` #8397) | #8390 #8573 |
| NetworkClient -> GameVersion | `0x350` | [Corroborated] | #8390 #8573 |
| NetworkClient -> Ping | `0x33C` | [Unverified] | #8470 #8573 |
| NetworkClient -> MapName | `0x38` | [Single-source] | #8393 |
| PlayerIdentity -> NetworkID | `0x30` | [Corroborated] | #8587 #8722 |
| PlayerIdentity -> GUID | `0x38` | [Unverified] ("untested") | #8311 #8501 |
| PlayerIdentity -> SteamID | `0xA0` | [Corroborated] | #8524 #8587 #8722 |
| PlayerIdentity -> Name | `0xF8` | [Corroborated] | #8587 #8624 #8722 |
| PlayerIdentity -> PlainName | `0x100` | [1.30-exp] | #8710 |

Common pitfalls:
- **Double `+0x50`** (`base + NM + 0x50` and then `+0x50` again) (#8377).
- **Dereferencing the NetworkManager RVA.** It isn't a pointer, so `[module + RVA]` reads 0 (#8331 #8341 #8722 #8680->#8682).
- **Xbox/Microsoft build.** No working name chain was posted. #8643 reports `PlayerIdentity::Name = 0xB0` and a failed NetworkManager scan for that build. [Unverified]

---

## 9. Inventory, Cargo & Items

```text
inv          = [entity + 0x650]                 // also for items (nested inventories)
hands        = [inv + 0x1B0]                    // Entity* in hands
hands_valid  = (bool)[inv + 0x1CC]
equipped     = [inv + 0x150]                    // attachments / clothing slots
equip_count  = (uint16)[inv + 0x15C]
  item_i     = [equipped + 0x8 + 0x10*i]
cargo        = [inv + 0x148]                    // CargoGrid*
cargo_items  = [cargo + 0x38]
cargo_count  = (uint16)[cargo + 0x44]
  item_i     = [cargo_items + 0x8 + 0x10*i]
```

**[Corroborated]**: #8366 #8393 #8410 #8650 #8654 #8736. Weapon attachments use the same `inv+0x150 / +0x15C` structure on the weapon's own inventory (#8736).

| Conflict | Values | Posts |
|---|---|---|
| CargoGrid item count | `0x44` (#8393 #8650 nested_cargo_count) vs `0x40` | #8717 |
| Equipped count | `0x15C` (#8366 #8393 #8650) vs `0x158` ("slot_count_alt") | #8485 #8717 |
| Hands valid | `0x1CC` (#8637 #8717) | #8698 has the 1.30 variant |

CargoGrid width and height: `0x58` / `0x5C` (#8717). [Single-source]

Get the name of the item in hands (#8410):

```cpp
auto inv   = Read<uintptr_t>(player + 0x650);
auto item  = Read<uintptr_t>(inv + 0x1B0);
auto type  = Read<uintptr_t>(item + 0x180);
auto name  = ReadArmaString(Read<uintptr_t>(type + 0x518));
```

---

## 10. Weapons, Magazines & Ammunition

### 10.1 Weapon / chamber

| Member | Offset | Status | Posts |
|---|---|---|---|
| Chamber / muzzle array | `weapon + 0x6A8` | [Corroborated] | #8393 #8687 #8741 |
| Chamber entry stride | `0x100` | [Corroborated] | #8393 #8573 |
| Entry -> AmmoType* | `+0x20` | [Corroborated] (must reload after login for it to resolve, #8692) | #8393 #8687 |
| Entry -> jam value | `+0x18` | [Unverified] | #8393 |
| Entry -> internal-magazine max / current | `+0x58` / `+0x5C` | [Single-source] (latest post in thread) | #8741 |
| InitSpeedMultiplier | `0x960` | [Unverified] (1.28 data in #8303 -> [Needs Update for 1.29+]) | #8303 #8734 |

`Weapon + 0x6B0` reads values like `0x100000001`, which is not a pointer (#8687). #8734 reports that `AttachmentsArray`, `AmmoCapacityA/B` and `ChamberedPtr` from dumps don't behave as named. **[Unverified]**

### 10.2 Magazine & item quantity

Magazines and ammo piles both use class `Magazine` (#8738).

| Member | Offset | Type | Status | Posts |
|---|---|---|---|---|
| Magazine current ammo | `0x6AC` | `uint16` | [Corroborated] | #8388 #8393 #8738 |
| Magazine max ammo | `0x6A8` | `uint16` | [Corroborated] | #8388 #8738 |
| Magazine capacity A / B | `0x6B0` / `0x6B4` | - | [Unverified] (#8734: "stay at 1") | #8393 |
| InventoryItem quantity | `0x848` | `float` | [Corroborated] | #8690 #8738 |
| InventoryItem max quantity | `0x858` (#8738) vs `0x850` (= +0x8, #8690) | `uint32` | [Conflicting] | |

Dumpers also list `Magazine::AmmoCount 0x3B0` / `MaxAmmo 0x3A4` (#8470 #8501 #8617). These conflict with the hand-verified `0x6AC` / `0x6A8`. **[Conflicting]**

### 10.3 AmmoType config layout

Reached through `[weapon + 0x6A8] + 0x20` (#8687). The most detailed layout is #8392 (KingJulianz, 18 Apr 2026, 1.29 stable). Treat it as the reference.

| Offset | Field | Offset | Field |
|---|---|---|---|
| `0x370` | hit | `0x3BC` | explosive |
| `0x374` | indirectHit | `0x3C0` | caliber |
| `0x378` | indirectHitRange | `0x3C4` | deflecting (as sin) |
| `0x37C` | indirectHitRangeMultiplier | `0x3C8` | deflectingMultiplier |
| `0x380` / `0x384` | indirectHitAngle1 / 2 | `0x3CC` | dispersion |
| `0x388` | explosionType (0 radial, 1 directional) | `0x3D0` | projectilesCount |
| `0x38C` | **initSpeed** | `0x3D4` | deflectionSlowDown |
| `0x390` | 1 / initSpeed | `0x3D8` | timeToLive |
| `0x394` | maxLeadSpeed | `0x3DC` | **airFriction** |
| `0x398` / `0x39C` | typicalSpeed / typicalSpeed^2 | `0x3E0` | airFrictionChangeOnActivation |
| `0x3A4` | initTime | `0x3E4` | coefGravity |
| `0x3A8` | explosionTime | `0x408` / `0x40C` / `0x410` | tracerScale / tracerStartTime / tracerEndTime |
| `0x3AC` | fuseDistance | `0x414` | nvgOnly |
| `0x3B0` | slowdownThreshold | `0x418` / `0x41C` | damageBarrel / damageBarrelDestroyed |
| `0x3B4` | maxSpeed | `0x420` / `0x424` | jamChance / jamChanceDestroyed |
| `0x3B8` | simulationStep | `0x428` | dmgPerUse |

Derived timing fields: `0x738` TTLx100, `0x73C` explosion timing, `0x740` initTimex100, `0x744` = 200 when type is illuminating (#8392).

**Status:** [Single-source] for the full table. `initSpeed 0x38C` is **[Corroborated]** (#8380 #8393 #8463 #8573 #8726). `airFriction 0x3DC`, `caliber 0x3C0`, `dispersion 0x3CC`, `hit 0x370` and `fuseDistance 0x3AC` match the signature dumps (#8463 #8485 #8573).

**[Conflicting]:** several configs list `air_friction 0x3B4`, `dispersion 0x3A4` and `coefGravity 0x3BC` (#8388 #8393 #8573 "AmmoType::*"). In #8392's table those offsets are `maxSpeed`, `initTime` and `explosive`. The dumper keeps *both* `Ammo::AirFriction 0x3DC` and `AmmoType::AirFriction 0x3B4`. The thread never resolves this. #8392 is the only post that gives a complete, self-consistent layout.

Observed behaviour (#8692): writing `initSpeed` changes bullet speed and impact-effect size, and the change persists across reconnects. `coefGravity` read as 0. [Single-source]

### 10.4 Bullet manipulation / "Magic Bullet" / Silent Aim

The bullet table at `World + 0xE00` (S4.2) is an AutoArray of live bullet entities. Each bullet has a standard VisualState at `+0x1C8` with position at `+0x2C`.

**Basic technique** (#8684 #8687):
```cpp
// Get bullet visual state and write target position
uintptr_t visualState = Read<uintptr_t>(bullet + 0x1C8);
Write<vec3>(visualState + 0x2C, targetHeadPos);
```

**50m teleport limit / server validation** (#8687 #8726 #8734):
- Bullets cannot be teleported more than ~50 meters in a single frame
- Server-side validation rejects larger position changes
- The bullet continues its original trajectory beyond the teleport
- You CANNOT shoot through walls at >50m - bullet must travel naturally to within 50m of target first

#### 10.4.1 Validation bypass: InitSpeed boost

The standard bypass for the 50m limit is to boost the bullet's `initSpeed` so it travels to within 50m of the target almost instantly, then teleport it the final distance (#8127 #8128 #8491 #8687).

**Full chain to reach AmmoType InitSpeed:**
```cpp
// 1.29 offsets - verify for your build
uintptr_t inventory = Read<uintptr_t>(localPlayer + 0x650);  // Entity -> Inventory
uintptr_t hands = Read<uintptr_t>(inventory + 0x1B0);        // Inventory -> InHands (weapon)
uintptr_t ammoType1 = Read<uintptr_t>(hands + 0x6B8);        // Weapon -> AmmoType ptr (alt: 0x6B0)
uintptr_t ammoType2 = Read<uintptr_t>(ammoType1 + 0x20);     // -> actual AmmoType config
// InitSpeed is at AmmoType + 0x38C (1.29), was 0x364 in older versions
```

**Complete silent aim with validation bypass** (#8491):
```cpp
for (uint64_t bullet : BulletsList()) {
    // Boost initSpeed based on distance
    uint64_t inventory = Read<uint64_t>(localPlayer + 0x650);
    uint64_t hands = Read<uint64_t>(inventory + 0x1B0);
    uint64_t ammoType1 = Read<uintptr_t>(hands + 0x6B8);
    if (ammoType1) {
        uint64_t ammoType2 = Read<uintptr_t>(ammoType1 + 0x20);
        if (ammoType2) {
            vec3 bulletPos = GetPosition(bullet);
            float distance = bulletPos.Distance(targetPos);
            float boostSpeed = distance * 100.0f;  // or 1000.0f for more aggressive
            Write<float>(ammoType2 + 0x38C, boostSpeed);
        }
    }
    // Teleport bullet to target
    uintptr_t visualState = Read<uintptr_t>(bullet + 0x1C8);
    Write<vec3>(visualState + 0x2C, targetHeadPos);
}
```

**How it works** (#8734):
1. Fire the weapon normally (must have line-of-sight or at least be able to shoot toward target)
2. The boosted InitSpeed makes the bullet reach within 50m of target almost instantly
3. Once within 50m, the position write teleports it directly to the head
4. Server accepts the hit because the position delta is <50m

**Side effects** (#8692):
- Impact effects (blood, dust, smoke) scale with bullet speed - very visually obvious at high speeds
- InitSpeed changes persist even after disconnect/reconnect (until full game restart)
- Does NOT let you shoot through walls beyond 50m - the bullet must physically travel

**Alternative approaches:**
1. **High-frequency position spam**: Write target position every frame at very high rate (#8687) - works for close range
2. **Step teleportation**: Break path into 50m increments per tick (theoretical) (#8692)
3. **Aim toward target + boost**: Just boost speed, aim roughly at target, bullet arrives before you need to lead (#8692)

**[Corroborated]** bullet position writing, 50m limit, InitSpeed boost technique. **[Single-source]** step teleportation.

#### 10.4.2 Bullet TTL manipulation / "Recycled Bullets"

The `timeToLive` field at `AmmoType + 0x3D8` controls how long a bullet exists before despawning. By extending this value, a bullet can be kept alive indefinitely and repositioned repeatedly.

**Theoretical technique** (mentioned #7378, #8416, #5471):
1. Modify `AmmoType + 0x3D8` (timeToLive) to a very high value (e.g., 9999.0f)
2. Fire a bullet
3. Continuously teleport the bullet to a target (e.g., a fence/gate/wall)
4. The bullet hits the structure repeatedly, dealing damage each time
5. One bullet can destroy an entire structure

**AmmoType TTL-related offsets** (#8392):
| Offset | Field | Description |
|---|---|---|
| `0x3D8` | timeToLive | How long bullet exists (seconds) |
| `0x738` | TTL * 100 | Derived timing value |
| `0x3B8` | simulationStep | Physics step interval |

**Status:** **[Unverified]** - This technique is discussed/asked about (#7378, #8416) but no working code was posted in the thread. The videos linked in #7378 are external. The concept is: extend TTL + repeated position writes = continuous damage.

**Possible issues:**
- Server may validate bullet lifetime
- Hit registration may require specific conditions
- May only work on certain structure types

If you have this working, the thread would benefit from documentation.

---

## 11. Environment: Time, Eye Accommodation, Grass, Weather

| Item | Location | Status | Posts |
|---|---|---|---|
| Eye accommodation (night brightness) | `World + 0x296C` (float) | [Corroborated] | #8339 #8393 #8463 |
| Time scale / hour | `World + 0x2970` (float) | [Corroborated] | #8405 #8416 #8463 |
| Day / DayTime | `World + 0x2974` / `0x2978` | [Corroborated] (dumpers) | #8573 |
| Grass (online) | `World + 0xC00` (write `0.0f`) | [Corroborated] | #8380 #8393 #8463 |
| Grass (offline) | `World + 0xBF0` | [Corroborated] | #8393 #8463 |
| Weather controller | `World + 0x7460` | [Corroborated] | #8463 #8501 #8573 |
| Moon intensity | `module + 0x2648E0` | [Single-source], build unclear | #8377 |
| View distance / fog / grass-renderer layouts | - | [Needs Update for 1.29+] (only in the 1.28-binary dump #8303) | #8303 |

How to find the grass offset (#8261, Feb 2026): xref the string `"Visibility set to %f"`. In the first callee look for `if (!a4) ... result = *(unsigned int*)(a1 + offset)`. That `offset` is the grass field. A signature for `SetTerrainGrid` is in S15.

How to find Landscape (#8280, 1.28): the `"GetWeather"` string sits in a function returning `*((QWORD*)qword_LANDSCAPE + 0xE8B)`.

### 11.1 Weather manipulation

The weather system is accessible via `World + WeatherController` (0x7460 for 1.29). The controller contains pointers to individual weather phenomena (#8205):

```cpp
// WeatherController = World + 0x7460  (1.29)
// Each weather type (Overcast, Fog, Rain, Snowfall) is a pointer at:
constexpr uintptr_t WeatherManagerOffset = 0x7458;  // alt offset

namespace Weather {
    constexpr uintptr_t Time = 0x8C;           // Float - current time of day
    constexpr uintptr_t OvercastPtr = 0x10;    // Pointer to overcast controller
    constexpr uintptr_t FogPtr = 0x18;         // Pointer to fog controller
    constexpr uintptr_t RainPtr = 0x20;        // Pointer to rain controller
    constexpr uintptr_t SnowfallPtr = 0x28;    // Pointer to snowfall controller
    // Inside each weather controller:
    constexpr uintptr_t ActualValue = 0x10;    // Float (0.0-1.0) - current intensity
    constexpr uintptr_t TargetValue = 0x14;    // Float (0.0-1.0) - target intensity
    constexpr uintptr_t IsForced = 0x55;       // Byte (boolean) - force weather
}
```

**Usage example** (force no rain):
```cpp
uintptr_t weatherCtrl = Read<uintptr_t>(world + 0x7460);
uintptr_t rainCtrl = Read<uintptr_t>(weatherCtrl + 0x20);
Write<float>(rainCtrl + 0x10, 0.0f);  // actual = 0
Write<float>(rainCtrl + 0x14, 0.0f);  // target = 0
Write<uint8_t>(rainCtrl + 0x55, 1);   // force
```

**[Single-source]** #8205 - needs 1.29 verification.

---

## 12. Player State, Health & Stats

### 12.1 GameVariables table - blood, shock, bleeding, temperature

The **GameVariables** (or GameVarSpace) table on each entity stores player/entity status variables like blood, health, shock, temperature, heat comfort, and **bleedingeffects**. This is readable externally via RPM and works for both local player and other networked players.

#### Historical GameVariables offsets (32-bit -> 64-bit migration)

| Build era | Offset | Posts |
|---|---|---|
| 2014 (0.28-0.46) | `0x620` -> `0x628` | #293 #296 #500 #852 |
| 2014 (0.48) | `0x660` | #1845 #1880 |
| 2015 (0.55) | `0x698` | #3560 #3610 |
| 2015 (0.57) | `0x758` | #3736 |
| 2016 (0.59) | `0x7A8` | #3900 #3960 #3963 |
| 2017 (0.62) | `0x804` | #4340 #4343 #4346 |
| **1.29+ (x64)** | **[Needs Update for 1.29+]** - likely in the `0xD00`-`0xF00` range given 64-bit pointer scaling. Search for a 2D array of name/value pairs using the iteration pattern below. | #8520 asks but gets no answer |

#### GameVariables structure

GameVariables is a **2D array** (array of arrays). Iteration pattern (32-bit legacy, adjust pointer sizes for x64):

```cpp
// Entity + GameVarsOffset = ptr to array of arrays
// Array layout: each sub-array entry is at stride 0xC (32-bit) or 0x18 (64-bit)
// Variable entry layout: name ptr at +0x4/+0x8, value ptr at +0xC/+0x10

uintptr_t gameVars = Read<uintptr_t>(entity + GAMEVARS_OFFSET);
uint32_t outerCount = Read<uint32_t>(entity + GAMEVARS_OFFSET + 0x4);  // or +0x8 on x64

for (int i = 0; i < outerCount; i++) {
    uintptr_t innerArray = Read<uintptr_t>(gameVars + i * 0xC);  // 0x18 for x64
    if (!innerArray) continue;
    uint32_t innerCount = Read<uint32_t>(gameVars + i * 0xC + 0x4);
    
    for (int k = 0; k < innerCount; k++) {
        uintptr_t namePtr = Read<uintptr_t>(innerArray + k * 0x14 + 0x4);  // 0x28 stride, +0x8 on x64
        uintptr_t valObj = Read<uintptr_t>(innerArray + k * 0x14 + 0xC);   // +0x10 on x64
        
        std::string name = ReadArmaString(namePtr);
        float value = Read<float>(valObj + 0xC);  // value at GameDataObj + 0xC
        
        // Match variable names
        if (name == "blood") { /* 0-5000 range */ }
        if (name == "health") { /* 0-5000 range */ }
        if (name == "shock") { /* 0+ range, spikes under attack */ }
        if (name == "bleedingeffects") { /* > 0 means bleeding */ }
        if (name == "temperature") { /* body temp */ }
        if (name == "heatcomfort") { /* positive = warming, negative = cooling */ }
    }
}
```

**[Corroborated]** - structure documented in #500 (buFFy), #852 (ripper121), #2566 (DaxxTrias), #2741 (geckosnipp), #3960 (geckosnipp), #4346 (geckosnipp). Working code posted for 0.28 through 0.62.

#### Known variable names in GameVariables

| Variable name | Type | Description | Posts |
|---|---|---|---|
| `blood` | float | 0 (dead) to 5000 (healthy) | #500 #852 #2566 #3960 |
| `health` | float | 0 (dead) to 5000 (healthy) | #852 #2566 #3960 |
| `shock` | float | 0 normally, spikes to 5000+ under attack | #2571 #2741 #3960 |
| `temperature` | float | body temperature | #3120 #3960 |
| `heatcomfort` | float | + warming, - cooling | #2742 #3960 |
| **`bleedingeffects`** | float | **> 0 means player is bleeding**. Higher values = more severe. | #499 #852 #4346 |
| `bleedingsources` | - | mentioned #2566, likely count of wound locations | #2566 |
| `restrainedwith` | - | reported unreliable #4346 | #4346 |

### 12.2 Bleeding detection (external)

**Bleeding IS externally readable** via the `bleedingeffects` variable in GameVariables. From #852:

```cpp
if (name.Contains("bleedingeffects") && value > 0) {
    playerInfo.bleeding = value;
}
```

**Bleeding stages:** DayZ has 4 bleeding stages (light -> severe). The `bleedingeffects` value appears to increase with severity - a value > 0 indicates active bleeding. The exact thresholds for stages 1-4 were not explicitly documented in the thread, but the principle is: `bleedingeffects > 0` = bleeding, higher = more severe.

**ESP implementation:** Display a blood-drop icon or color-code the player name (e.g., red tint) when `bleedingeffects > 0`.

### 12.3 Unconscious detection

Formula documented by DaxxTrias (#2571 #2742 #2744):

```cpp
bool isUnconscious = (shock >= blood - 550);
```

- If `shock >= blood - 550`, the player is unconscious
- Shock decays ~40-50 per second, so unconsciousness is brief unless they keep taking damage
- Using `blood - 250` gives a ~7-9 second warning before wake-up (#2744)

**Starvation unconsciousness:** If blood drops below ~500-800, the player passes out regardless of shock (#2744).

### 12.4 Other player state fields

- **Health of other players (damage system):** The thread consensus (#8367) is that authoritative health is server-side. GameVariables blood/health may reflect networked state for some entity types, but this is **[Unverified]** for 1.29+. Item **Quality** at `entity + 0x194` (0 Pristine ... 4 Ruined) is reliably readable (#8374 #8675).
- **Unconscious state (direct):** `0x3AAC` works only for local player (#8267). [Single-source, 1.28/early-1.29]
- **Stance:** `[[entity + 0x7E8] + 0x110] + 0x1C8` (int: 0 stand, 1 crouch, 2 prone, 3-5 = aiming variants) is **1.28 data** (skeleton `0x7E8`). For 1.29+ use skeleton `0x7E0` (#8285 #8289). **[Needs Update for 1.29+]**
- `Player::StatsContainer 0x6F0`, `PlayerStats::RecordValue 0x2C` (#8573 #8643). [Unverified]
- The 1.28 damage-system description (EntityType+0x120 hash table with `Blood=5000`, `Shock=100`) appears only in #8303. **[Needs Update for 1.29+]**
- **Player look direction (for ESP):** Read head bone position and forward vector from skeleton. Works even with freelook. Code in #8729 shows reading head bone (index 21 for players) and deriving look direction from bone matrix forward vector. [Single-source]

### 12.5 Finding GameVariables offset for 1.29+

To locate GameVariables on 1.29+:

1. In ReClass, attach to a running game (single-player/offline for safety)
2. Navigate to local player entity base
3. Scan the entity for a pointer that leads to a 2D array structure
4. Look for ArmaString pointers containing "blood", "health", "shock", "temperature"
5. Once found, the offset from entity base is your GameVariables offset

**Signature hint:** The variable names ("blood", "shock", etc.) are added by game scripts at runtime. Some variables only appear after joining a server (not in main menu). **[Needs Update for 1.29+]**

---

## 13. Raycasting & Visibility

### 13.1 Internal: engine raycast

Engine functions (RVAs for 1.29.162510 per #8394 / #8439, Apr-Jun 2026):

| Symbol | RVA (Apr 2026) | RVA (Jul, #8578) | RVA (Aug, #8617) |
|---|---|---|---|
| `AllocateCollisionBuffer` | `0x98450` | - | `0x97850` |
| `FreeCollisionBuffer` | `0x987E0` | `0x987E0` | `0x97BE0` |
| `Landscape_ObjectCollisionLine` (main raycast, `sub_912F40`) | `0x912F40` | - | `PhysicsRaycast 0x912840` |
| `FilterIgnoreTwo_Init` | `0x4B76E0` | `0x4B7870` | `0x4B6C80` |
| `gStatisticsScene` | `0x42648E0` | - | - |
| `Landscape::FilterIgnoreTwo` vtable | `0xC0B728` | - | - |
| `gFilterInternalObj` | `0x1008CE0` | - | - |

Physics world: `[[gStatisticsScene] + 0xB38]` (= `scene[359]`) (#8394 #8439).

Data structures as posted (#8394 #8427 #8431 #8439):

```cpp
struct CollisionBuffer {          // allocate 0x40 bytes
    CollisionInfo* _data;         // 0x00
    int            _n;            // 0x08  hit count
    uint8_t        gap[3];
    unsigned       _oldSP;        // 0x10
    void*          _buffer;       // 0x18
    bool           _memused;
    int            maxN;
};
struct Landscape__FilterIgnoreTwo {
    void* vfptr;                  // = module + FilterIgnoreTwo vtable
    void* _internalPtr;           // = FilterIgnoreTwo_Init(&gFilterInternalObj) -> local player's physics proxy
    void* _obj1;                  // physics object or NULL (NOT a raw entity base)
    void* _obj2;
};
#pragma pack(push,1)
struct PhysicsRaycastEntry {      // sizeof == 104
    int hier_level; int pad;      // 0x00
    uintptr_t obj, parent;        // 0x08, 0x10
    char pad18[16];
    uintptr_t surface;            // 0x28
    float hit_pos[3], hit_dir[3]; // 0x30, 0x3C
    char pad48[16];
    int component;                // 0x58
    char pad5C[4];
    bool is_entry, is_exit;       // 0x60, 0x61
};
#pragma pack(pop)

// Call convention used by the game's own callers (#8439):
// Landscape_ObjectCollisionLine(physWorld, 0, &buf, &filter, &from, &to,
//                               radius_bits, 1 /*mode*/, 4 /*geom*/, flags);
// flags: 0x01 stop at first hit, 0x02 nearest contact, 0x08 no AI,
//        0x10 include water, 0x20 land contact
// visible <=> no hit, or first hit distance >= |to - from| - 0.5
```

Gotchas:
- Pass the **physics proxy** from `FilterIgnoreTwo_Init`, not the entity base. The filter makes a virtual call at `+0x240` on that object, and a raw entity never matches, so your own body blocks the ray (#8439).
- Pull both endpoints in by ~0.3 m to avoid self-hits on head-to-head rays (#8431).
- Alternative: call script `DayZPhysics.RaycastRV`, which ends up in the same function (#8395).
- A related thread is linked: `dayz-sa/511150-dayz-raycasting-visible-check`, `dayz-sa/738525-raycast` (#8394 #8398).

**Status:** structures [Corroborated] (#8431 #8439 #8487 report working). RVAs are build-specific. Use the signatures in S15.3.

### 13.2 External: rebuilt physics scene

There is no engine call externally. Posters rebuilt the Bullet-physics collision data and raycast it themselves, e.g. with Intel Embree (#8494 #8496 #8518). **[Single-source]** per field:

| Object | Member | Offset |
|---|---|---|
| Physics list | count / data | `+0x0C` / `+0x18` |
| Game object -> collision shape | `obj + 0xC0` | |
| Shape -> type | `shape + 0x08` (terrain = `HEIGHTFIELD_TERRAIN_SHAPE_PROXYTYPE` = 24) | |
| Terrain height data | heights `+0x2B0`, strideX `+0x2C8`, cellSize `+0x31C` | |
| BtObject | gameobject `0xC8`, col `0x10/0x20/0x30`, x/y/z `0x40/0x44/0x48` | |
| BtCompound | count `0x84`, children `0x90` | |
| BtCompoundChild | transform `0x00`, child_shapes `0x40`, strideidx `0x60` | |
| BtConvexHull | number_points `0x84`, points `0x90` | |
| BtConvexInternal | scaling `0x10`, shape `0x20` | |
| BtTriMesh | vertices `0xB4`, vertex `0xC0`, indices `0xCC`, index `0xD8` | |

Reported side effect: entities present in the physics scene but missing from the normal tables reveal hidden objects (#8518). #8703-#8707 show an external raycast (head bone to head bone) working. Remaining issues include wrong half-extents on some objects and terrain occlusion via `EnfHeightfieldTerrainShape`.

---

## 14. Enfusion Script VM & Config Access

Internal only. From #8408 (Apr 2026, found with AI assistance and "works great"). **[Single-source]**

```cpp
// __int64 FindClassByName(ScriptModule* module, const char* name, uint8_t flags)   (sub_31F870)
#define SIG_FIND_CLASS_BY_NAME     "48 83 EC 38 45 0F B6 C8 4C 8B C2 48 8D 54 24 20 E8"
// int GetFunctionIndexByName(EnfClass* cls, const char* name)                      (sub_319D20)
#define SIG_GET_FUNC_INDEX_BY_NAME "48 89 5C 24 10 48 89 74 24 18 57 48 83 EC 20 48 8B FA 48 8B D9 48 85 C9 74"
```

| Struct | Member | Offset | Notes |
|---|---|---|---|
| EnfClass | type flags | `0x08` | `(flags & 0xF0000000) == 0x60000000` -> valid class |
| EnfClass | parent | `0x28` | `EnfClass*` |
| EnfClass | function table | `0x68` | `EnfFuncEntry**` |
| EnfClass | hash base / capacity / entries | `0x78` / `0x88` / `0x90` | entries stride 16 (8 key + 4 index + 4 pad) |
| EnfFuncEntry | type info / native address | `0x00` / `0x08` | |
| ScriptModule | class array / parent | `0x68` / `0x90` | |
| ScriptModule | hash ctrl / used / capacity / entries | `0x148` / `0x150` / `0x158` / `0x160` | ctrl byte `0x80` = empty |

Later dumps: `FindClassByName` at `0x31F960` (Jul, #8578) and `0x31ED60` (Aug, #8617). Experienced posters say these helpers are unnecessary and recommend dumping natives at runtime into a self-updating SDK (#8432). The ScriptContext ConstantTable `+0x68` exists in 1.28 dumps (#8233).

**Native registration pattern** (#8453): natives are registered as `sub_14031C300(ctx, cls, "GetObjectTexture", fn, 0)`. String xrefs on native names give you the function address. Chams-relevant natives (Jun 2026, #8489): `SetObjectMaterial 0x478C50`, `SetObjectTexture 0x478F30`, `GetObjectMaterial 0x473810`, `GetObjectTexture 0x473850` (Aug values in #8617 differ, e.g. `SetObjectMaterial 0x478180`). A vanilla material path such as `dz\gear\consumables\data\chemlight_green_on.rvmat` can be applied directly (#8453).

**Config traversal** (#8458): resolve `"CfgWorlds enoch Names"` through `ConfigGet`. The returned entry's vtable slots are `2` IsValid, `39` GetValueRawRef, `40` GetValueRef, `65` ChildCount, `66` ChildAtIndex. A child's name is at `*(entry+8) + 0x10`. Config root globals for the Aug build are in S1.2 (#8644).

**ParamClass** (1.30-exp): entries `+0x48`, count `+0x50`, parent `+0x60`. Display names resolve through a stringtable at `module + 0x4A863D8` (#8710 #8716 #8718). [1.30-exp]

**Remote script execution** is mentioned as the cause of server-side effects (spawning, moving base building) seen in the wild (#8311 #8314 #8315 #8489). No technical details are posted, and it is out of scope here.

---

## 15. Signatures

IDA-style (`?` = wildcard). "Method" describes how the posting tool extracts the value: **MovCs** = RIP-relative global, **MovReg** = 32-bit displacement, **MovRegSml / Byte** = 8-bit displacement. All are from 1.29 stable builds. Dates are given so you can judge staleness.

### 15.1 Module globals

| Target | Pattern | Date / Post |
|---|---|---|
| World | `4C 8B 05 ? ? ? ? 48 8B CF FF 50` | 14 May, #8463 |
| World | `4C 8B 05 ?? ?? ?? ?? 48 8B CF FF 50 08 8B 83 64` | 5 Jun, #8497 |
| World | `48 8B 05 ? ? ? ? 48 8D 54 24 ? 48 8B 48 30` | 21 May, #8474 |
| NetworkManager | `48 8D 0D ? ? ? ? E9 ? ? ? ? CC CC CC CC 40 53 48 83 EC ? 83 B9` | 14 May, #8463 |
| NetworkManager | `48 8D 0D ? ? ? ? E8 19 6E 01 00 48 8B 1D DA 2F 92 00 84 C0 74 20 41` (contains hard-coded rel32s, so brittle) | 21 May, #8474 |
| Tick | `4C 8B 05 ? ? ? ? 0F 57 C0 0F 57 C9` | 14 May, #8463 (#8497 same + `48 8B D0`) |
| Tick | `48 8B 05 ? ? ? ? 0F 57 C9 66 0F 6E 03` | 21 May, #8474 |
| Landscape | `48 8B 0D ? ? ? ? 48 89 44 24 ? 48 89 44 24` | 14 May, #8463 |
| Landscape (1.28) | `48 8B ? ? ? ? ? 48 89 ? ? ? 48 89 ? ? ? 4C 89 ? ? ? E8 ? ? ? ? 88 43` | 10 Mar, #8280 |
| gStatisticsScene | `48 8B 0D ? ? ? ? 33 F6 89 74 24 48 C7 44 24` | 30 Apr, #8439 (shorter form in #8394) |
| gFilterInternalObj | `48 8D 0D ? ? ? ? E8 ? ? ? ? 48 85 C0 74 ? F3 0F 10 40` | 18 Apr, #8394 / #8439 |
| FilterIgnoreTwo vtable | `48 8D 05 ? ? ? ? 48 89 44 24 ? 4C 8D 44 24 ? 48 8B 02` | 18 Apr, #8394 / #8439 |
| FOV context | `48 8B 05 ?? ?? ?? ?? 48 0F 45 05 21 75 B5 00 48` (contains a rel32, so brittle) | 5 Jun, #8497 |
| Local-player getter (1.28) | `48 83 EC 28 48 8B 0D ? ? ? ? 48 85 C9 74 11 E8 ? ? ? ?` | 17 Feb, #8253 |

### 15.2 Member-offset signatures

Fully specified patterns from #8463 (14 May 2026, 1.29.162510). The displacement is embedded in the bytes, so **these match only while the offset is unchanged** and act as offset *validators*, not finders.

```text
world::local_player       48 89 87 60 29 00 00
world::near_ent_list      48 8D 99 48 0F 00 00
world::near_table_size    39 B9 50 0F 00 00 0F 8E E6 00 00 00
world::far_ent_list       48 8B 83 90 10 00 00
world::far_table_size     83 BB 98 10 00 00 00
world::slow_ent_list      8B 87 10 20 00 00 45 8B CE
world::slow_table_size    39 93 18 20 00 00
world::bullet_table       48 8D 8B 00 0E 00 00 E8 CA 00 00 00
world::bullet_count       4C 63 B3 08 0E 00 00 0F 28 B4 24 80 00 00 00
world::weather_controller 48 8B 81 60 74 00 00 4C 8B 40 20
world::eye_accom          F3 0F 10 80 6C 29 00 00
world::time_scale         F3 0F 10 B2 70 29 00 00
world::grass_online       48 89 86 00 0C 00 00
world::grass_offline      4C 89 80 F0 0B 00 00
player::skeleton          48 8B 98 E0 07 00 00 4C 8B A0 00 02 00 00
player::input_controller  0F 10 82 E8 07 00 00
infected::skeleton        48 8D 88 70 06 00 00
camera::base              48 8D 8B B8 01 00 00 E8 32 7E FF FF
animation::animation_comp 48 89 BB 18 01 00 00 89 BB 20 01 00 00 48 89 BB 28 01 00 00
animation::matrix_array   4C 89 B1 E8 0B 00 00
animation::matrix_b       8B 43 54 8B 53 50 4C 8B 73 48
inventory::base           48 8B 81 50 06 00 00
inventory::item_quality   89 81 94 01 00 00 8B 42 08
inventory::hands          41 89 BE B0 01 00 00
inventory::cargo_grid     48 89 BB 50 01 00 00 48 89 BB 58 01 00 00 89 BB 60 01 00 00 48 8B 0C C8
inventory::slot_count_alt 48 03 83 58 01 00 00
inventory::nested_cargo   49 8D 96 48 01 00 00 41 B8 20 00 00 00 48 8D 4C 24 20 E8 5F AC 0D 00
inventory::chamber_array  48 8D 9E A8 06 00 00
network::entity_net_id    80 B8 DC 06 00 00 00
network::server_name      48 89 B7 08 03 00 00
network::game_version     89 81 50 03 00 00 89 81 A0 03 00 00
network::client_id_size   4C 8B 89 70 01 00 00 45 85 C0
network::third_person     8B 83 9C 00 00 00 4C 8B 11 C7 44 24 50 FF FF FF FF
network::crosshair        48 89 B7 A0 00 00 00 48 8D 4F 68
network::player_name      48 89 BB F8 00 00 00 89 BB 00 01 00 00
common::type              89 81 80 01 00 00 48 C7 81 68 01 00 00 00 00 80 3F
common::visual_state      48 8B 87 C8 01 00 00 48 83 C0 2C F3 0F 10 00 F3 0F 10 48 04 F3 0F 10 50 08 F3 0F 11 43 28
entity_type::clean_name   48 81 C1 18 05 00 00 E8 8F FD FF FF
entity_type::type_name    80 B9 D0 00 00 00 00 48 8B D9
ammo::hit                 48 89 9F 70 03 00 00
ammo::indirect_hit_range  49 8B 86 78 03 00 00 48 3B 05 2D 81 EB 00
ammo::init_speed          41 8B 82 8C 03 00 00
ammo::caliber             48 89 9F C0 03 00 00
ammo::dispersion          8B 88 CC 03 00 00 41 3B CC 44 0F 4C E1 44 89 64 24 44 4C 8D A3 90 00 00 00 8B 4C 24 44 83 F9 01 0F 8F 24 01 00 00
ammo::air_friction        41 89 87 DC 03 00 00
ammo::fuse_distance       66 C7 83 AC 03 00 00 FF 00
ammo::magazine_ammo_count 39 9F AC 06 00 00
ammo::magazine_capacity_a 49 8D 8F B0 06 00 00
ammo::magazine_capacity_b 44 3B 82 B4 06 00 00
camera::projection_d1     48 8B B7 D0 00 00 00 48 85 F6
camera::projection_d2     8B 87 DC 00 00 00 8B 5B 1C
```

Wildcarded member-offset finders, which survive offset changes: #8474 (21 May, ~35 patterns, `\x..` + mask form) and #8497 (5 Jun, **106 patterns** with extraction method). Both are reproduced verbatim in the consolidated dataset (`#post-8474`, `#post-8497`). They are too long to duplicate here. **Status:** [Single-source] each. #8517 says the #8497 set keeps the World pointer resolving across patches.

### 15.3 Function signatures

| Function | Pattern | Resolve | Post |
|---|---|---|---|
| `Landscape_ObjectCollisionLine` (prologue) | `48 8B C4 48 89 58 08 48 89 70 18 48 89 78 20 55 41 54 41 55 41 56 41 57 48 8D 68 D1` | direct | #8439 |
| `Landscape_ObjectCollisionLine` (call site) | `E8 ? ? ? ? 8B 84 24 ? ? ? ? 0F 57 FF` | E8 rel32 | #8394 |
| `AllocateCollisionBuffer` | `E8 ? ? ? ? 48 8B 0D ? ? ? ? 48 8D 05 ? ? ? ? 48 89 85` | E8 rel32 | #8394 #8439 |
| `FreeCollisionBuffer` | `E8 ? ? ? ? 0F 28 BC 24 ? ? ? ? 0F 28 C6` | E8 rel32 | #8394 #8439 |
| `FilterIgnoreTwo_Init` | `E8 ? ? ? ? 48 8B C8 E8 ? ? ? ? 48 85 C0 0F 84 ? ? ? ? 0F B6 94 24` | E8 rel32 | #8394 #8439 |
| `FindClassByName` | `48 83 EC 38 45 0F B6 C8 4C 8B C2 48 8D 54 24 20 E8` | direct | #8408 |
| `GetFunctionIndexByName` | `48 89 5C 24 10 48 89 74 24 18 57 48 83 EC 20 48 8B FA 48 8B D9 48 85 C9 74` | direct | #8408 |
| `SetTerrainGrid` (grass) | `8B 81 ? ? ? ? ? ? ? 77 ? F3 0F 10 89` | - | #8261 (Feb 2026; build unclear) |
| FOV state / ptrs (1.28 era) | see #8259 (`83 3D ? ? ? ? ? 74 ? 48 8B 05`, ...) | - | #8259 [Needs Update for 1.29+] |

---

## 16. Function RVAs

Function RVAs move with every patch. These are listed so you can rebuild your own IDB, not to be hard-coded.

| Function | Jun 2026 (#8489) | 14 Jul 2026 (#8573) | 15 Jul (#8578) | Aug 2026 (#8617) |
|---|---|---|---|---|
| SetObjectMaterial | `0x478C50` | - | - | `0x478180` |
| SetObjectTexture | `0x478F30` | - | - | `0x4784D4` |
| GetObjectMaterial | `0x473810` | - | `0x473930` | `0x472D40` |
| GetObjectTexture | `0x473850` | - | `0x4739E0` | `0x472DF0` |
| FindClassByName | - | - | `0x31F960` | `0x31ED60` |
| FilterIgnoreTwo_Init | - | - | `0x4B7870` | `0x4B6C80` |
| AllocateCollisionBuffer | - | `0x98450` | - | `0x97850` |
| FreeCollisionBuffer | - | - | `0x987E0` | `0x97BE0` |
| DamageSystem_GetHealth | - | `0x429C10` | `0x429D60` | `0x429160` |
| DayZPlayer_GetName | - | `0x4E4FA0` | - | `0x4E4540` |
| PhysicsRaycast | - | - | - | `0x912840` |

The full automated function lists (250+ entries) are in the dataset (`#post-8573`, `#post-8578`, `#post-8617`). Their names are dumper-assigned labels (e.g. `HealthCalc_7`, `RegionA_Func1`), not engine symbols. **[Unverified]**

1.28-era RVAs (World getter `0x5659C0`, W2S `0x578D10`, etc.) from #8303 are **[Needs Update for 1.29+]**.

---

## 17. 1.30 Experimental Preview

> **[1.30-exp]** Experimental builds change often. Two sources disagree even on the World RVA.

- #8698 (glennf, 17 Sep) is an automated dump with World `0x4262FE8`. #8699 (MZivert) says that is stale: "**New world is 0x18228B8** and local (CameraOn) is wrong".
- #8710 (oggobogg, 22 Sep) is a hand-curated list:

```text
World                         0x18228B8
NetworkManager (object)       0x4A7C1B0
World::Camera                 0x1B0
Near / size                   0xF70 / 0xF78
Far / size                    0x10B8 / 0x10C0
Slow                          0x2010 / 0x2018 / 0x2020
Item                          0x2218 / 0x2220
Entity::Type                  0x130
Entity::VisualState           0x158
Entity::Inventory             0x5D8
Entity::NetworkId             0x664   (NetworkIdExternal 0x6DC)
Entity::IsDead                0x156 bit 2
Entity::Quality               0x154
Entity::HierarchyParent       0x345
HumanType::ObjectName         0x70
HumanType::Realclassname      0x78
HumanType::ClassName          0x98
HumanType::ModelName          0xB0
HumanType::Config             0xB8   (ParamClass -> display name, see S14)
HumanType::CategoryName       0xD0
HumanType::CleanName          0x518
Stringtable                   0x4A863D8
ArmaString length / chars     +0x8 (uint32) / +0x10
Inventory::Hands / HandValid  0x190 / 0x1AC
Inventory::Cargo              0x128
Inventory::Equipped           0x130 / 0x13C
PlayerSkeleton                0xB98 / 0x21C0
InfectedSkeleton              0x620
AnimGraph                     0x188 / 0x190
MatrixArray                   0x38
MatrixBoneBase / BoneStride   0x24 / 0x30
Remap                         *(graph+0x20)+0xE0
NM::Client                    +0x50 / +0x48
NetworkClient Scoreboard/Size +0x18 / +0x24   (Scoreboard::Count +0x0C)
Identity NetID/Name/PlainName/SteamID  +0x30 / +0xF8 / +0x100 / +0xA0
Camera                        unchanged layout (Right 0x8 ... Proj 0xD0/0xDC)
VisualState                   transform +0x8, pos +0x2C
```

- In 1.30-exp, display names should come from `GetDisplayName` through the `ParamClass` and stringtable instead of `type+0x518` (#8716 #8718).
- Swapping 1.29 offsets for #8710's without changing the read chains broke names and items (#8717). The **chains changed**, not just the numbers.

---

## 18. Conflict Register & Open Questions

| # | Topic | Positions | Posts | Suggested resolution |
|---|---|---|---|---|
| C1 | World -> LocalPlayer | `0x2960` vs `0x2958` | S4.3 | Disassemble the `- 0xA8` getter on your build (#8620 shows the pattern). |
| C2 | Near vs Far naming | `0xF48`=Near (majority) vs `0xF48`=Far | S4.1 | Doesn't matter if you read both. |
| C3 | Item-table count field | `+0x8` vs `+0x10` | S4.2 | Iterate `+0x8` (capacity) slots and check slot state = 1. |
| C4 | Skeleton -> AnimClass | `0x118` vs `0x110` vs `0x90` | S7.1 | `0x118` has the most independent confirmations. |
| C5 | Bone stride / base | `0x30`/`+0x54` vs `0x60` vs `0x30`/`+0x24` | S7.1 | Base/index convention. Pick one with the matching index set. |
| C6 | Player bone index sets | Set A vs Set B (off by one) vs #8557 | S7.3 | Match to C5. |
| C7 | Infected bone indices | 4 incompatible tables | S7.4 | Render all indices and map by eye. |
| C8 | AmmoType airFriction / dispersion / coefGravity | `0x3DC/0x3CC/0x3E4` (#8392) vs `0x3B4/0x3A4/0x3BC` | S10.3 | Check against a known ammo's config values (e.g. compare initSpeed to `config.cpp`). |
| C9 | Magazine count | `0x6AC/0x6A8` (uint16) vs `0x3B0/0x3A4` | S10.2 | `0x6AC` is hand-verified (#8738). |
| C10 | Item max quantity | `0x858` vs `0x850` | S10.2 | Unresolved. |
| C11 | CargoGrid count / equipped count | `0x44` vs `0x40`; `0x15C` vs `0x158` | S9 | `0x44` / `0x15C` have the most posts. |
| C12 | NetworkClient size field | `0x24` vs `0x170` | S8 | Use `0x24`. #8377 says `0x170` is wrong for this purpose. |
| C13 | Third-person flag location | `client+0x9C` vs `client+0x28->+0x74` | S8 | Both reported. The `0x9C` path has a signature (#8463). |
| C14 | EntityType ObjectName | `0x70` vs `0x6C` vs `0x98` | S5.3 | Unresolved. Use TypeName/CleanName instead. |
| C15 | View-angle aimbot externally | impossible vs simple writes | S3 | Unresolved. Input-controller layout is 1.28-only. |
| C16 | Camera FOV/aspect | `0x70` vs `0x6C` | S6.1 | #8420 and #8421 converge on aspect `0x6C`. |

**Open questions the thread never answers:**
- A full fix for ESP alignment while scoped (S6.2).
- 1.29 layouts for the input controller (aim override), recoil and sway. #8418 `0x1F4E` is disputed in #8425, and #8446 posts a code-patch byte sequence. [Unverified]
- Player name chain on the Xbox/Microsoft PC build.
- What `Weapon + 0x6B0` holds in 1.29 (#8687).
- Head-position member on VisualState (#8655 #8656).

---

## 19. Source Index

Key contributors, with the posts used most in this wiki:

| Area | Primary posts |
|---|---|
| 1.29 exp module RVAs & first struct deltas | #8268, #8298, #8311 (MZivert), #8329 |
| 1.29 stable full config (Steam 1.29.162510) | #8393 (dullxzixed) |
| AmmoType layout | #8392 (KingJulianz) |
| Skeleton chain & rigs | #8396 (hejmeddig123xd), #8401, #8557, #8615, #8654, #8675 |
| Raycast (internal) | #8394, #8427, #8431, #8439, #8487 |
| Script VM | #8408, #8453, #8458 |
| Signatures | #8463 (dullxzixed), #8474, #8497 (glennf) |
| Network / names | #8587, #8624, #8722 |
| Inventory & items | #8630, #8650, #8658, #8736, #8738, #8741 (leonardgilmore) |
| Automated dumps (globals per patch) | #8485, #8501, #8573, #8576-#8578, #8617, #8643 |
| Freecam write-ups | #8470, #8611 |
| External physics / vis-check | #8494, #8496, #8518, #8703-#8707 |
| Bullet manipulation / magic bullet | #8127, #8128, #8491, #8684, #8687, #8692, #8726, #8734 (leonardgilmore, yLevitate, FPKCeifador, sdasfewewgw, needer) |
| No recoil / input controller | #8418, #8421, #8446, #8720 (dullxzixed, fiercjay) |
| Aimbot / prediction | #8498, #8504, #8535 (alexanderyy, DuOreng, glennf) |
| Vehicles | #8093, #8099, #8100 (waywardDonkey, Vituxa01) |
| GameVariables (blood/shock/bleeding) | #500, #852, #2566, #2741, #3960, #4346 (historical, pre-1.29) |
| 1.30 experimental | #8698, #8699, #8710, #8716, #8718 |

Every post above is reproduced in full, under the anchor `#post-NNNN`, in `DayZ_1.29_Consolidated_Dataset.md`.

---

## 20. Vehicles

> **Note:** Vehicle physics are now **server-authoritative** (see [Bohemia's writeup](https://reforger.armaplatform.com/news/server-authoritative-vehicles)). Reading data externally is possible; writing to manipulate vehicle behavior may have limited or no effect.

### 20.1 Vehicle member map

From #8093 (waywardDonkey, Oct 2025) - **[Single-source, pre-1.29]** - structure may need offset adjustment for 1.29+:

| Field | Offset | Type | Description |
|---|---|---|---|
| WheelCount | `0xAD4` | int32 | Number of wheels (e.g., 4) |
| SteeringAngle | `0xE7C` | float | Current steering wheel angle |
| GearboxPtr | `0x21C0` | ptr | Gearbox structure pointer |
| CrewPtr | `0x1C90` | ptr | Passengers/crew structure |
| FluidPtr | `0x20D8` | ptr | Fuel/oil/brake/coolant levels |
| WheelManagement | `0xAE0` | ptr | Wheel physics array |
| AttachmentSlots | `0x6A8` | ptr | Vehicle part attachments |

**Gearbox structure** (`[Entity + 0x21C0]`):

| Field | Offset | Type | Description |
|---|---|---|---|
| CurrentGear | `+0x38` | int32 | 0 = reverse, 1 = neutral, 2+ = forward gears |
| MaxGears | `+0x3C` | int32 | Total gear count |

**Crew structure** (`[Entity + 0x1C90]`):

| Field | Offset | Type | Description |
|---|---|---|---|
| PassengerList | `+0x10` | ptr | Array of passenger entity pointers (stride 0x10) |
| OccupiedSeats | `+0x18` | int32 | Current passenger count |
| MaxSeats | `+0x20` | int32 | Total seat capacity |

### 20.2 Wheel physics

**Wheel management** (`[Entity + 0xAE0]`):

```
WheelManagement+0x00 = Wheel0Physics* (Front-Left)
WheelManagement+0x08 = Wheel1Physics* (Front-Right)
WheelManagement+0x10 = Wheel2Physics* (Rear-Left)
WheelManagement+0x18 = Wheel3Physics* (Rear-Right)
```

**Per-wheel physics structure:**

| Field | Offset | Type | Description | Range |
|---|---|---|---|---|
| AngularVelocity | `+0x88` | float | Rotation speed (rad/s) | + = forward, - = reverse |
| SuspensionLoad | `+0x94` | float | Ground contact / compression | 0.0 = airborne, 0.4-0.45 = normal |
| SlipRatio | `+0x98` | float | Wheel slip (traction loss) | 0.0 = full grip, 1.0 = spinning/locked |

### 20.3 Vehicle fluids & parts

**Fluid system** (`[Entity + 0x20D8]`):

| Field | Offset | Type | Description |
|---|---|---|---|
| FuelLevel | `+0x18` | float | 0.0-1.0 (empty to full) |
| OilLevel | `+0x1C` | float | 0.0-1.0 (unused in-game) |
| BrakeFluid | `+0x20` | float | 0.0-1.0 (unused in-game) |
| CoolantLevel | `+0x24` | float | 0.0-1.0 |

**Direct entity offsets** (from #8099, Vituxa01):

| Field | Offset | Type | Description |
|---|---|---|---|
| FuelLiters | `0x257C` | float | Actual fuel in liters (e.g., Gunter = 55L max) |
| Battery | `0x25A4` | float | -1.0 = missing, 0.0-1.0 = charge state |
| SparkPlug | `0x25A8` | float | -1.0 = missing, 0.0-1.0 = condition |
| Radiator | `0x259C` | float | 0.0 = missing, 1.0 = pristine |

**[Single-source, pre-1.29]** - offsets from October/November 2025 (#8093, #8099, #8100). Likely need re-verification for 1.29 stable, as struct layouts may have shifted slightly.

---

## 21. Complete Feature Catalog

This section catalogs all features discussed in the thread with their technical requirements and implementation details.

### 21.1 External features

Features achievable with external memory read/write only (DMA, kernel driver, hypervisor).

#### ESP / Visuals

| Feature | How it works | Key offsets/techniques | Status |
|---|---|---|---|
| **Player ESP (boxes, lines, names)** | Read entity tables, get positions via VisualState, project to screen | World->NearEnt/FarEnt->Entity->VisualState+0x2C, W2S math (S6.2) | [Corroborated] |
| **Player names** | Read scoreboard, match NetworkID to entity | NetworkManager->Scoreboard->PlayerIdentity->Name (S8) | [Corroborated] |
| **Skeleton / bone ESP** | Read bone matrices, project each bone | Entity->Skeleton->AnimClass->MatrixArray (S7) | [Corroborated] |
| **Item / loot ESP** | Read item table, filter by name/type | World->ItemTable, EntityType->CleanName (S5.3) | [Corroborated] |
| **Zombie / infected ESP** | Same as player ESP, different bone indices | Entity tables + infected skeleton (S7.4) | [Corroborated] |
| **Vehicle ESP** | Read entity tables, filter by type "car" | EntityType->ConfigName == "car" (S20) | [Corroborated] |
| **Dead body ESP** | SlowTable entities with IsDead=true | Entity+0xE2 (bit check), filter SlowTable (S4.2) | [Corroborated] |
| **Distance display** | Calculate 3D distance from camera | `sqrt((x2-x1)^2 + (y2-y1)^2 + (z2-z1)^2)` | [Corroborated] |
| **Health bars (via Quality)** | Read item quality as proxy | Entity+0x194 (0=Pristine to 4=Ruined) | [Corroborated] |
| **Weapon in hands display** | Read inventory hands slot, get name | Entity->0x650->0x1B0->Type->CleanName (#8410) | [Corroborated] |
| **Bleeding indicator** | Read GameVariables `bleedingeffects` | GameVariables table, find "bleedingeffects" > 0 (S12.2) | [Corroborated] (historical) |
| **Unconscious indicator** | Calculate shock >= blood - 550 | GameVariables "shock" and "blood" values (S12.3) | [Corroborated] (historical) |
| **Blood/Health/Shock values** | Read GameVariables table | Entity->GameVariables->iterate for "blood"/"health"/"shock" (S12.1) | [Needs Update for 1.29+] |
| **Vehicle info (fuel, passengers)** | Read vehicle structure | Entity->FluidPtr, CrewPtr (S20) | [Single-source] |
| **Admin detection** | Scan inventory for admin/invisible clothing items | Read CargoGrid, check for mod-specific admin item names (#8473) | [Single-source] |

#### Combat / Aimbot

| Feature | How it works | Key offsets/techniques | Status |
|---|---|---|---|
| **Mouse aimbot (legit)** | Calculate angle to target, send mouse input via OS | `SendInput()` / `mouse_event()`, angle math | [Corroborated] #8498 |
| **Aim prediction** | Lead target by velocity * (distance / bullet_speed) | Read target velocity (VisualState+0x54), weapon InitSpeed | [Single-source] #8535 |
| **Silent aim / magic bullet** | Write bullet position to target head | BulletTable->Entity->VisualState+0x2C (S10.4) | [Corroborated] |
| **50m bypass (InitSpeed boost)** | Boost bullet speed so it reaches target faster | AmmoType+0x38C = distance * 100 (S10.4.1) | [Corroborated] |
| **No spread (write)** | Write dispersion to 0 | AmmoType+0x3CC = 0.0f (#8388 #8393) | [Single-source] |
| **Instant hit** | Combine InitSpeed boost + position write | See S10.4.1 full code | [Corroborated] |
| **Bullet TTL / recycled bullets** | Extend timeToLive, repeatedly teleport bullet to structure | AmmoType+0x3D8 (TTL), continuous position writes (S10.4.2) | [Unverified] |

**Magic bullet full technique:**
1. Fire weapon toward target (need rough line of sight)
2. On each bullet in BulletTable:
   - Boost InitSpeed: `AmmoType2 + 0x38C = distance * 100`
   - Write position: `VisualState + 0x2C = targetHead`
3. Server accepts because position delta <50m and bullet reached there "legitimately"

#### Environment Manipulation

| Feature | How it works | Key offsets/techniques | Status |
|---|---|---|---|
| **No grass** | Write 0.0f to grass scalar | World+0xC00 (online) or World+0xBF0 (offline) | [Corroborated] |
| **Night vision / fullbright** | Write eye accommodation to high value | World+0x296C = 1.0f (S11) | [Corroborated] |
| **Time scale** | Write time scale | World+0x2970 (S11) | [Corroborated] |
| **Third-person force** | Write third-person flag | NetworkClient+0x9C (S8) | [Corroborated] |
| **Crosshair force** | Write crosshair flag | NetworkClient+0xA0 (S8) | [Corroborated] |
| **FOV change** | Write camera FOV | Camera+0x70 (aspect 0x6C) | [Corroborated] |
| **Speed hack (tick manipulation)** | Write game tick scalar | module+TICK_RVA (S1.2) | [Corroborated] - likely detected |
| **Weather manipulation** | Write to weather controller pointers | WeatherCtrl->Rain/Fog/Overcast+0x10/0x14/0x55 (S11.1) | [Single-source] |

#### Freecam

| Feature | How it works | Key offsets/techniques | Status |
|---|---|---|---|
| **Freecam (data-only)** | Write camera mode + position + orientation | CameraMode global, camera transform matrices (#8611) | [Corroborated] |
| **Freecam (vtable swap)** | Swap camera object vtable to debug camera | vtable manipulation (#8470) | [Single-source] |

#### Player State Reading

| Feature | How it works | Key offsets/techniques | Status |
|---|---|---|---|
| **Own blood/health/shock** | Read GameVariables on local player | S12.1 - iterate GameVariables for variable names | [Needs Update for 1.29+] |
| **Other player bleeding** | Read `bleedingeffects` from their GameVariables | S12.2 | [Needs Update for 1.29+] |
| **Unconscious detection** | Calculate shock >= blood - 550 | S12.3 | [Needs Update for 1.29+] |
| **Stance (local only)** | Read stance from skeleton chain | [[Entity+0x7E0]+0x110]+0x1C8 | [Needs Update for 1.29+] |

### 21.2 Internal-only features

Features requiring code execution inside `DayZ_x64.exe`.

| Feature | How it works | Key techniques | Status |
|---|---|---|---|
| **Chams / glow ESP** | Call `SetObjectMaterial` on entities | Script VM call, material swap (#8453 #8489) | [Single-source] |
| **Engine raycast / visibility check** | Call `Landscape_ObjectCollisionLine` | Function at ~0x912F40, physics world access (S13.1) | [Corroborated] |
| **No recoil (code patch)** | Patch recoil calculation code | Patch at 0x7B56E7: `0F 84 ED 00 00 00` -> `E9 EE 00 00 00 90` (#8446) | [Single-source] |
| **No sway** | Zero sway values or patch calculation | Near recoil in InputController (#8421 #8422) | [Unverified] |
| **Script execution** | Call `FindClassByName`, invoke script functions | EnfClass resolution, function index lookup (#8408 #8428) | [Corroborated] |
| **Config access** | Read game config values | ConfigRoot globals (S14) | [Single-source] |
| **Vector aimbot (aim override)** | Write aim direction directly | InputController aim override fields (#8504) | [Conflicting] |
| **Item spawning** | Call CreateInInventory or similar | Script VM calls (#8239) | [Single-source] |

**No recoil code patch detail** (#8446):
```
Address: module + 0x7B56E7
Original: 0F 84 ED 00 00 00  (je +0xED)
Patched:  E9 EE 00 00 00 90  (jmp +0xEE, nop)
```

**InputController structure** (1.28 data, needs 1.29 verification):
- Entity + 0x7E8 = InputController*
- Recoil fields reported around 0x1F4E area (disputed #8418 #8425)
- Sway fields reported "right above recoil" (#8423)

### 21.3 Server-side / impossible features

Features that cannot work because the server has authority.

| Feature | Why impossible | Notes |
|---|---|---|
| **Teleport self** | Server validates position, rubberbands you back | Writing position locally does nothing |
| **Teleport other players** | Their position is on their client | Server-side only |
| **Give yourself items** | Inventory is server-authoritative | Writing inventory locally doesn't sync |
| **Infinite health** | Health is server-side | GameVariables health is read-only |
| **Damage modification** | Hit registration is server-side | Can't increase damage dealt |
| **Kill other players directly** | Damage calculated server-side | Must use projectiles |
| **Vehicle physics hacks** | Vehicle physics are server-authoritative | Can read but not manipulate (#8093) |
| **Dupe items** | Server tracks inventory | Historical exploits patched |
| **Other player stance** | Not networked to clients | #8285 #8289 |

### 21.4 Feature implementation checklist

For each feature, verify these components:

**ESP features:**
- [ ] World pointer valid
- [ ] Entity table iteration working
- [ ] VisualState position reading
- [ ] World-to-screen projection
- [ ] Entity filtering (type name, IsDead, distance)
- [ ] Name resolution (scoreboard or CleanName)

**Combat features:**
- [ ] BulletTable access
- [ ] Bullet VisualState writing
- [ ] AmmoType chain for InitSpeed
- [ ] Target selection (closest, FOV, visible)
- [ ] Bone position for headshots

**Magic bullet specifically:**
- [ ] BulletTable at World+0xE00
- [ ] Bullet->VisualState+0x1C8
- [ ] Position at VisualState+0x2C
- [ ] Inventory->Hands->AmmoType chain
- [ ] InitSpeed at AmmoType+0x38C
- [ ] High-frequency write loop
- [ ] Distance calculation for speed boost

**GameVariables (bleeding/health):**
- [ ] Find GameVariables offset for 1.29 (scan for 2D array with "blood" string)
- [ ] Outer array iteration (stride 0xC on 32-bit, likely 0x18 on 64-bit)
- [ ] Inner array iteration (stride 0x14 on 32-bit, likely 0x28 on 64-bit)
- [ ] ArmaString name comparison
- [ ] Float value at GameDataObj+0xC

### 21.5 Detection considerations

Features likely to be detected (from thread discussion):

| Feature | Detection risk | Notes |
|---|---|---|
| Speed hack / tick manipulation | **HIGH** | Easily detected server-side |
| Code patches (no recoil) | **HIGH** | BattlEye integrity checks |
| Script execution | **HIGH** | BE monitors script calls |
| Item spawning | **HIGH** | Server logs impossible inventory |
| Magic bullet (aggressive) | **MEDIUM** | Massive blood splashes, suspicious hit patterns |
| Freecam vtable swap | **MEDIUM** | Memory integrity |
| Freecam data-only | **LOW** | No code modification (#8611) |
| ESP (read-only) | **LOW** | Pure reads, hard to detect |
| External memory reads | **LOW** | DMA/hypervisor adds hardware layer |

**Thread recommendations:**
- Prefer data writes over code patches
- External/DMA is harder to detect than internal
- Read-only features have lowest risk
- Signature scan your offsets rather than hardcode (#8517)
