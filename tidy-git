#!/usr/bin/env python3
"""
tidy-git: A configuration management assistant for Minecraft servers and 'tidy'.

Scans for Minecraft server and plugin configuration files (.yml, .yaml, .json, .properties, etc.),
stages them cleanly to Git, and ensures worlds, JARs, databases, and runtime caches stay ignored.
"""

import argparse
import fnmatch
import os
import shutil
import subprocess
import sys
from pathlib import Path
from typing import Dict, List, Optional, Set, Tuple

# Configuration file extensions recognized by tidy-git
CONFIG_EXTENSIONS = {
    ".yml",
    ".yaml",
    ".json",
    ".properties",
    ".toml",
    ".conf",
    ".txt",
}

# Files that should NEVER be staged as config even if extension matches
ALWAYS_IGNORE_FILENAMES = {
    "usercache.json",
    "session.lock",
    "uid.dat",
    "about.txt",  # plugin temp dumps
}

# Extensions that are binary, runtime databases, or archives
BINARY_AND_DATA_EXTENSIONS = {
    ".jar",
    ".db",
    ".sqlite",
    ".sqlite3",
    ".h2.db",
    ".mv.db",
    ".zip",
    ".tar",
    ".gz",
    ".schem",
    ".schematic",
    ".litematic",
    ".mca",
    ".dat",
    ".dump",
    ".bin",
    ".class",
}

# Directories that should never be traversed for configs
IGNORED_DIRECTORIES = {
    "world",
    "world_nether",
    "world_the_end",
    "world_the_end_nether",
    ".git",
    ".paper",
    ".purpur",
    "libraries",
    "bundler",
    "cache",
    "logs",
    "crash-reports",
    "sessions",
    "session",
    "tmp",
    "temp",
    "updater",
    "schematics",
}

# Player lists that users might want to treat as config or exclude
PLAYER_LIST_FILES = {
    "ops.json",
    "whitelist.json",
    "banned-ips.json",
    "banned-players.json",
}

# Max allowed config file size (2 MB) to prevent accidental large binary tracking
MAX_CONFIG_SIZE_BYTES = 2 * 1024 * 1024


class Colors:
    def __init__(self, enable: bool = True):
        self.enable = enable and sys.stdout.isatty() and "NO_COLOR" not in os.environ

    @property
    def RESET(self): return "\033[0m" if self.enable else ""
    @property
    def BOLD(self): return "\033[1m" if self.enable else ""
    @property
    def GREEN(self): return "\033[32m" if self.enable else ""
    @property
    def CYAN(self): return "\033[36m" if self.enable else ""
    @property
    def YELLOW(self): return "\033[33m" if self.enable else ""
    @property
    def RED(self): return "\033[31m" if self.enable else ""
    @property
    def MAGENTA(self): return "\033[35m" if self.enable else ""
    @property
    def GRAY(self): return "\033[90m" if self.enable else ""


def find_server_root(start_dir: Optional[Path] = None) -> Path:
    """Finds server root by checking for server.properties or plugins/."""
    current = (start_dir or Path.cwd()).resolve()
    while current != current.parent:
        if (current / "server.properties").exists() or (current / "plugins").is_dir():
            return current
        current = current.parent
    return (start_dir or Path.cwd()).resolve()


def run_git_cmd(args: List[str], cwd: Path) -> Tuple[int, str, str]:
    """Executes a git command returning (exit_code, stdout, stderr)."""
    try:
        proc = subprocess.run(
            ["git"] + args,
            cwd=cwd,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            check=False,
        )
        return proc.returncode, proc.stdout, proc.stderr
    except FileNotFoundError:
        return -1, "", "git executable not found in PATH"


def is_git_repo(root: Path) -> bool:
    """Checks if directory is inside a git work tree."""
    code, out, _ = run_git_cmd(["rev-parse", "--is-inside-work-tree"], cwd=root)
    return code == 0 and out.strip() == "true"


