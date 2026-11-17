# Last Used – Architecture

> Based on the concept of Last Use Tooltip by perusi. Independently developed for Minecraft 26.3 Fabric and NeoForge.
> See [CREDITS.md](CREDITS.md).

## 1. Overview

```
core/                      Minecraft-free Java 21: time formatting, tracking rules, config parsing (unit tests)
api/                       tiny public API jar (component id, API version)
platform/mc26/common/      Minecraft 26.3 code shared by both loaders (official Mojang names, no loader API)
  src/main/java              component, tracker, item groups, mixins, network payload, GameTests, boot/persist probes
  src/client/java            tooltip line, server-info state, client boot check + self tests
platform/mc26/fabric/      Fabric entry points, registration, events, durability mixin
platform/mc26/neoforge/    NeoForge entry points, registration, events, durability mixin
tools/                     mod-toolkit checks (cellcheck, bootcheck, releasecheck, ...), gen_assets.py
```

Each loader jar is compiled from `core` + `api` + `common` + its loader module; nothing is shaded, there is no
jar-in-jar. Minecraft 26.x is unobfuscated, so the shared mixins use the official names on both loaders.

| | Version (26.3 pins, `platform/mc26/versions/26.3.properties`) |
|---|---|
| Java | 25 (toolchain), core/api compiled for Java 21 |
| Gradle | 9.8.0 (wrapper) |
| Fabric Loom | 1.19.4 (non-remapping `net.fabricmc.fabric-loom`) |
| Fabric Loader / API | 0.20.1 / 0.166.2+26.3 |
| NeoForge / NeoForm | 26.3.0.109 / 26.3-1, ModDevGradle 2.0.148 |

## 2. Data model

**One item data component, `lastused:last_used`** (`LastUsedComponents.LAST_USED`): a `Long` with the time of the
last use in **Unix epoch milliseconds of the server clock**.

- *Persistent* (`Codec.LONG`): saved with the item wherever vanilla saves items – chunks/block entities (chests,
  shulker boxes, hoppers, item frames), player data, ender chests, dropped item entities. Restarts of server or
  client therefore keep the value (checked by the restart probe, section 8).
- *Network synchronised* (`VAR_LONG`): the client gets it with the normal item sync; no extra packets per use.
- *`ignoreSwapAnimation`*: vanilla 26.3 replays the "re-equip" animation of the held item when one of its
  components changes, unless the component type opts out (vanilla does this for `minecraft:damage`). Without the
  flag every use would make the item bob in the hand.
- **No component = never used** (since Last Used tracks it). There is no separate "never" value.
- Real time instead of game ticks: ticks stop when the server stops, differ per world and jump with `/time`;
  epoch milliseconds survive restarts and mean the same on every server.

Why a dedicated component and not `minecraft:custom_data`: data packs often test `custom_data` with exact
matches (`[custom_data={...}]`); adding a key there would silently break those checks. A dedicated component is
typed, cheap and invisible to such predicates. Other mods can read it by id (`LastUsedApi.COMPONENT_ID`).

## 3. When does an item count as "used"?

The timestamp is written **only on the logical server**, only for a **server-side player** (`ServerPlayer`, which
includes fake players of machines) who is **not a spectator**, and only **after the game accepted the action**.
Hooks (`common/.../mixin`, loader `ItemStackDurabilityMixin`):

