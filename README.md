# linuxxivsolver

#!/usr/bin/env bash
#
# act-plugin-sync.sh
#
# Purpose:
#   Sync updated ACT plugins (DLLs and/or plugin folders) into the ACT Plugins directory
#   used by FFXIV + ACT under Wine / XLCore.
#
# What it does:
#   - Takes a "staging" directory as input
#   - Copies any *.dll / *.dll.config / *.exe.config files into ACT Plugins
#   - Copies known plugin folders (OverlayPlugin, cactbot)
#   - Backs up anything it overwrites (timestamped)
#
# Usage:
#   ./act-plugin-sync.sh /path/to/staging_dir
#
# Example workflow:
#   mkdir -p ~/act-updates
#   cp ~/Downloads/FFXIV_ACT_Plugin.dll ~/act-updates/
#   unzip OverlayPlugin.zip -d ~/act-updates
#   unzip cactbot-0.36.0.zip -d ~/act-updates
#   ./act-plugin-sync.sh ~/act-updates
#
# After running:
#   Launch ACT with:
#     bash ~/.local/share/ffxiv-tools/ffxiv-run-act.sh
#

set -euo pipefail

STAGE="${1:-}"
if [[ -z "$STAGE" ]]; then
  echo "Usage: $0 /path/to/staging_dir"
  exit 1
fi
if [[ ! -d "$STAGE" ]]; then
  echo "ERROR: staging_dir not found: $STAGE"
  exit 1
fi

# ACT Plugins directory (XLCore / Wine)
PLUGINS="$HOME/.xlcore/wineprefix/drive_c/users/cory/AppData/Roaming/Advanced Combat Tracker/Plugins"

# Backup location
BACKUP_ROOT="$HOME/Downloads/act-plugin-backups"
TS="$(date +%Y%m%d-%H%M%S)"
BACKUP_DIR="$BACKUP_ROOT/$TS"

mkdir -p "$PLUGINS" "$BACKUP_DIR"

echo "== ACT plugin sync =="
echo "Staging directory : $STAGE"
echo "Target Plugins    : $PLUGINS"
echo "Backup directory  : $BACKUP_DIR"
echo

backup_path() {
  local target="$1"
  if [[ -e "$target" ]]; then
    local rel="${target#$PLUGINS/}"
    local dest="$BACKUP_DIR/$rel"
    mkdir -p "$(dirname "$dest")"
    cp -a "$target" "$dest"
    echo "  backed up: $rel"
  fi
}

copy_file() {
  local src="$1"
  local base
  base="$(basename "$src")"
  local dst="$PLUGINS/$base"
  backup_path "$dst"
  cp -a "$src" "$dst"
  echo "  installed file: $base"
}

copy_dir() {
  local src="$1"
  local base
  base="$(basename "$src")"
  local dst="$PLUGINS/$base"
  backup_path "$dst"
  rm -rf "$dst"
  cp -a "$src" "$dst"
  echo "  installed dir : $base/"
}

shopt -s nullglob

# 1) Copy DLLs / config files
mapfile -t FILES < <(
  find "$STAGE" -type f \
    \( -iname '*.dll' -o -iname '*.dll.config' -o -iname '*.exe.config' \)
)

if (( ${#FILES[@]} > 0 )); then
  echo "-- Files --"
  for f in "${FILES[@]}"; do
    copy_file "$f"
  done
  echo
fi

# 2) Copy known plugin folders (add names here if needed)
PLUGIN_DIR_NAMES=(
  "OverlayPlugin"
  "cactbot"
)

echo "-- Plugin folders --"
found_any=0
for name in "${PLUGIN_DIR_NAMES[@]}"; do
  while IFS= read -r -d '' d; do
    found_any=1
    copy_dir "$d"
  done < <(find "$STAGE" -type d -name "$name" -print0)
done

if (( found_any == 0 )); then
  echo "  (none found)"
fi

echo
echo "Done."
echo "Launch ACT with:"
echo "  bash ~/.local/share/ffxiv-tools/ffxiv-run-act.sh"