def get_git_status_map(root: Path) -> Dict[str, str]:
    """Returns mapping of relative file path -> status code (e.g. '?', 'M', 'A', ' ')"""
    if not is_git_repo(root):
        return {}
    code, out, _ = run_git_cmd(["status", "--porcelain", "-uall"], cwd=root)
    if code != 0:
        return {}
    status_map = {}
    for line in out.splitlines():
        if len(line) >= 4:
            st = line[:2].strip()
            filepath = line[3:].strip()
            # Normalize path
            status_map[filepath] = st
    return status_map


def is_git_ignored(filepath: Path, root: Path) -> bool:
    """Uses git check-ignore if in git repo, otherwise basic heuristics."""
    rel = filepath.relative_to(root)
    rel_str = str(rel)

    # Check hardcoded directory ignores
    for part in rel.parts[:-1]:
        if part in IGNORED_DIRECTORIES:
            return True

    if filepath.name in ALWAYS_IGNORE_FILENAMES:
        return True

    if is_git_repo(root):
        code, out, _ = run_git_cmd(["check-ignore", "-q", rel_str], cwd=root)
        if code == 0:
            return True

    return False


def scan_repository(root: Path) -> Tuple[List[Path], Dict[str, List[Path]], Dict[str, int]]:
    """
    Scans the server directory for:
    - Root config files
    - Plugin config files (grouped by plugin)
    - Ignored files stats
    """
    root_configs: List[Path] = []
    plugin_configs: Dict[str, List[Path]] = {}
    ignored_counts: Dict[str, int] = {
        "world_dirs": 0,
        "jars": 0,
        "databases": 0,
        "archives": 0,
        "caches_and_temp": 0,
        "other_ignored": 0,
    }

    plugins_dir = root / "plugins"

    for current_dir, dirnames, filenames in os.walk(root):
        cur_p = Path(current_dir)
        rel_dir = cur_p.relative_to(root)

        # Check if this entire directory should be pruned
        if any(part in IGNORED_DIRECTORIES for part in rel_dir.parts):
            if "world" in rel_dir.parts:
                ignored_counts["world_dirs"] += 1
            dirnames.clear()
            continue

        # Prune known ignored child directories from descending
        pruned = []
        for d in list(dirnames):
            if d in IGNORED_DIRECTORIES or d.startswith("."):
                pruned.append(d)
                dirnames.remove(d)
                if "world" in d:
                    ignored_counts["world_dirs"] += 1
                elif d in {"tmp", "temp", "sessions", "session", "cache", ".paper"}:
                    ignored_counts["caches_and_temp"] += 1

        for f in filenames:
            file_path = cur_p / f
            ext = file_path.suffix.lower()
            name = file_path.name.lower()

            # Tally ignored categories
            if ext == ".jar":
                ignored_counts["jars"] += 1
                continue
            elif ext in {".db", ".sqlite", ".sqlite3", ".h2.db", ".mv.db"}:
                ignored_counts["databases"] += 1
                continue
            elif ext in {".zip", ".tar", ".gz", ".schem", ".schematic"}:
                ignored_counts["archives"] += 1
                continue
            elif name in ALWAYS_IGNORE_FILENAMES or "tmp" in rel_dir.parts or "sessions" in rel_dir.parts:
                ignored_counts["caches_and_temp"] += 1
                continue

            # Check if it's a valid config extension
            if ext in CONFIG_EXTENSIONS or name == "server-icon.png":
                # Check file size limit
                try:
                    if file_path.stat().st_size > MAX_CONFIG_SIZE_BYTES:
                        ignored_counts["other_ignored"] += 1
                        continue
                except OSError:
                    continue

                if is_git_ignored(file_path, root):
                    ignored_counts["other_ignored"] += 1
                    continue

                # Determine if root config, config/ subdir, or plugin config
                if cur_p == root or cur_p == root / "config":
                    root_configs.append(file_path)
                elif plugins_dir in file_path.parents:
                    # Belongs to a plugin
                    rel_to_plugins = file_path.relative_to(plugins_dir)
                    plugin_name = rel_to_plugins.parts[0]
                    plugin_configs.setdefault(plugin_name, []).append(file_path)
                else:
                    root_configs.append(file_path)

    root_configs.sort()
    for p in plugin_configs:
        plugin_configs[p].sort()

    return root_configs, plugin_configs, ignored_counts


