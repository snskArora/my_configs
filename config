#!/usr/bin/env bash
# lid-watch.sh: switch monitor layouts on lid close/open, but only when an
# external monitor is connected. Otherwise it does nothing.
#
# Start it from i3:
#   exec --no-startup-id bash ~/.config/i3/scripts/lid-watch.sh
#
# Needs acpid running. acpi_listen reads /run/acpid.socket, no root needed.
# Logs: journalctl -t lid-watch -f

CLOSE_LAYOUT="$HOME/.screenlayout/external_monitor.sh"     # lid close + external
OPEN_LAYOUT="$HOME/.screenlayout/dual_monitor.sh"          # lid open  + external
OPEN_ALONE_LAYOUT="$HOME/.screenlayout/internal_monitor.sh" # lid open, no external

INTERNAL_RE='^eDP'

log() { logger -t lid-watch -- "$*"; }

command -v acpi_listen >/dev/null || { log "acpi_listen not found, install acpid"; exit 1; }

# Replace an older copy of this watcher (e.g. one left over from a previous login).
pidfile="${XDG_RUNTIME_DIR:-/tmp}/lid-watch.pid"
if [[ -r $pidfile ]]; then
    old=$(<"$pidfile")
    if [[ $old != "$$" ]] && ps -o args= -p "$old" 2>/dev/null | grep -q lid-watch; then
        pkill -P "$old" 2>/dev/null
        kill "$old" 2>/dev/null
    fi
fi
echo $$ >"$pidfile"

external_connected() {
    xrandr --query | awk -v re="$INTERNAL_RE" \
        '$2 == "connected" && $1 !~ re { found = 1 } END { exit !found }'
}

on_lid() {
    local action=$1
    if external_connected; then
        case $action in
            close) log "lid close, external connected, running $CLOSE_LAYOUT"; bash "$CLOSE_LAYOUT" ;;
            open)  log "lid open, external connected, running $OPEN_LAYOUT";   bash "$OPEN_LAYOUT" ;;
        esac
    else
        case $action in
            close) log "lid close, no external monitor, doing nothing" ;;
            open)  log "lid open, no external monitor, running $OPEN_ALONE_LAYOUT"; bash "$OPEN_ALONE_LAYOUT" ;;
        esac
    fi
}

last=""
while true; do
    # acpi_listen prints lines like: button/lid LID close
    while read -r event _ action _; do
        [[ $event == button/lid ]] || continue
        [[ $action == open || $action == close ]] || continue
        [[ $action == "$last" ]] && continue   # some firmware sends the same event twice
        last=$action
        on_lid "$action"
    done < <(acpi_listen)
    # acpi_listen exits if acpid restarts, so reconnect
    sleep 2
done
