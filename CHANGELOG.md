# Changelog

## 1.0.0 – first release (Minecraft 26.3, Fabric and NeoForge)

Based on the concept of Last Use Tooltip by perusi. Independently developed for Minecraft 26.3 Fabric and NeoForge.

- Tooltip line "Last used: …" / "Last used: never" (English, German), gray, below the item information.
- Server-side tracking of real uses: right-click with the item, item use on blocks and mobs, melee hits, block
  breaking with tools (survival and creative), bow/trident release, finished eating, durability use (shields,
  armor, elytra, shears, ...). Failed and cancelled attempts do not count.
- Timestamp as item data component `lastused:last_used` (Unix epoch milliseconds), saved with the item and synced
  to clients; no re-equip animation when it changes.
- Unstackable items tracked by default; stackable items optional (`tracking.stackable_items`) to avoid inventory
  fragmentation.
- Server info packet at login: "never" only for items the server tracks; client clock offset correction.
- Config file `config/lastused.properties`: relative/absolute time, seconds, clock symbol, per-group tracking,
  update interval.