def format_size(size_bytes: int) -> str:
    """Formats bytes into human readable string."""
    if size_bytes < 1024:
        return f"{size_bytes} B"
    elif size_bytes < 1024 * 1024:
        return f"{size_bytes / 1024:.1f} KB"
    else:
        return f"{size_bytes / (1024 * 1024):.1f} MB"


def cmd_init(args, root: Path, c: Colors):
    """Initializes Git repository and creates recommended .gitignore and .gitattributes."""
    print(f"{c.BOLD}{c.CYAN}Initializing tidy-git repository at:{c.RESET} {root}")

    if not (root / ".git").exists():
        code, out, err = run_git_cmd(["init"], cwd=root)
        if code == 0:
            print(f"  {c.GREEN}✓ Initialized new Git repository{c.RESET}")
        else:
            print(f"  {c.RED}✗ Git init failed: {err}{c.RESET}")
    else:
        print(f"  {c.GRAY}• Existing Git repository found{c.RESET}")

    gitignore_path = root / ".gitignore"
    if not gitignore_path.exists() or args.force:
        default_gitignore = """# =============================================================================
# Minecraft Server Configuration .gitignore (Tailored for tidy)
# =============================================================================

# --- Server Executables, Bundlers & Libraries ---
*.jar
libraries/
bundler/
versions/
cache/
.paper/
.purpur/
.fabric/
.quilt/

# --- World Directories & Data ---
/world/
/world_nether/
/world_the_end/
/world_the_end_nether/
**/region/
**/entities/
**/poi/
**/playerdata/
**/stats/
**/advancements/
**/level.dat*
**/session.lock
**/uid.dat
*.mca

# --- Runtime Databases & Data Storage ---
*.db
*.sqlite*
*.h2.db
*.mv.db
*.dat
!server-icon.png

# --- Runtime Caches, Sessions & Ephemeral Files ---
usercache.json
**/sessions/
**/session/
**/tmp/
**/temp/
**/updater/
**/cache/
**/backups/
**/backup/

# --- Logs & Crash Reports ---
logs/
crash-reports/
*.log
*.log.gz

# --- Binary Assets, Archives & World Exports ---
*.zip
*.tar
*.tar.gz
*.schem
*.schematic
*.litematic
*.dump
*.bin
*.class
*.exe

# --- OS & Editor Files ---
.DS_Store
Thumbs.db
.idea/
.vscode/
"""
        gitignore_path.write_text(default_gitignore, encoding="utf-8")
        print(f"  {c.GREEN}✓ Created optimized .gitignore{c.RESET}")
    else:
        print(f"  {c.GRAY}• .gitignore already exists (use --force to overwrite){c.RESET}")

    gitattributes_path = root / ".gitattributes"
    if not gitattributes_path.exists() or args.force:
        default_gitattributes = """# Normalize line endings to LF for configs
* text=auto eol=lf
*.yml text eol=lf
*.yaml text eol=lf
*.json text eol=lf
*.properties text eol=lf
*.toml text eol=lf
*.conf text eol=lf
*.txt text eol=lf
*.png binary
*.jar binary
*.db binary
*.sqlite binary
*.zip binary
"""
        gitattributes_path.write_text(default_gitattributes, encoding="utf-8")
        print(f"  {c.GREEN}✓ Created .gitattributes with LF normalization{c.RESET}")
    else:
        print(f"  {c.GRAY}• .gitattributes already exists{c.RESET}")

    print(f"\n{c.GREEN}{c.BOLD}Done!{c.RESET} Run {c.CYAN}./tidy-git status{c.RESET} to inspect detectable configs.")


