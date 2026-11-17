# Last Used API for mod developers

API version **1** · package `dev.cptgummiball.lastused.api` · mod id `lastused`

**Most mods need nothing from this page** if Last Used uses the standard loader interfaces.

## Getting the API

`lastused-api-1.jar` and `lastused-api-1-javadoc.jar` from the [releases](../../../releases),
as a **compile-only** dependency. Do not ship the API or any Last Used files in your mod. If
Last Used is optional for your mod, check that the mod `lastused` is loaded before touching its
classes.

The license (section 5) allows compiling against the API and publishing mods that use it.
