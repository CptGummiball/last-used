# Last Used API for mod developers

API version **1** · package `dev.cptgummiball.lastused.api` · mod id `lastused`

**Most mods need nothing from this page.** Last Used adds its tooltip line through the standard tooltip events of
Fabric and NeoForge and does not change other lines or item behaviour.

## Reading the timestamp

The time of the last use is a regular item data component, so you can read it without the API jar:

| | |
|---|---|
| Component id | `lastused:last_used` (`LastUsedApi.COMPONENT_ID`) |
| Value | `Long`: Unix epoch milliseconds of the server clock |
| Absent | the item was never used since Last Used tracks it |
| Tracked items | unstackable items by default; servers may enable stackable items |

```java
DataComponentType<?> type = BuiltInRegistries.DATA_COMPONENT_TYPE.getValue(Identifier.parse("lastused:last_used"));
if (type != null && stack.get(type) instanceof Long lastUsed) {
    long ageMillis = System.currentTimeMillis() - lastUsed;
}
```

Only read it. The server-side tracker owns the value; writing it from other mods is not supported.

## Getting the API jar

`lastused-api-1.jar` and `lastused-api-1-javadoc.jar` from the [releases](https://github.com/CptGummiball/last-used/releases),
as a **compile-only** dependency, if you want the constants. Do not ship the API or any Last Used files in your mod.
If Last Used is optional for your mod, check that the mod `lastused` is loaded before touching its classes.

The license (section 5) allows compiling against the API and publishing mods that use it.