def cmd_status(args, root: Path, c: Colors):
    """Inspects and displays detectable configs, git status, and ignored files."""
    root_configs, plugin_configs, ignored = scan_repository(root)
    git_map = get_git_status_map(root)
    in_git = is_git_repo(root)

    total_configs = len(root_configs) + sum(len(files) for files in plugin_configs.values())
    total_size = sum(f.stat().st_size for f in root_configs if f.exists()) + sum(
        f.stat().st_size for files in plugin_configs.values() for f in files if f.exists()
    )

    print(f"{c.BOLD}=== tidy-git Configuration Status ==={c.RESET}")
    print(f"Repository Root : {c.CYAN}{root}{c.RESET}")
    print(f"Git Initialized : {'Yes' if in_git else c.YELLOW + 'No (run ./tidy-git init)' + c.RESET}")
    print(f"Total Configs   : {c.GREEN}{total_configs}{c.RESET} ({format_size(total_size)})\n")

    def print_file_entry(f: Path):
        rel = f.relative_to(root)
        size_str = format_size(f.stat().st_size)
        st_symbol = " "
        st_color = c.GRAY
        if str(rel) in git_map:
            code = git_map[str(rel)]
            if "?" in code:
                st_symbol = "?"
                st_color = c.YELLOW
            elif "A" in code:
                st_symbol = "+"
                st_color = c.GREEN
            elif "M" in code:
                st_symbol = "M"
                st_color = c.CYAN
            else:
                st_symbol = code
                st_color = c.MAGENTA
        elif in_git:
            st_symbol = "✓"
            st_color = c.GRAY

        print(f"  [{st_color}{st_symbol}{c.RESET}] {rel} {c.GRAY}({size_str}){c.RESET}")

    # 1. Root & Server configs
    print(f"{c.BOLD}{c.CYAN}● Server Root Configs ({len(root_configs)}):{c.RESET}")
    for f in root_configs:
        print_file_entry(f)
    print()

    # 2. Plugin configs grouped
    print(f"{c.BOLD}{c.CYAN}● Plugin Configurations ({len(plugin_configs)} plugins):{c.RESET}")
    for plugin_name, files in sorted(plugin_configs.items()):
        p_size = sum(f.stat().st_size for f in files if f.exists())
        print(f"  {c.BOLD}▸ {plugin_name}{c.RESET} {c.GRAY}({len(files)} configs, {format_size(p_size)}){c.RESET}")
        for f in files:
            print_file_entry(f)
    print()

    # 3. Deleted tracked files (if any)
    deleted_files = []
    if in_git:
        code, out, _ = run_git_cmd(["ls-files", "--deleted"], cwd=root)
        if code == 0:
            deleted_files = [line.strip() for line in out.splitlines() if line.strip()]

    if deleted_files:
        print(f"{c.BOLD}{c.RED}● Deleted Files Pending Removal ({len(deleted_files)}):{c.RESET}")
        for df in deleted_files:
            print(f"  [{c.RED}D{c.RESET}] {df}")
        print()

    # 4. Ignored summary
    print(f"{c.BOLD}{c.MAGENTA}● Ignored / Filtered Files (Safe from Git):{c.RESET}")
    print(f"  • World directories : {c.GRAY}{ignored['world_dirs']} directories ignored (world/, etc.){c.RESET}")
    print(f"  • Plugin JAR files  : {c.GRAY}{ignored['jars']} .jar files ignored{c.RESET}")
    print(f"  • Databases (.db)   : {c.GRAY}{ignored['databases']} database files ignored{c.RESET}")
    print(f"  • Archives (.zip)   : {c.GRAY}{ignored['archives']} archive files ignored{c.RESET}")
    print(f"  • Caches & Temp     : {c.GRAY}{ignored['caches_and_temp']} temp/cache files ignored (usercache.json, tmp, etc.){c.RESET}")
    if ignored["other_ignored"] > 0:
        print(f"  • Other ignored     : {c.GRAY}{ignored['other_ignored']} items matching .gitignore{c.RESET}")

    if in_git:
        print(f"\n{c.GRAY}Legend: [?] untracked, [+] staged, [M] modified, [D] deleted, [✓] clean{c.RESET}")
    print(f"Next step: {c.CYAN}./tidy-git add{c.RESET} to stage configs for commit.")


