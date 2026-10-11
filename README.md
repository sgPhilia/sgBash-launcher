# s

A small application launcher for interactive Bash shells. Type `s -brv` instead of
`brave >/dev/null 2>&1 & disown` and get your prompt back immediately.

`s` is a **Bash function**, not an executable installed on your `PATH`. It only exists
in shells that have sourced this file. The `.bashrc` in this repository is an example
configuration meant to be sourced from, or merged into, your own `~/.bashrc`. It is
not a full replacement for it.

## Features

- Short aliases for GUI apps, kept in one Bash associative array
- `s -brv` and `s brv` both work
- Extra arguments are passed to the app unchanged, quoting included
- Apps start detached with `setsid -f`, so they don't block the terminal or close with it
- Errors go to stderr, with meaningful exit codes
- No `eval`, no changed shell options, no exported variables
- Optional fastfetch summary, shown once per interactive shell

## Requirements

| Dependency  | Arch package | Needed for |
|-------------|--------------|------------|
| Bash 4.0+   | `bash`       | Associative arrays (required) |
| `sort`      | `coreutils`  | Sorted alias list (required) |
| `setsid`    | `util-linux` | Detaching apps (optional, falls back to a background subshell) |
| `fastfetch` | `fastfetch`  | System summary in new terminals (optional) |

`bash`, `coreutils` and `util-linux` are part of `base` on Arch Linux.

The default aliases expect these programs. You only need the ones you use:

| Command            | Package |
|--------------------|---------|
| `spotify-launcher` | `spotify-launcher` |
| `code`             | `code` (Code - OSS) or AUR `visual-studio-code-bin` |
| `obsidian`         | `obsidian` |
| `firefox`          | `firefox` |
| `brave`            | AUR `brave-bin` |
| `dolphin`          | `dolphin` |
| `kate`             | `kate` |

Package names can change. Check with `pacman -Ss` or the AUR if one is missing.

## Installation (Arch Linux)

Install the dependencies and clone the repository:

```bash
sudo pacman -S --needed bash coreutils util-linux fastfetch
git clone https://github.com/<user>/<repo>.git
cd <repo>
```

Then pick **one** of the options below.

### Option A: source it from your existing `~/.bashrc` (recommended)

Your current configuration stays untouched apart from one added line.

```bash
mkdir -p ~/.config/s
cp .bashrc ~/.config/s/launcher.bash
cp ~/.bashrc ~/.bashrc.bak
printf '\n# s launcher\n[[ -f ~/.config/s/launcher.bash ]] && source ~/.config/s/launcher.bash\n' >> ~/.bashrc
```

Run the `printf` line only once. Check with `grep -n launcher.bash ~/.bashrc`.

### Option B: merge by hand

Copy the `s()` function and the Fastfetch block from `.bashrc` into your own
`~/.bashrc`. If your file already runs `fastfetch`, keep only one of them.

### Option C: use it as your whole `~/.bashrc`

Only do this on a fresh setup. This file does not include the Arch defaults
such as `alias ls='ls --color=auto'` or a custom `PS1`.

```bash
cp -i ~/.bashrc ~/.bashrc.bak
cp -i .bashrc ~/.bashrc
```

### Reload and check

```bash
source ~/.bashrc     # or open a new terminal
type -t s            # should print: function
```

## Usage

```text
s [-]ALIAS [ARGS...]    launch an app; extra ARGS are passed through
s -h | --help           show help and the alias list
s --list                list aliases and the commands they run
s --version             show the version
s source                show where s is defined and how to edit it
```

## Examples

```bash
s -brv                                  # open Brave
s brv                                   # same thing
s code                                  # open VS Code
s code ~/projects/demo                  # open a folder in VS Code
s ff "https://wiki.archlinux.org/title/Bash"
s --help
s --list
s --version
s source
```

Example output of `s source`:

```text
Defined in: /home/you/.config/s/launcher.bash
Edit:       nano /home/you/.config/s/launcher.bash
Reload:     source /home/you/.config/s/launcher.bash
Aliases are in the apps array inside the s() function.
```

The editor shown is `$VISUAL`, then `$EDITOR`, then `nano`.

## Aliases

| Alias               | Command            |
|---------------------|--------------------|
| `music`, `spotify`  | `spotify-launcher` |
| `code`, `vscode`    | `code`             |
| `obsi`, `obsidian`  | `obsidian`         |
| `ff`, `firefox`     | `firefox`          |
| `brv`, `brave`      | `brave`            |
| `dol`, `dolphin`    | `dolphin`          |
| `kate`              | `kate`             |

## Adding an alias

1. Run `s source` to find the file that defines `s`.
2. Add a line to the `apps` array inside the `s()` function:

   ```bash
           [term]="konsole"
   ```

3. Reload with `source ~/.bashrc` and check with `s --list`.

Rules:

- Alias names may contain letters, digits, `_`, `.` and `-`, and must not start with `-`.
- `source` is reserved.
- The value must be a single command name in `PATH` or an absolute path, without
  flags. To launch an app with default flags, put a small wrapper script in
  `~/.local/bin` and point the alias at it.

## Exit status

| Code  | Meaning |
|-------|---------|
| `0`   | Success, or the app was started |
| `1`   | The app could not be started, Bash is older than 4, or `s source` cannot locate its file |
| `2`   | Usage error: no arguments, unknown option or unknown alias |
| `127` | The command behind the alias is not installed |

## Fastfetch

The file runs `fastfetch` once when a new interactive terminal starts. It is skipped when:

- the shell is non-interactive (scripts, `scp`/`rsync` over ssh, `bash -c ...`)
- stdout is not a terminal
- `fastfetch` is not installed (no error is printed)
- the file is sourced again in the same shell

It uses one non-exported shell variable, `_s_fastfetch_done`, to remember that it already
ran. A new terminal or a nested `bash` shows the summary again. To turn it off, delete the
Fastfetch block, or set `_s_fastfetch_done=1` before this file is sourced.

## Keeping `bashrc.txt` in sync

`bashrc.txt` is a byte-for-byte copy of `.bashrc` that is easy to read and copy in the
browser. After editing `.bashrc`:

```bash
cp .bashrc bashrc.txt
cmp .bashrc bashrc.txt && echo "in sync"
git add .bashrc bashrc.txt
```

Always commit the two files together.

## Known limitations

- `s` is a shell function, so it is not available in scripts, cron jobs, `sudo s ...`,
  or other shells such as zsh and fish.
- Each alias maps to one executable. Default flags need a wrapper script.
- App output is discarded. `s` reports problems it can detect before launching
  (unknown alias, missing command). After the app has started, crashes are not reported.
  Run the command directly to debug.
- There is no tab completion for aliases.
- Without `setsid`, the fallback does not start a new session. In some setups the app
  may close together with the terminal.
- If the original version of this file (which used a global read-only `apps` array) is
  already loaded in your shell, open a new terminal. A read-only variable cannot be
  removed from a running shell.
- `s source` shows the path exactly as the file was sourced. If you sourced it with a
  relative path, the result is relative too.
- An existing `alias s=...` breaks the function definition. Remove it with `unalias s`.

## License

MIT. See [LICENSE](LICENSE).
