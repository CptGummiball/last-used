# Credits

> **Based on the concept of Last Use Tooltip by perusi. Independently developed for Minecraft 26.3 Fabric and NeoForge.**

## Original project

| | |
|---|---|
| Name | Last Use Tooltip |
| Author | **perusi** (owner of the project on Modrinth) |
| Project page | <https://modrinth.com/mod/lastusetooltip> |
| Platform of the original | Minecraft Forge, 1.16.1 – 1.21.11 |
| License of the original | MIT (as listed on Modrinth; checked 2026-11-17) |

The author and license were read from the public Modrinth project data on 2026-11-17
(`api.modrinth.com/v2/project/lastusetooltip`, team member `perusi`, role Owner; license id `MIT`).
The original project lists no source repository.

## Functional inspiration

Last Used takes the *idea* of Last Use Tooltip: hovering an item shows when it was last used
("Last used: 12 minutes ago", "Last used: never"), updated by right-clicks with the item, interactions with
blocks and entities, attacks and block breaking.

## Independent reimplementation

- Every class, build file, data file, text and image of Last Used was written for this project. No code, classes,
  assets or build files of Last Use Tooltip were copied or decompiled; the original served only as a functional
  reference (its public project description).
- Last Used uses its own identifiers (mod id `lastused`, component `lastused:last_used`, package
  `dev.cptgummiball.lastused`) and its own metadata.
- Some behaviour deliberately differs from the description of the original (see `ARCHITECTURE.md`, section
  "Differences to the original"): for example, a mere left click on a block without breaking it does not count
  as a use, and stackable items are not tracked by default so inventories do not fragment.

## No affiliation

Last Used is **not affiliated with, endorsed by or connected to** perusi or the Last Use Tooltip project.
Please report problems with Last Used to the Last Used issue tracker, not to the original author.

## Libraries and platforms

Built with Fabric Loader, Fabric API, NeoForge, Mixin (all used as provided by the loaders; nothing is bundled)
and the official Minecraft mappings.