def cmd_add(args, root: Path, c: Colors):
    """Stages configuration files to Git index."""
    if not is_git_repo(root):
        print(f"{c.RED}Error: Git repository not initialized.{c.RESET} Run {c.CYAN}./tidy-git init{c.RESET} first.")
        sys.exit(1)

    root_configs, plugin_configs, _ = scan_repository(root)
    files_to_stage: List[Path] = []

    # Filter files
    if args.root:
        files_to_stage.extend(root_configs)
    elif args.plugin:
        targets = [p.lower() for p in args.plugin]
        for p_name, files in plugin_configs.items():
            if p_name.lower() in targets:
                files_to_stage.extend(files)
        if not files_to_stage:
            print(f"{c.YELLOW}Warning: No configs found matching plugin(s): {', '.join(args.plugin)}{c.RESET}")
            return
    else:
        # Stage all
        files_to_stage.extend(root_configs)
        for files in plugin_configs.values():
            files_to_stage.extend(files)

    # Exclude player lists if requested
    if args.exclude_player_lists:
        files_to_stage = [f for f in files_to_stage if f.name not in PLAYER_LIST_FILES]

    # Always include .gitignore and .gitattributes if they exist
    for meta_file in [".gitignore", ".gitattributes"]:
        p = root / meta_file
        if p.exists() and p not in files_to_stage:
            files_to_stage.append(p)

    # Check for deleted tracked files
    deleted_to_stage = []
    if is_git_repo(root):
        code, out, _ = run_git_cmd(["ls-files", "--deleted"], cwd=root)
        if code == 0:
            for line in out.splitlines():
                del_path = line.strip()
                if not del_path:
                    continue
                if args.plugin:
                    targets = [p.lower() for p in args.plugin]
                    parts = Path(del_path).parts
                    if len(parts) >= 2 and parts[0] == "plugins" and parts[1].lower() in targets:
                        deleted_to_stage.append(del_path)
                elif args.root:
                    if len(Path(del_path).parts) == 1:
                        deleted_to_stage.append(del_path)
                else:
                    deleted_to_stage.append(del_path)

    if not files_to_stage and not deleted_to_stage:
        print(f"{c.YELLOW}No configuration files to stage.{c.RESET}")
        return

    rel_paths = [str(f.relative_to(root)) for f in files_to_stage]

    total_changes = len(rel_paths) + len(deleted_to_stage)
    print(f"{c.BOLD}{c.CYAN}Staging {total_changes} changes to Git ({len(rel_paths)} files, {len(deleted_to_stage)} removals)...{c.RESET}")

    if args.dry_run:
        print(f"{c.YELLOW}[DRY RUN] Would execute git add / git rm:{c.RESET}")
        for p in rel_paths:
            print(f"  + {p}")
        for p in deleted_to_stage:
            print(f"  - {p} (deleted)")
        return

    # Add in batches to avoid any command line length limits
    batch_size = 50
    staged_count = 0
    for i in range(0, len(rel_paths), batch_size):
        batch = rel_paths[i:i + batch_size]
        code, out, err = run_git_cmd(["add"] + batch, cwd=root)
        if code != 0:
            print(f"{c.RED}Git add failed for batch: {err}{c.RESET}")
            sys.exit(1)
        staged_count += len(batch)

    # Stage removals
    deleted_count = 0
    if deleted_to_stage:
        code, out, err = run_git_cmd(["rm", "-q"] + deleted_to_stage, cwd=root)
        if code != 0:
            run_git_cmd(["add", "-u"] + deleted_to_stage, cwd=root)
        deleted_count = len(deleted_to_stage)

    print(f"{c.GREEN}✓ Successfully staged {staged_count} files, {deleted_count} removals!{c.RESET}")
    print(f"Run {c.CYAN}./tidy-git status{c.RESET} or {c.CYAN}git status{c.RESET} to review staged changes.")


