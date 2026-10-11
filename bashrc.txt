# shellcheck shell=bash
#
# s - a small application launcher for interactive Bash shells
#
# Usage:   s [-]ALIAS [ARGS...]      e.g.  s -brv   s code ~/project
#          s -h | --help | --list | --version | source
#
# Requires: bash >= 4 (associative arrays), sort (coreutils)
# Optional: setsid (util-linux) to fully detach apps from the terminal,
#           fastfetch for the system summary shown in new terminals
#
# This file is meant to be sourced from ~/.bashrc (or merged into it).
# It defines one function, s, and runs fastfetch once per interactive shell.
#
# SPDX-License-Identifier: MIT

# ---------------------------------------------------------------------------
# Launcher
# ---------------------------------------------------------------------------

s() {
    local -r version="1.0.0"

    if (( BASH_VERSINFO[0] < 4 )); then
        printf 's: Bash 4 or newer is required (this is %s)\n' "$BASH_VERSION" >&2
        return 1
    fi

    # Alias -> executable. Add new apps here, one per line.
    # The array lives inside the function, so it is local to each call,
    # does not touch the global namespace and picks up edits on re-source.
    declare -Ar apps=(
        [music]="spotify-launcher"
        [spotify]="spotify-launcher"

        [code]="code"
        [vscode]="code"

        [obsi]="obsidian"
        [obsidian]="obsidian"

        [ff]="firefox"
        [firefox]="firefox"

        [brv]="brave"
        [brave]="brave"

        [dol]="dolphin"
        [dolphin]="dolphin"

        [kate]="kate"
    ) || return 1

    if (( $# == 0 )); then
        printf '%s\n' \
            'Usage: s [-]ALIAS [ARGS...]' \
            "Run 's --help' for options or 's --list' for aliases." >&2
        return 2
    fi

    case $1 in
        -h|--help)
            printf '%s\n' \
                'Usage: s [-]ALIAS [ARGS...]' \
                '' \
                'Launch an application in the background without blocking the terminal.' \
                'Extra ARGS are passed to the application unchanged.' \
                '' \
                'Options:' \
                '  -h, --help     Show this help' \
                '      --list     List aliases and the commands they run' \
                '      --version  Show the version' \
                '  source         Show where s is defined and how to edit it' \
                '' \
                'Examples:' \
                '  s -brv' \
                '  s code ~/projects/demo' \
                '  s ff https://archlinux.org' \
                '' \
                'Aliases:'
            ;&  # fall through: help ends with the alias list
        --list)
            local name
            while IFS= read -r name; do
                printf '  %-10s %s\n' "$name" "${apps[$name]}"
            done < <(printf '%s\n' "${!apps[@]}" | LC_ALL=C sort)
            return 0
            ;;
        --version)
            printf 's %s\n' "$version"
            return 0
            ;;
        source|-source)
            local file=${BASH_SOURCE[0]-}
            if [[ -z $file || ! -f $file ]]; then
                printf 's: cannot tell which file defines s (usually ~/.bashrc)\n' >&2
                return 1
            fi
            printf 'Defined in: %s\n' "$file"
            printf 'Edit:       %s %q\n' "${VISUAL:-${EDITOR:-nano}}" "$file"
            printf 'Reload:     source %q\n' "$file"
            printf 'Aliases are in the apps array inside the s() function.\n'
            return 0
            ;;
        --*)
            printf "s: unknown option '%s'\n" "$1" >&2
            printf "Run 's --help' for usage.\n" >&2
            return 2
            ;;
    esac

    local key=${1#-} cmd=
    shift

    # Only look up plain names; this also rejects '' and odd characters.
    if [[ -n $key && $key != *[![:alnum:]_.-]* ]]; then
        cmd=${apps[$key]-}
    fi

    if [[ -z $cmd ]]; then
        printf "s: unknown alias '%s'\n" "$key" >&2
        printf "Run 's --list' to see available aliases.\n" >&2
        return 2
    fi

    if ! type -P "$cmd" >/dev/null; then
        printf "s: '%s' is not installed or not in PATH\n" "$cmd" >&2
        return 127
    fi

    # Detach the app from this terminal: no blocked prompt, no job-control
    # messages, and it keeps running after the terminal closes.
    if type -P setsid >/dev/null; then
        if ! setsid -f "$cmd" "$@" </dev/null >/dev/null 2>&1; then
            printf "s: failed to start '%s'\n" "$cmd" >&2
            return 1
        fi
    else
        # Fallback when util-linux is missing: a backgrounded subshell keeps
        # the app out of this shell's job table (less isolated than setsid).
        ( "$cmd" "$@" </dev/null >/dev/null 2>&1 & )
    fi
    return 0
}

# ---------------------------------------------------------------------------
# Fastfetch
# ---------------------------------------------------------------------------
# Show the system summary once per interactive shell. Skipped when:
#   - the shell is not interactive (scripts, scp/rsync over ssh, ...)
#   - stdout is not a terminal
#   - fastfetch is not installed
#   - this file is sourced again in the same shell
# The marker variable is not exported, so every new terminal shows it once.

if [[ $- == *i* && -t 1 && -z ${_s_fastfetch_done-} ]] && type -P fastfetch >/dev/null; then
    _s_fastfetch_done=1
    fastfetch
fi
