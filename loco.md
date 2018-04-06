```shell mdsh
@module loco.md
@main loco_main

@require pjeby/license @comment LICENSE
@require bashup/realpaths cat "$BASHER_INSTALL_BIN/realpaths"
```

```shell
# For documentation, see https://github.com/bashup/loco

set -euo pipefail

fn_exists() { declare -F -- "$1"; } >/dev/null
fn_copy()   { REPLY="$(declare -f $1)"; eval "$2 ${REPLY#$1}"; }
findup()    { walkup "${1:-$PWD}" reply_if_exists "${@:2}"; }

reply_if_exists() {
    local pat dir=$1 IFS= ; shift
    for pat; do
        for REPLY in ${dir%/}/$pat; do [[ -f "$REPLY" ]] && return 0; done
    done
    return 1
}

walkup() {
    realpath.absolute "$1"
    until set -- "$REPLY" "${@:2}"; "$2" "$1" "${@:3}"; do
        [[ "$1" != "/" ]] || return 1; realpath.dirname "$1"
    done
}

_loco_usage() { loco_error "Usage: $LOCO_COMMAND command args..."; }
_loco_error() { echo "$@" >&2; exit 64; }
_loco_cmd() { REPLY="$LOCO_NAME.$1"; }
_loco_exec() { loco_error "Unrecognized command: $1"; }
_loco_exists() { type -t "$1"; } >/dev/null

_loco_do() {
    [[ "${1-}" ]] || loco_usage   # No command given, exit w/usage
    REPLY=""; loco_cmd "$1"; local cmd="$REPLY"
    [[ "$cmd" ]] || loco_usage   # Unrecognized command, exit w/usage

    if loco_exists "$cmd"; then
        # Command, alias, function, or builtin exists
        shift; "$cmd" "$@"
    else
        # Invoke the default command interpreter
        loco_exec "$@"
    fi
}

_loco_findproject() {
    findup "$LOCO_PWD" "${LOCO_FILE[@]}" && LOCO_PROJECT=$REPLY ||
        loco_error "Can't find $LOCO_FILE here";
}
_loco_preconfig() { true; }
_loco_postconfig() { true; }
_loco_findroot() { realpath.dirname "$LOCO_PROJECT"; LOCO_ROOT=$REPLY; }
_loco_loadproject() { cd "$LOCO_ROOT";  $LOCO_LOAD "$1"; }
_loco_site_config() { source "$1"; }
_loco_user_config() { source "$1"; }


# Find our configuration, exposing relevant paths and defaults

_loco_config() {
    LOCO_ARGS=("$@")
    loco_preconfig "$@"
    ${LOCO_COMMAND:+:} realpath.basename "$LOCO_SCRIPT"; LOCO_COMMAND="${LOCO_COMMAND-$REPLY}"
    LOCO_NAME="${LOCO_NAME-${LOCO_COMMAND}}"
    LOCO_PWD="${LOCO_PWD-$PWD}"

    LOCO_SITE_CONFIG="${LOCO_SITE_CONFIG-/etc/$LOCO_NAME/config}"
    [ -f "$LOCO_SITE_CONFIG" ] && loco_site_config "$LOCO_SITE_CONFIG"
    LOCO_RC="${LOCO_RC-.${LOCO_NAME}rc}"
    LOCO_USER_CONFIG="${LOCO_USER_CONFIG-$HOME/$LOCO_RC}"
    [ -f "$LOCO_USER_CONFIG" ] && loco_user_config "$LOCO_USER_CONFIG"

    [[ ${LOCO_FILE-} ]] || LOCO_FILE=(".$LOCO_NAME")
    LOCO_LOAD="${LOCO_LOAD-source}"
    loco_postconfig "$@"
}

_loco_main() {
    loco_config "$@"
    fn_exists $LOCO_NAME || eval "$LOCO_NAME() { loco_do \"\$@\"; }"
    ${LOCO_PROJECT:+:} loco_findproject "$@"
    ${LOCO_ROOT:+:}    loco_findroot "$@"
    loco_loadproject "$LOCO_PROJECT"
    loco_do "$@"
}

# Initialize default function implementations
for f in $(compgen -A function _loco_); do
    fn_exists "${f#_}" || fn_copy "$f" "${f#_}"
done

# Clear all LOCO_*  variables before beginning
for lv in ${!LOCO_@}; do unset $lv; done

LOCO_SCRIPT=$0
```