| Hook | Counts when | Covers |
|---|---|---|
| `ItemStack#use` | result is `Success` *and* `wasItemInteraction()` | right-click in the air: bows/crossbows/tridents/spears (start), shields (raise), food/potions (start), ender pearls, fishing rods, goat horns, spyglass, ... |
| `ItemStack#useOn` | same | right-click on a block with the item: placing blocks, tilling, stripping, path making, flint and steel, bone meal, buckets, ... |
| `BlockStateBase#useItemOn` | same | block-specific item interactions: bucket/bottle on a cauldron, shears on a pumpkin or bee nest, disc into a jukebox, item into a composter or lectern, ... |
| `ItemStack#interactLivingEntity` | same | item on a mob: name tag, dye, saddle, bucket on an axolotl, ... |
| `ItemStack#releaseUsing` | the item's release logic reports that it did something | releasing a drawn bow (shot), throwing a trident, finishing a crossbow charge |
| `ItemStack#finishUsingItem` | always (the stack is only marked if something of it is left) | finishing eating/drinking |
| `ItemStack#hurtEnemy` | called by vanilla only after the hit hurt the target | melee hits with any held item |
| `ItemStack#mineBlock` (at the `ITEM_USED` statistic) | the item's mining logic accepted the block | breaking blocks in survival with tools, swords, shears |
| `ServerPlayerGameMode#destroyBlock` | creative player broke a block with an item that has a `tool` component | creative mining (vanilla skips `mineBlock` there) |
| `ItemStack#hurtAndBreak` (Fabric: `ServerPlayer` overload; NeoForge: `LivingEntity` overload) | a player's damageable item is asked to lose durability (`amount > 0`) | shield blocks, armor absorbing hits, elytra flight, shears on sheep, fishing rods, flint and steel, ... |

`wasItemInteraction` is the vanilla flag that separates "the item did something" from "the block/entity did
something" – it is what vanilla uses for the `ITEM_USED_ON_BLOCK` advancement trigger.

**What does not count**

- Failed or cancelled attempts: a full player trying to eat (`use` returns `Fail`), a bow without arrows, an item
  on cooldown, an action cancelled by another mod's event (NeoForge `RightClickItem`/`UseItemOnBlockEvent`,
  Fabric `UseItemCallback`/`UseBlockCallback` returning a result before the item runs). In 26.3 `Fail` also
  "consumes" the action (`consumesAction() == true`), so the hooks test for `Success` explicitly.
- Opening a chest, door or crafting table while holding something (`useWithoutItem`), trading with a villager.
- A left click on a block that does not break it, and swinging at air.
- Breaking a block with a non-mining item (e.g. a book) in survival.
- Spectators, mobs, dispensers (no player), and the client (prediction calls of the same methods are ignored).

**Documented behaviour per item kind** (default rules; "tracked" = gets and shows a timestamp, section 4)

| Kind | Tracked by default | Updated by |
|---|---|---|
| Tools (pickaxe, axe, shovel, hoe, shears, flint and steel, fishing rod, brush) | yes | breaking blocks, using them on blocks/mobs, any durability use |
| Weapons (sword, mace, spear) | yes | hits that hurt a mob, block breaking (swords), spear use |
| Bows, crossbows, tridents | yes | start of drawing, the shot/throw (release), durability use |
| Shields | yes | raising the shield, every blocked hit (durability) |
| Armor, elytra | yes | durability loss: absorbed hits, gliding |
| Potions, milk, stews (unstackable consumables) | yes | start of drinking/eating; afterwards the item is gone (the bottle/bucket/bowl is a new item without timestamp) |
| Food, blocks, arrows and other stackable items | **no** (optional, see 4) | – / with `tracking.stackable_items=true`: eating, placing, throwing |
| Placeable unstackable blocks (beds, shulker boxes with contents, ...) | yes | placing marks the stack, which is then consumed; a shulker box keeps the timestamp of its own item |
| Empty hand | – | nothing is stored (there is no item) |
| Items without an active use (e.g. a totem, a music disc in the inventory) | yes | only the interactions above (a disc inserted into a jukebox counts, the totem has no tracked action) |

## 4. Item stacks and identity

A component belongs to the **stack**, not to individual items in it. Vanilla only merges two stacks if all
components are equal (`ItemStack.isSameItemSameComponents`). Consequences, verified by GameTests:

- **Splitting** copies the timestamp to both halves; halves with the same value merge again.
- **Different timestamps never merge**, and a stamped stack never merges with an unstamped one. This is a hard
  vanilla rule; working around it would mean patching every merge site (inventories, hoppers, item entities,
  bundles, creative tabs, network hashing) and changing what equality means for all other mods.