def cmd_list(args, root: Path, c: Colors):
    """Lists all configuration files in plain format or JSON."""
    root_configs, plugin_configs, _ = scan_repository(root)
    all_files = list(root_configs)
    for files in plugin_configs.values():
        all_files.extend(files)

    if args.json:
        result = {
            "root_configs": [str(f.relative_to(root)) for f in root_configs],
            "plugins": {
                p: [str(f.relative_to(root)) for f in files]
                for p, files in plugin_configs.items()
            },
        }
        print(json.dumps(result, indent=2))
        return

    if args.raw:
        for f in all_files:
            print(f.relative_to(root))
        return

    print(f"{c.BOLD}{'Path':<50} {'Size':<10}{c.RESET}")
    print("-" * 62)
    for f in all_files:
        rel = str(f.relative_to(root))
        size = format_size(f.stat().st_size)
        print(f"{rel:<50} {size:<10}")


def cmd_check(args, root: Path, c: Colors):
    """Audits repository for hygiene: ensures no worlds, jars, or databases are tracked in Git."""
    print(f"{c.BOLD}{c.CYAN}Running repository hygiene audit...{c.RESET}\n")
    if not is_git_repo(root):
        print(f"{c.YELLOW}Git not initialized. Initializing audit on filesystem only.{c.RESET}")

    issues_found = 0

    # 1. Check if .gitignore exists and contains world/
    gi = root / ".gitignore"
    if not gi.exists():
        print(f"  {c.RED}✗ Missing .gitignore! Worlds and binaries may be accidentally tracked.{c.RESET}")
        issues_found += 1
    else:
        content = gi.read_text(encoding="utf-8", errors="ignore")
        if "world" not in content:
            print(f"  {c.YELLOW}⚠ .gitignore does not explicitly mention 'world/'.{c.RESET}")
            issues_found += 1
        else:
            print(f"  {c.GREEN}✓ .gitignore properly excludes world directories.{c.RESET}")

        if "*.jar" not in content:
            print(f"  {c.YELLOW}⚠ .gitignore does not explicitly exclude '*.jar'.{c.RESET}")
            issues_found += 1
        else:
            print(f"  {c.GREEN}✓ .gitignore properly excludes JAR files.{c.RESET}")

    # 2. Check tracked files in Git
    if is_git_repo(root):
        code, out, _ = run_git_cmd(["ls-files"], cwd=root)
        if code == 0:
            tracked = out.splitlines()
            forbidden_patterns = ["world/*", "*.jar", "*.db", "*.sqlite*", "usercache.json", "*.zip"]
            for path in tracked:
                for pat in forbidden_patterns:
                    if fnmatch.fnmatch(path, pat):
                        print(f"  {c.RED}✗ Forbidden tracked file found in Git: {path}{c.RESET}")
                        issues_found += 1

    if issues_found == 0:
        print(f"\n{c.GREEN}{c.BOLD}Audit passed cleanly!{c.RESET} No worlds, JARs, or runtime databases in Git.")
    else:
        print(f"\n{c.RED}{c.BOLD}Audit found {issues_found} issue(s).{c.RESET} Run {c.CYAN}./tidy-git init --force{c.RESET} to fix .gitignore.")


def cmd_diff(args, root: Path, c: Colors):
    """Shows git diff for configuration files."""
    if not is_git_repo(root):
        print(f"{c.RED}Error: Git repository not initialized.{c.RESET}")
        sys.exit(1)
    extra = ["--staged"] if args.staged else []
    code, out, err = run_git_cmd(["diff"] + extra, cwd=root)
    if out:
        print(out)
    elif code == 0:
        print(f"{c.GRAY}No diff found.{c.RESET}")
    else:
        print(f"{c.RED}{err}{c.RESET}")


