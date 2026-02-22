# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build

```
mvn clean install
```

The output JAR is built with `maven-shade-plugin` (not Gradle shadow). HikariCP and minelib-scheduler are shaded and relocated under `pk.ajneb97.libs.*`.

Always run the build after code changes and fix any compile errors before finishing.

## Architecture

**Main class**: `pk.ajneb97.PlayerKits2` — initialises all managers in `onEnable()` in dependency order. No service interface pattern; managers are plain classes instantiated directly and accessed via plugin getters.

**Package**: `pk.ajneb97`

### Config system

Raw Bukkit `FileConfiguration` / YAML — **not** ConfigLib. Each config type has its own manager:

- `ConfigsManager` — orchestrates all sub-managers; `reload()` triggers all of them
- `KitsConfigManager` — loads per-kit YAML files from `plugins/PlayerKits2/kits/`
- `MainConfigManager` — `config.yml`
- `MessagesConfigManager` — `messages.yml`
- `PlayersConfigManager` — flat-file player data (used only when MySQL is disabled)
- `InventoryConfigManager` — `inventory.yml` (defines GUI layouts)

Messages are **never hard-coded** — always use `messagesConfig.getString("<key>")` with replacements like `.replace("%kit%", kitName)`. Message sending goes through `MessagesManager.sendMessage()` which applies the prefix.

### Data storage

Two modes (configured in `config.yml`):
- **Flat-file**: `PlayersConfigManager` saves `PlayerData` to YAML; `PlayerDataSaveTask` periodically saves modified records.
- **MySQL**: `MySQLConnection` (HikariCP-backed); all DB operations run async via `AsyncScheduler`, then callback to main thread via `GlobalScheduler`.

In-memory state is `PlayerDataManager.players: Map<UUID, PlayerData>`.

### Scheduler (Folia compatibility)

Uses **minelib-scheduler** (`io.github.projectunified.minelib`), relocated to `pk.ajneb97.libs.minelib`:
- `AsyncScheduler.get(plugin).run(...)` — async tasks
- `GlobalScheduler.get(plugin).run(...)` — main-thread tasks
- `EntityScheduler` / common scheduler variants also available

Do **not** use `Bukkit.getScheduler()` — it is not Folia-compatible.

### GUI / Inventory system

Custom inventory system — **not** InvUI. Key classes:
- `KitInventory` — layout loaded from `inventory.yml`; holds `List<ItemKitInventory>` (slots + item definitions)
- `InventoryPlayer` — per-player state tracking which inventory is open and which kit is selected
- `InventoryManager` — opens inventories, handles clicks, routes to kit-give or sub-inventory navigation
- `InventoryEditManager` — in-game kit editor GUI
- Items are tagged using the Paper PersistentDataContainer via `ItemUtils.setTagStringItem` / `getTagStringItem` with keys like `playerkits_kit`, `playerkits_kit_status`, `playerkits_open_inventory`, etc.

Kit display items have four states: `default`, `cooldown`, `no_permission`, `one_time` (and `one_time_requirements`).

### Kit items: two save modes

- **original** — stores the raw `ItemStack` (full fidelity, not user-editable in config)
- **configurable** — stores individual properties (material, name, lore, enchants, etc.) as YAML fields

Mode is set per-kit (`saveOriginalItems` flag on `Kit`) and chosen at creation time.

### Commands

Vanilla Bukkit `CommandExecutor` / `TabCompleter` — **not** CommandAPI. Single command `kit` (aliases: `kits`, `playerkits`) handled by `MainCommand`. Admin check via `PlayerUtils.isPlayerKitsAdmin(sender)` (permission `playerkits.admin`).

### Key managers summary

| Manager | Responsibility |
|---|---|
| `KitsManager` | Kit CRUD, `giveKit()` logic, permission/cooldown/requirement checks |
| `PlayerDataManager` | In-memory player data (cooldowns, one-time flags, bought flags) |
| `InventoryManager` | GUI open/click/navigation |
| `InventoryEditManager` | In-game kit editor |
| `KitItemManager` | Build `ItemStack` from `KitItem`, serialize/deserialize kit items |
| `MessagesManager` | Send messages with prefix, legacy color parsing |
| `DependencyManager` | Vault economy + PlaceholderAPI hooks |
| `VerifyManager` | Startup config validation |
| `MigrationManager` | Legacy data migration |

### External integrations

- **Vault** (soft-depend) — economy for kit purchase prices
- **PlaceholderAPI** (soft-depend) — extra requirement conditions + `%playerkits_*%` placeholders via `ExpansionPlayerKits`
- **bStats** — `Metrics` with plugin ID 19795