- Therefore **stackable items are not tracked by default** (`tracking.stackable_items=false`): placing one
  cobblestone from a stack of 64 would otherwise turn the stack into a "different" item that no longer merges
  with cobblestone you pick up – inventories would fragment with every use. Unstackable items (max stack size 1,
  i.e. tools, weapons, armor, shields, bows, potions, ...) are exactly the ones whose "last used" matters and
  they cannot fragment.
- With `tracking.stackable_items=true` the server tracks them anyway, with the vanilla consequence above
  (documented in the config file). A "newest wins" merge is impossible without changing vanilla equality.
- **Hoppers, chests, dropping, picking up, death drops, giving to another player** (GameTests) and shulker boxes,
  bundles, item frames (same vanilla item storage): the stack is moved unchanged, so is the timestamp.
- **Crafting** creates a new item → no timestamp. **Anvil, smithing table, grindstone, enchanting table** work on a
  copy of the main input in vanilla, so the result keeps that input's timestamp (anvil: the left item; the right
  item's timestamp is dropped like its other data). This follows from vanilla's code and is not covered by an
  automated test (manual test matrix).
- **Updates are throttled**: the same stack is rewritten at most once per `tracking.update_interval_seconds`
  (default 1 s). Continuous use (mining, gliding, a long fight) changes the item at most once per second, which
  keeps slot-update packets to the vanilla level (durability changes already cause them) and avoids resetting the
  client's block-breaking progress, which compares the held item's components.

## 5. Server and client

Tracking is **server-authoritative**. The client can see clicks but not whether the server accepted them (block
protection, cooldowns, cancelled events, failed eating), and client-written data would be trivially forgeable.
Only the server writes the component; the client only displays it.