def cmd_commit(args, root: Path, c: Colors):
    """Stages all configs and commits with the given message."""
    if not is_git_repo(root):
        print(f"{c.RED}Error: Git repository not initialized.{c.RESET}")
        sys.exit(1)

    # Automatically run add first
    cmd_add(args, root, c)

    code, out, err = run_git_cmd(["commit", "-m", args.message], cwd=root)
    if code == 0:
        print(f"\n{c.GREEN}{c.BOLD}✓ Commit created successfully:{c.RESET}\n{out}")
    else:
        print(f"\n{c.RED}Commit failed or nothing to commit: {err or out}{c.RESET}")


def main():
    parser = argparse.ArgumentParser(
        prog="tidy-git",
        description="Minecraft server configuration Git helper for tidy.",
    )
    parser.add_argument(
        "--root",
        type=Path,
        default=None,
        help="Specify Minecraft server root directory (defaults to auto-detected root)",
    )
    parser.add_argument(
        "--no-color",
        action="store_true",
        help="Disable ANSI colors in terminal output",
    )

    subparsers = parser.add_subparsers(dest="command", help="Available commands")

    # init
    p_init = subparsers.add_parser("init", help="Initialize Git and create .gitignore & .gitattributes")
    p_init.add_argument("--force", action="store_true", help="Overwrite existing .gitignore/.gitattributes")

    # status / scan
    subparsers.add_parser("status", help="Scan configs, inspect git status, and view ignored files")

    # add / stage
    p_add = subparsers.add_parser("add", help="Stage configuration files to Git")
    p_add.add_argument("-p", "--plugin", action="append", help="Stage only specific plugin configs (repeatable)")
    p_add.add_argument("--root-only", dest="root", action="store_true", help="Stage only root server configs")
    p_add.add_argument("-n", "--dry-run", action="store_true", help="Show what would be staged without running git add")
    p_add.add_argument("--exclude-player-lists", action="store_true", help="Exclude ops.json, whitelist.json, banned-*.json")

    # list
    p_list = subparsers.add_parser("list", help="List all discovered configs")
    p_list.add_argument("--raw", action="store_true", help="Output raw relative paths only")
    p_list.add_argument("--json", action="store_true", help="Output in JSON format")

    # check / doctor
    subparsers.add_parser("check", help="Audit repository to ensure no worlds, JARs, or databases are tracked")

    # diff
    p_diff = subparsers.add_parser("diff", help="Show git diff for tracked configs")
    p_diff.add_argument("--staged", action="store_true", help="Show staged changes")

    # commit
    p_commit = subparsers.add_parser("commit", help="Stage and commit configs with message")
    p_commit.add_argument("-m", "--message", required=True, help="Commit message")
    p_commit.add_argument("-p", "--plugin", action="append", help="Commit only specific plugin configs")
    p_commit.add_argument("--root-only", dest="root", action="store_true", help="Commit only root configs")
    p_commit.add_argument("--dry-run", action="store_true", help="Dry run")
    p_commit.add_argument("--exclude-player-lists", action="store_true", help="Exclude player lists")

    args = parser.parse_args()

    root = find_server_root(args.root)
    c = Colors(enable=not args.no_color)

    if not args.command:
        # Default action when run without subcommands: show status
        cmd_status(args, root, c)
        return

    cmds = {
        "init": cmd_init,
        "status": cmd_status,
        "add": cmd_add,
        "list": cmd_list,
        "check": cmd_check,
        "diff": cmd_diff,
        "commit": cmd_commit,
    }

    handler = cmds.get(args.command)
    if handler:
        handler(args, root, c)
    else:
        parser.print_help()


if __name__ == "__main__":
    main()
