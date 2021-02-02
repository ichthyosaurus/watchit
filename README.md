<!--
SPDX-FileCopyrightText: 2021 Mirian Margiani
SPDX-License-Identifier: GPL-3.0-or-later

Example code in this file:
SPDX-FileCopyrightText: 2021 Mirian Margiani
SPDX-License-Identifier: CC0-1.0
-->

# watchit -- wait for file system events

## Dependencies

- Python 3
- [pyinotify](https://github.com/seb-m/pyinotify) (optional)

## Usage

    usage: watchit [-h] (--polling | --watcher) [--continued] [--success]
                   [--interval [INTERVAL]] [--timeout [TIMEOUT]]
                   [--no-recursive] [-x [PATTERNS [PATTERNS ...]]]
                   [-g [GLOBS [GLOBS ...]]] [--machine-readable] [--debug]
                   files [files ...]

By default, `watchit` waits until any change occurs, prints an event, and exits.
Return codes are `0=changed`, `1=added`, `2=removed`. Use a `while`-loop to wait
for changes.

Note: pyinotify is a soft dependency and will only be imported if not using
polling. Using the watcher is the default, though.

Also note that the polling mechanism is seriously limited: it only supports
watching explicitly specified files for changes. Use the file system watcher if
you want to watch directories, patterns, and want to detect added and removed
files.

    positional arguments:
        files                 files and/or directories to watch

    optional arguments:
        -h, --help            show this help message and exit
        --polling, -p         use polling instead of watching for file
                              system events; does not support watching
                              directories (default: False)
        --watcher, -w         watch for file system events using pyinotify
                              (external dependency) (default: True)
        --continued, -c       continue watching after a change has occurred
                              (default: False)
        --success, -s         return 0 on any change (instead of 0=changed,
                              1=added, 2=removed) (default: False)
        --interval [INTERVAL], -i [INTERVAL]
                              interval in seconds to use when polling (-p)
                              or time for one run when watching in
                              continuous mode (-c -w) (default: 1)
        --timeout [TIMEOUT], -I [TIMEOUT]
                              time limit in seconds for one run when
                              watching or polling in continuous mode (-c);
                              disabled with 0 (default: 0)
        --no-recursive, -R    do not watch directories recursively when
                              using the watcher (-w) (default: False)
        -x [PATTERNS [PATTERNS ...]], --regex [PATTERNS [PATTERNS ...]]
                              watch for files matching this regex (requires
                              -w) (default: ['.*'])
        -g [GLOBS [GLOBS ...]], --glob [GLOBS [GLOBS ...]]
                              watch for files matching this glob pattern
                              (requires -w) (default: [])
        --machine-readable, -m
                              print messages in a machine-readable format
                              ("<time> <type> <file>") (default: False)
        --debug, -d           print detailed info for all events (default:
                              False)

## Examples

Wait until files have been modified, then restart a program. The watcher works
recursive by default (use `-R` to disable).

```bash
while watchit . -wg '*.py'; do killall myprog.py; ./myprog.py; done
```

Wait until a file has been modified, then do something with it based on the
event.

```bash
watchit -w myfile.ext

case $? in
    0) echo "modified - copying it somewhere...";;
    1) echo "(added - event cannot happen)";;
    2) echo "removed - fetching it from somewhere...";;
esac
```

Watch for the first event below the current directory (`.`), then exit, and
handle files based on their suffices. Only watch for changes that match any of
the given glob patterns.

```bash
function build() { echo "building..."; }  # dummy

while event="$(watchit . -wsg '*.qml' '*.js' '*.svg' '*.cpp' '.h')"; do
    printf "%s\n" "$event"
    if [[ "$event" =~ \.cpp$ || "$event" =~ \.h$ ]]; then
        build
    elif [[ "$event" =~ \.qml$ || "$event" =~ \.js$ ]]; then
        qmllint qml/**/*.qml
        build
    elif [[ "$event" =~ \.svg$ ]]; then
        # render images...
        inkscape image.svg -o image.png -w 200 -h 100
    else
        refresh_files
    fi
    restart_app
done
```

Collect events matching the given patterns (`$cPATTERNS`) below the current
directory (`.`) for `$cTIMEOUT` seconds. Parse and handle events in the loop.

```bash
function do_something() { echo "doing something..."; }  # dummy
function log() { # 1: type, 2: time, 3: message
    printf "[%7s] %s: %s\n" "$@"
}
cPATTERNS=('*.cpp' '*.h' '*.md' '*.png')
cTIMEOUT=1

do_something
while events="$(watchit -wmsI "$cTIMEOUT" . -g "${cPATTERNS[@]}")"; do
    mapfile -t events_arr <<<"$events"

    for event in "${events_arr[@]}"; do
        status="${event#* }"; status="${status%% *}"
        file="${event#* * }"
        time="${event%% *}"

        # do something with $file...

        case "$status" in
            "*") log "CHANGED" "$time" "$file";;
            "+") log "ADDED" "$time" "$file";;
            "-") log "REMOVED" "$time" "$file";;
            *) log "UNKNOWN" "$time" "$file"; exit 2;;
        esac
    done

    do_something
done
```

Wait for events below the current directory, but explicitly include the current
script as file and as pattern. Restart the script when it is changed itself.

```bash
function do_something() { echo "doing something..."; }  # dummy
function fullpath() {
    [[ -z "$1" ]] && return 1
    echo "$(readlink -m "$(dirname "$1")")/$(basename "$1")"
}

function run_watch_loop() {
    while event="$(watchit -wms "$0" . -g '*.cpp' '*.h' "$(fullpath "$0")")"; do
        local file="${event#* * }"

        if [[ "$(fullpath "$file")" == "$(fullpath "$0")" ]]; then
            echo RELOADING
            return 255
        fi

        do_something
    done

    printf "%s\n" "$events"
    return 1
}

run_watch_loop; ret=$?
if (( $ret == 255 )); then
    echo "script has changed, restarting..."
    "$0"
fi
```

## Polling vs. watching

Watching (`-w`) has better performance and is much more versatile than polling
(`-p`), but it depends on [pyinotify](https://github.com/seb-m/pyinotify) which
might not be available. Polling has no external dependencies.

Polling cannot detect renamed files, and will report them as deleted instead.
This is because polling can only check explicitly listed files. It does pick up
previously deleted files in continuous mode, though.

In exiting mode (default), the watcher will report renamed files as deleted,
too. This is because renaming consists of two events, which will only be
reported in continuous mode (remove, then add; enable with `-c`).

## License

`watchit` is released under the [GNU General Public License v3 (or later)](http://www.gnu.org/licenses).

Example code is [CC0-1.0](https://spdx.org/licenses/CC0-1.0.html).