- **Install on both sides.** Last Used registers a data component, which both sides must know: a server with
  Last Used needs Last Used on every client. Tested (`tools/jointest.sh`): a client without the mod is refused at
  login by both loaders (Fabric: "Received a registry entry that is unknown to this client"; NeoForge: "The server
  sent registries with unknown keys: ... lastused:last_used"). A client with Last Used on a server without it can
  join; it shows existing timestamps but never claims "never used" (also tested).
- **Server info packet** (`lastused:server_info`, server → client, once per login, ~20 bytes): tells the client
  that the server tracks, which items it tracks (stackable switch and disabled groups, so the client shows
  "never used" exactly for items the server would stamp), and the server's current time. The client keeps the
  clock difference and computes ages on the server time scale, so a client clock that is minutes off still shows
  correct ages. Fabric: sent only if `ServerPlayNetworking.canSend`; NeoForge: optional channel, sent only if the
  client has it.
- **Single player** runs the same server logic in the integrated server; the packet goes over the local
  connection.

## 6. Time display

- Relative age is computed **when the tooltip is drawn** from the stored value; nothing ticks, nothing is cached
  per item. Floor rounding: `just now` (< 1 min, or < 1 s with seconds enabled), `N seconds`, `minutes` (< 1 h),
  `hours` (< 1 day), `days` (< 7), `weeks` (< 30 days), `months` (30 days, < 365 days), `years`.
- **No negative ages**: a timestamp in the future (server clock moved back, or a creative player's edited item)
  shows `just now`; on the next use the future value is replaced at once instead of being throttled.
- System clock jumps forward simply make items look older; there is no tick-based correction because ticks do not
  run while the server is off.
- Optional absolute time (`tooltip.absolute`) in the viewer's time zone with a validated
  `DateTimeFormatter` pattern.
- English and German texts (`en_us`, `de_de`, `de_at`, `de_ch`), singular/plural keys per unit.

## 7. Tooltip

- Added through the loader tooltip events (Fabric `ItemTooltipCallback`, NeoForge `ItemTooltipEvent`); existing
  lines are never changed. Normal tooltips: last line. Advanced tooltips (F3+H): above the vanilla debug lines
  (durability, item id, component count).
- Gray (`ChatFormatting.GRAY`), optional 8×8 clock glyph from the own bitmap font `lastused:icons`
  (`tooltip.icon`, off by default to match the original look).
- Respects `minecraft:tooltip_display` (hiding `lastused:last_used` hides the line) and vanilla's hidden tooltips.

## 8. Configuration

`config/lastused.properties`, created with comments on first start, read **once** at startup (restart to apply).
Invalid values fall back to the default with a log warning; unknown keys are reported.

| Key | Default | Side |
|---|---|---|
| `tooltip.enabled` | true | client |
| `tooltip.show_never` | true | client |
| `tooltip.relative` | true | client |
| `tooltip.absolute` | false | client |
| `tooltip.absolute_pattern` | `yyyy-MM-dd HH:mm` | client |
| `tooltip.seconds` | false | client |
| `tooltip.icon` | false | client |
| `tracking.enabled` | true | server |
| `tracking.stackable_items` | false | server |
| `tracking.update_interval_seconds` | 1 (0–60) | server |
| `tracking.category.<shields, ranged, consumables, weapons, armor, tools, blocks, other>` | true | server |

If both `tooltip.relative` and `tooltip.absolute` are false, the relative age is shown (warning in the log).

## 9. Performance

- Event-based: work happens only in the hooked actions, a few component reads and at most one component write.
- No inventory scans and no per-tick work: the only tick listener is the client boot check, which returns at once
  unless `-Dlastused.bootTest=true`. No background threads (the join-test watchdog exists only with
  `-Dlastused.joinServer=true`), no file access after startup, one network packet per login.
- Tooltip: one component lookup, one category check and a string format while a tooltip is visible.

No benchmark numbers are claimed; nothing was measured beyond the automated tests.

## 10. Differences to the original (as described on its project page)

| Original description | Last Used | Reason |
|---|---|---|
| left-click on blocks counts | only a completed block break counts | a click that breaks nothing does not use the item (no durability, no statistic) |
| right-click on blocks/entities counts | only when the item itself did something (`wasItemInteraction`) | opening a chest with a sword in hand is not a sword use |
| per-item storage (stack behaviour not described) | unstackable items by default, stackable optional | different stamps prevent vanilla stacking; avoids inventory fragmentation |
| Forge 1.16.1–1.21.11 | Fabric and NeoForge 26.3 | target of this project |
| – | durability uses (shield blocks, armor hits, elytra) count | they are real uses without a click |
| – | server info packet, clock-offset correction, "never" only where tracked | correct display in multiplayer |

## 11. Tests

| Level | What | Where |
|---|---|---|
| Unit (JUnit, `./gradlew :core:test`) | time formatting, never-used state, throttle and clock changes, tracking rules, config parsing/validation, absolute format | `core/src/test` |
| GameTests (server, both loaders) | pickaxe mining, creative mining, non-tool mining, sword hit, durability use and disabled group, failed use, opening a chest, eating (full/hungry/finish), placing (default/stackable), chest + hopper, death drop + pickup by a second player + throw, split/merge rules, NBT and network serialization, throttle/clock, only players write | `common/.../gametest` |
| Client boot check (both loaders) | fresh world, server info received, language loaded, tooltip "5 minutes ago", "never" for a new sword, nothing for stackables, hidden component, advanced tooltip order, re-equip flag | `common/src/client/.../ClientSelfTests` |
| Restart probe (dedicated server, both loaders) | write a chest with a used and an unused sword, stop, start a new server process, read them back | `PersistProbe`, `-Dlastused.persistTest=write|verify` |
| Join tests (release jars, dedicated server + client over the network, both loaders) | mod on both sides (server info, self tests, server-stamped sword arrives with its timestamp); client with mod on a server without it; client without mod on a server with it | `tools/jointest.sh` |
| Tooltip screenshots (dev client) | default look, clock symbol + absolute time, never used | `TooltipShots`, `-Dlastused.tooltipShot=true` |

What automated tests cannot cover (real mouse input, two real clients on one server, visual look) is listed in
`docs/MANUAL_TESTS.md`.
