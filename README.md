# Minecraft Server Configuration Repository (for tidy)

This repository serves as an example configuration repository for **tidy**, managing Minecraft server configs and plugin settings (`.yml`, `.yaml`, `.json`, `.properties`, `.conf`, `.toml`).

All large binary files, runtime state, and persistent data are ignored via [.gitignore](.gitignore) so the repository remains lightweight and clean.

---

## What is Tracked vs Ignored

| Category | Tracked | Ignored |
|---|---|---|
| **Server Settings** | `server.properties`, `bukkit.yml`, `spigot.yml`, `commands.yml`, `wepif.yml`, `config/*.yml` | Cache (`.paper/`), player UUID cache (`usercache.json`), session locks |
| **Plugin Configs** | `plugins/<Plugin>/*.yml`, `*.json`, `*.properties`, `*.conf` | Plugin binaries (`*.jar`), SQLite/H2 databases (`*.db`, `*.sqlite*`), archives (`*.zip`) |
| **World Data** | None | Entire `world/`, `world_nether/`, `world_the_end/`, region files (`*.mca`), playerdata |
| **Server Logs** | None | `logs/`, `crash-reports/`, `*.log` |

---

## Using `tidy-git`

A dedicated helper tool, `tidy-git`, is provided to inspect, stage, and audit configurations.

### 1. View Detected Configs and Git Status
```bash
./tidy-git status
```
Shows which configs are untracked, staged, or modified, along with a tally of ignored runtime files.

### 2. Stage Configs to Git
Stage all detected configurations:
```bash
./tidy-git add
```

Stage configs for a specific plugin only:
```bash
./tidy-git add -p ViaVersion
./tidy-git add -p spark
```

Preview what would be staged without modifying Git:
```bash
./tidy-git add --dry-run
```

Stage configs excluding player moderation lists (`ops.json`, `whitelist.json`, `banned-*.json`):
```bash
./tidy-git add --exclude-player-lists
```

### 3. List Tracked Configs
```bash
./tidy-git list
# Or output plain paths for scripts:
./tidy-git list --raw
# Or JSON output:
./tidy-git list --json
```

### 4. Audit Repository Hygiene
```bash
./tidy-git check
```
Verifies that no world files, `.jar` files, database files (`.db`), or cache files are tracked in Git.

### 5. Commit Changes
```bash
./tidy-git commit -m "Update ViaVersion and spark configs"
```
Stages all configuration files and creates the commit in one step.
