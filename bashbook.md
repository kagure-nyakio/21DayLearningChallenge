# Bash Quide

# Part 1 — The Language

## Chapter 1: Orientation

A script is a text file of commands the shell would otherwise run one at a time interactively. Three things make it a *script* rather than just a text file:

**Shebang line** — line 1 of the file, tells the OS which interpreter to hand the file to:
```bash
#!/bin/bash
echo "hello"
```

**Executable bit** — without it, the OS won't let you run the file directly:
```bash
chmod +x script.sh
./script.sh          # works now
bash script.sh        # also works, even without the executable bit
```

**Comments** — `#` to end of line, no block comment syntax:
```bash
# this whole line is a comment
echo "hi"   # this part runs, this part doesn't →
```

Bash has no compile step. Every line runs top to bottom, in order, the moment the interpreter reaches it:
```bash
#!/bin/bash
echo "first"
echo "second"
# runs "first" then "second" — never both at once, never out of order
```
There's no hoisting, no forward declarations — if line 5 calls a function, that function has to be defined *above* line 5, not below it. We'll hit this again in Chapter 6.

## Chapter 2: Variables

```bash
name="Nyakio"       # no spaces around =
echo "$name"         # prints: Nyakio
```

Rules that trip people up coming from other languages:

**Spaces break assignment.** `name = value` is *not* assignment — bash parses it as: run a command called `name`, passing it two arguments, `=` and `value`.
```bash
name = "Nyakio"     # ERROR: "name: command not found"
name="Nyakio"        # correct
```

**Everything is a string by default**, including things that look like numbers:
```bash
count=5
echo "$count + 1"    # prints literally: 5 + 1  (no math happened)
echo $((count + 1))  # prints: 6  — arithmetic needs $(( ))
```

**Scope**: variables are global by default, even inside functions, unless declared `local`:
```bash
x=10
change_it() {
  x=20   # this changes the GLOBAL x, not a local copy
}
change_it
echo "$x"    # prints: 20
```

`readonly` locks a variable, `unset` removes it:
```bash
readonly PI=3.14
PI=4        # ERROR: PI: readonly variable

unset count
echo "$count"   # prints nothing — it's gone
```

**Quoting** is the single most important habit to build early:
```bash
file="my file.txt"

echo "$file"     # prints: my file.txt          (one thing, as intended)
echo $file       # prints: my file.txt           (looks the same here...)

ls $file          # tries to ls "my" and "file.txt" — TWO args, breaks
ls "$file"        # tries to ls "my file.txt" — ONE arg, correct
```
- `"$var"` — expands, preserves whitespace, prevents word-splitting/glob expansion. Default choice, almost always.
- `'$var'` — literal, no expansion at all: `echo '$var'` prints the text `$var`, not its value.
- Unquoted `$var` — expands *and* gets split on whitespace and glob-expanded. This is where "works on my machine, breaks on a filename with a space in it" bugs come from.

**Environment vs. shell variables — parent and child processes**

Every process has its own private variable table. When one script runs another as a separate program, the OS spins up a new process for it — that's a *child* process, and the script that launched it is the *parent*. A child does **not** automatically see the parent's variables; it only gets a copy of the ones explicitly marked for export.

```bash
color="blue"
echo "$color"                # prints: blue        (same shell, works fine)
bash -c 'echo "$color"'      # prints: nothing      (new process, no export)
```

`export` copies the variable into the environment, which *is* inherited by child processes from that point on:
```bash
export color="blue"
bash -c 'echo "$color"'      # prints: blue — now it's there
```

It's one-directional and it's a **copy**, not a live link — a child can't hand values back up to its parent:
```bash
export x="1"
bash -c 'x="2"; echo "child: $x"'   # prints: child: 2
echo "parent: $x"                     # prints: parent: 1 — unchanged
```

**Two files, parent → child in practice:**
```bash
# child.sh
#!/bin/bash
echo "child sees color: $color"
echo "child's own PID: $$"
```
```bash
# parent.sh
#!/bin/bash
export color="blue"
echo "parent's own PID: $$"
./child.sh              # runs child.sh as a NEW process
```
Run `./parent.sh` and the two PIDs will differ, and `child sees color: blue` only shows up because `color` was exported first.

**The alternative: `source` instead of running as a separate process.** If two files should really share state directly — more "one script split across files" than true parent/child — use `source` (or its shorthand `.`) instead of `./`:
```bash
# helpers.sh
color="blue"
```
```bash
# main.sh
#!/bin/bash
source ./helpers.sh    # or: . ./helpers.sh
echo "$color"            # prints: blue — no export needed
```
`source` doesn't create a new process — it runs those lines *in the current shell*, as if pasted in. Same `$$`, same variable table, no export needed in either direction. This is how the "library of functions" pattern from Chapter 6 works.

| | New process? | Needs `export`? | Child's changes visible to caller? |
|---|---|---|---|
| `./child.sh` | Yes | Yes | No |
| `source child.sh` | No | No | Yes (it's the same shell) |

Use `./script.sh` for an independent script meant to run on its own (a deploy step, a cron job, something you'd also invoke standalone). Use `source` for shared helper code split out purely for organization. `bash -x parent.sh` is a good way to watch this handoff happen live.

This distinction matters constantly in DevOps: config passed via environment (`export DB_HOST=...`) vs. a variable that only exists inside the current script.

**Special variables** worth knowing now:
```bash
echo "$?"    # exit status of the last command (0 = success)
echo "$$"    # PID of the current shell/script
echo "$0"    # the script's own name/path
echo "$#"    # number of arguments passed to the script (Ch. 7)
```

## Chapter 3: Input, Output, and Redirection

```bash
echo "hello"                  # simple output
printf "%s\n" "hello"          # when you need format control — echo's exact
                                # behavior (e.g. handling of \n, -e) varies
                                # across shells/implementations; printf doesn't
printf "%-10s %5d\n" "cpu" 42   # printf also does column alignment, padding —
                                # things echo can't do at all
```

`read` pulls a line of input into a variable:
```bash
read name
echo "you said: $name"

read -p "Enter your name: " name   # -p shows a prompt inline
read -s -p "Password: " pass        # -s hides what's typed
```

**Streams**: stdout is stream `1` (normal output), stderr is stream `2` (errors/diagnostics), stdin is stream `0` (input). By default `echo`/`printf` go to stdout.

**Redirection**:
```bash
echo "log line" > out.txt        # overwrite out.txt with this line
echo "another line" >> out.txt    # append instead of overwrite

command 2> errors.txt              # send stderr only to a file
command > all.txt 2>&1             # send BOTH stdout and stderr to all.txt
                                    # (2>&1 means "point stream 2 at wherever
                                    # stream 1 is currently going")

sort < names.txt                   # feed a file in as stdin
```

**Pipes** chain one command's stdout into the next command's stdin — this is the core idiom of shell scripting, small commands composing into a real tool:
```bash
cat access.log | grep "ERROR" | wc -l
# read the file, keep only ERROR lines, count how many
```

**`/dev/null`** is the discard bin:
```bash
command > /dev/null 2>&1   # silence all output, keep only the exit code
```

## Chapter 4: Exit Status and Conditionals

Every command returns an exit status when it finishes: `0` means success, anything `1–255` means failure — the opposite of most booleans in other languages, where 0 usually means false.
```bash
grep "ERROR" app.log
echo "$?"    # 0 if it found a match, 1 if it didn't, 2 if app.log doesn't exist
```

**Test expressions** — the building blocks of every `if`:
```bash
[[ -f "$file" ]]        # true if $file exists and is a regular file
[[ -d "$dir"  ]]        # true if $dir exists and is a directory
[[ -z "$s"    ]]        # true if $s is an empty string
[[ -n "$s"    ]]        # true if $s is a non-empty string
[[ "$a" == "$b" ]]      # true if the strings are equal
(( a == b ))             # true if the NUMBERS are equal
```
Use `[[ ]]` for string/file tests, `(( ))` for arithmetic comparisons. There's an older `[ ]` (single bracket) form too — it's more portable to non-bash shells but stricter about quoting and easier to get wrong, so stick with `[[ ]]` unless you specifically need portability.

### `if` — Basic Syntax
```bash
if [[ condition ]]; then
  # commands if condition is true
elif [[ other_condition ]]; then
  # commands if other_condition is true
else
  # commands if nothing above matched
fi
```
**Key syntax rules:**
- `if [[ condition ]]; then` — the `; then` can go on the same line as `if`, or `then` can sit on its own line below it (no semicolon needed in that case). Pick one style and stay consistent.
- `elif` — as many of these as you want, checked in order, top to bottom. First one that's true wins; the rest are skipped.
- `else` — optional, catches anything not matched above.
- `fi` — closes the block. It's `if` spelled backward, same pattern you'll see with `esac`/`case` and `done`/`do`.

```bash
if [[ -f "$file" ]]; then
  echo "found it"
elif [[ -d "$file" ]]; then
  echo "that's a directory, not a file"
else
  echo "nothing there"
fi
```

### `case` — Basic Syntax
```bash
case "$VARIABLE" in
    pattern1)
        # commands to execute if VARIABLE matches pattern1
        ;;
    pattern2|pattern3)
        # commands to execute if VARIABLE matches pattern2 OR pattern3
        ;;
    *)
        # default commands if no patterns match (wildcard)
        ;;
esac
```
**Key syntax rules:**
- `case ... in` — starts the block; whatever's in quotes is the value being matched against each pattern below.
- `pattern)` — closes each pattern option. Patterns support globbing (`*.txt`, `foo*`), not full regex.
- `|` — acts as an "OR," matching multiple patterns in one clause (`pattern2|pattern3)`).
- `;;` — terminates a clause, like `break` in a C/Java switch statement. Without it, bash would keep falling through — but by default each clause is self-contained, so you'll almost always end one with `;;`.
- `*)` — a catch-all default, same role as `default:` in other languages. Convention is to put it last, since patterns are checked top to bottom and `*` would swallow everything after it.
- `esac` — ends the block (`case` spelled backward).

`case` is the better tool than a long `if`/`elif` chain once you're matching more than two or three patterns — it reads cleaner and supports wildcards directly:
```bash
case "$1" in
  start) echo "starting...";;
  stop)  echo "stopping...";;
  *.txt) echo "a text file was passed";;
  *)     echo "unknown option: $1";;
esac
```

`&&` and `||` chain commands by exit status:
```bash
mkdir -p "$dir" && echo "created"          # only echo if mkdir succeeded
mkdir -p "$dir" || exit 1                   # bail out if mkdir failed
command1 && command2 || echo "one of them failed"
```

## Chapter 5: Loops

### `for` — Basic Syntax
```bash
for item in list; do
  # commands, using $item
done
```
**Key syntax rules:**
- `for var in list; do` — `var` takes on each value in `list` one at a time, in order. `list` can be literal words, a glob, a command substitution, or an array.
- `do` / `done` — bracket the loop body, same pairing pattern as `if`/`fi` and `case`/`esac`. `done` is not a reversed spelling of anything, it's just the closing keyword.
- The C-style form (`for ((i=0; i<5; i++))`) drops `in list` entirely and uses C-like init/condition/increment instead — closer to what you're used to from other languages.

```bash
# C-style
for ((i = 0; i < 5; i++)); do
  echo "$i"
done

# iterate a list
for item in one two three; do
  echo "$item"
done

# iterate files via glob
for f in /var/log/*.log; do
  echo "$f"
done

# iterate an array
files=(a.txt b.txt c.txt)
for f in "${files[@]}"; do
  echo "$f"
done
```

### `while` / `until` — Basic Syntax
```bash
while [[ condition ]]; do
  # runs as long as condition is TRUE
done

until [[ condition ]]; do
  # runs as long as condition is FALSE — stops once it becomes true
done
```
**Key syntax rules:**
- `while` re-checks its condition before every iteration, and stops the moment it's false. `until` is the mirror image — stops the moment its condition becomes *true*. Same `do`/`done` bracketing as `for`.
- Any command works as the condition, not just `[[ ]]` — `while curl ...; do` is valid, since `while` just checks the command's exit status (Ch. 4).

```bash
count=0
while [[ "$count" -lt 5 ]]; do
  echo "$count"
  ((count++))
done

# while read — the idiom for processing a file line by line
while IFS= read -r line; do
  echo "Line: $line"
done < input.txt
# IFS= preserves leading/trailing whitespace, -r stops backslashes
# from being treated as escape characters — use both, by default

# until — loop until a condition becomes TRUE (opposite of while)
until curl -sf http://localhost:8080/health > /dev/null; do
  echo "waiting for service..."
  sleep 2
done
```

**`while` vs. `until` — which to reach for:** they're mirror images (`until [[ x ]]` behaves the same as `while [[ ! x ]]`), so it's not about capability, it's about which one lets you write the condition without bolting a `!` onto the front of it.
- **`while`** — for an ongoing, positive condition: "keep going *while* count is still under 5," "while there's still a line to read." Most loops in practice are `while`.
- **`until`** — for waiting on something to happen: "keep going *until* the service is healthy," "until the lockfile appears." Reserve it for these polling/waiting patterns, where the `until` phrasing reads more naturally than a negated `while`:
```bash
# reads awkwardly — negation up front
while ! curl -sf http://localhost:8080/health > /dev/null; do
  sleep 2
done

# reads directly — this is what until is for
until curl -sf http://localhost:8080/health > /dev/null; do
  sleep 2
done
```

### Loop control
```bash
for n in 1 2 3 4 5; do
  if [[ "$n" -eq 3 ]]; then
    continue    # skip 3, keep going
  fi
  if [[ "$n" -eq 5 ]]; then
    break        # stop the loop entirely
  fi
  echo "$n"
done
```

The one thing worth making explicit: loops are how you turn one-off commands into batch operations, which is most of what a real DevOps script actually is — "do this thing to every file/host/user/log line."

## Chapter 6: Functions

```bash
my_func() {
  local x="$1"          # $1 here is the function's OWN first argument,
  echo "got $x"          # separate from the script's $1
  return 0
}

my_func "hello"    # prints: got hello
```

- Defined before use — no hoisting, same as Chapter 1's rule for the whole script.
- `local` scopes a variable to the function. Without it, everything is global by default, which is a common source of bugs when a function accidentally overwrites a variable the caller still needs:
```bash
count=100
bump() {
  count=1        # no "local" — this stomps the outer $count!
}
bump
echo "$count"    # prints: 1, not 100 — probably not what you wanted
```
- `return` sets the exit status only (`0`–`255`), it doesn't hand back arbitrary data like `return` in most languages. To get actual data out of a function, `echo` it and capture the output:
```bash
get_hostname() {
  echo "web-01"
}
host=$(get_hostname)
echo "$host"    # prints: web-01
```
- A "library" script is just a file of function definitions you pull into another script with `source`:
```bash
source ./helpers.sh    # or: . ./helpers.sh
```

## Chapter 7: Positional Parameters and Arguments

```bash
#!/bin/bash
echo "script name: $0"
echo "first arg: $1"
echo "second arg: $2"
echo "arg count: $#"
echo "all args, each quoted separately: $@"
```
Run as `./script.sh foo bar` → prints `foo`, `bar`, `2`, `foo bar`.

`"$@"` vs `"$*"` matters once arguments have spaces in them. First, a quick note on `set --`, used below to fake having command-line arguments without actually invoking the script: `set` can reassign the script's positional parameters directly, and `--` just means "everything after this is a plain value, not a `set` option." So `set -- "hello world" "second arg"` makes `$1`, `$2`, `$@` behave exactly as if the script had been run with those two arguments:
```bash
set -- "hello world" "second arg"
for a in "$@"; do echo "[$a]"; done
# [hello world]
# [second arg]     ← two items, as expected

for a in "$*"; do echo "[$a]"; done
# [hello world second arg]   ← ONE item — almost never what you want
```
`set --` is also useful for reprocessing arguments after transforming them mid-script — e.g. splitting a string into fields and then using `set -- "${fields[@]}"` to load them as `$1`, `$2`, ... for the rest of the script to use.

`shift` drops `$1` and renumbers the rest down — this is how you loop over an unknown number of arguments:
```bash
while [[ $# -gt 0 ]]; do
  echo "processing: $1"
  shift
done
```

`getopts` handles real flags (short options only — `--long-flag` needs manual parsing):
```bash
while getopts "vf:h" opt; do
  case "$opt" in
    v) verbose=true ;;
    f) file="$OPTARG" ;;     # f: means -f takes an argument
    h) echo "usage: script.sh [-v] [-f file]"; exit 0 ;;
  esac
done
```
Worth learning once you're writing anything you'll run more than a couple of times — positional-only args get unreadable fast (`myscript.sh 1 0 file.txt "" 1` — what do those even mean without reading the source?).

## Chapter 8: Strings and Numbers

**Parameter expansion** does a lot of what you'd otherwise reach for `sed`/`cut` to do, faster and without spawning a subprocess:
```bash
name=""
echo "${name:-default}"     # prints: default (name itself is untouched)
echo "${name:=default}"     # prints: default (and now ALSO assigns name=default)

file="report.txt"
echo "${#file}"              # prints: 11 (length)
echo "${file:0:6}"           # prints: report (substring: offset 0, length 6)

path="/home/user/notes.txt"
echo "${path%.txt}"          # prints: /home/user/notes   (strip .txt from end)
echo "${path##*/}"           # prints: notes.txt           (strip dir from front)
echo "${path%/*}"            # prints: /home/user           (strip filename from end)

greeting="hello world"
echo "${greeting/world/there}"    # prints: hello there  (replace first match)
echo "${greeting^^}"               # prints: HELLO WORLD  (uppercase, bash 4+)
echo "${greeting,,}"               # prints: hello world  (lowercase)
```
The `#`/`%` pair above is exactly how you strip extensions or directories without calling `basename`/`dirname` as separate processes.

**Arithmetic** — bash only does integers natively:
```bash
x=5
y=3
echo $((x + y))       # prints: 8
((x++))                 # increment in place
result=$((x * y))
```
For floating point, shell out to `bc` or `awk`:
```bash
echo "scale=2; 10/3" | bc     # prints: 3.33
```

**Text processing tools** aren't part of bash's syntax, but they're the vocabulary you'll pair bash with constantly:
```bash
grep "ERROR" app.log            # find matching lines
cut -d: -f1 /etc/passwd          # slice out column 1, "," as delimiter
sed 's/foo/bar/' file.txt        # substitute foo → bar
awk '{print $1}' file.txt        # print column 1 (whitespace-delimited)
sort file.txt | uniq -c           # sort, then count duplicate lines
tr 'a-z' 'A-Z' < file.txt         # translate lowercase to uppercase
```
A large fraction of "shell scripting" in practice is bash gluing these together via pipes, not bash doing the text manipulation itself.

## Chapter 9: Arrays

```bash
arr=(one two three)
arr[3]="four"
echo "${arr[0]}"        # prints: one
echo "${arr[@]}"         # prints: one two three four
echo "${#arr[@]}"         # prints: 4 (length)

for item in "${arr[@]}"; do
  echo "$item"
done
```

Associative arrays (bash 4+, key/value pairs):
```bash
declare -A map
map[name]="Nyakio"
map[role]="founder"

echo "${map[name]}"          # prints: Nyakio

for key in "${!map[@]}"; do   # ! gives you the KEYS, not the values
  echo "$key -> ${map[$key]}"
done
```
Arrays are how you hold a list without spawning a subprocess for it — command output split into lines, a list of hosts to loop over, a set of flags collected during argument parsing.

## Chapter 10: Errors, Exit Codes, and Cleanup

This is the chapter that separates "script that works when I run it by hand" from "script that's safe in a pipeline or a cron job."

```bash
set -e            # exit immediately if any command fails
set -u            # error out on any unset variable, instead of silently
                    # expanding it to an empty string
set -o pipefail   # a pipeline's exit status is the LAST NON-ZERO status in
                    # it, not just the last command's

set -euo pipefail   # all three together — a reasonable default at the top
                      # of most real scripts
```
Example of what `pipefail` actually fixes:
```bash
false | true
echo "$?"                # prints: 0 — without pipefail, this LOOKS successful

set -o pipefail
false | true
echo "$?"                # prints: 1 — now the failure is visible
```

`trap` runs cleanup code no matter how the script exits — success, error, or Ctrl+C:
```bash
tmpfile=$(mktemp)
trap 'rm -f "$tmpfile"' EXIT

echo "working with $tmpfile"
# ... whatever happens next, even a crash, the tmpfile still gets removed
```

`exit N` sets an explicit exit code so callers (CI systems, other scripts) can branch on what happened:
```bash
if [[ ! -f "$config" ]]; then
  echo "config missing" >&2
  exit 1
fi
exit 0
```

## Chapter 11: Files, Processes, and the System

Not new syntax — just naming the tools you'll lean on constantly.

```bash
[[ -f "$f" ]] && echo "regular file"
[[ -d "$f" ]] && echo "directory"
[[ -r "$f" ]] && echo "readable"
[[ -w "$f" ]] && echo "writable"
[[ -x "$f" ]] && echo "executable"

find /var/log -name "*.log" -mtime +7          # files older than 7 days
find /var/log -name "*.log" -mtime +7 -delete    # ...and delete them
find . -name "*.tmp" | xargs rm                    # or pipe into xargs
```

Process basics:
```bash
echo "$$"          # this script's own PID
long_task &          # run in the background
echo "$!"            # PID of that background job
wait                  # block until background jobs finish
```

Bash can check a port without needing `nc`, using its built-in `/dev/tcp`:
```bash
if timeout 2 bash -c "echo > /dev/tcp/localhost/8080" 2>/dev/null; then
  echo "port 8080 is open"
fi
```

A simple lock-file pattern to stop a cron job from overlapping itself:
```bash
lockfile="/tmp/myscript.lock"
if [[ -e "$lockfile" ]]; then
  echo "already running"
  exit 1
fi
touch "$lockfile"
trap 'rm -f "$lockfile"' EXIT
```

## Chapter 12: Debugging

```bash
bash -x script.sh    # prints every command as it runs, WITH expansions
                        # resolved — the single most useful bash debugging tool

bash -n script.sh    # syntax check only, doesn't actually run anything
```
Inside a script, you can turn tracing on/off for just one section:
```bash
set -x
tricky_command "$var"
set +x
```
`set -v` echoes the raw source lines before execution (as written, not expanded) — less useful than `-x` for chasing expansion bugs, but occasionally handy.

**ShellCheck** (external tool, not built into bash) does static analysis and catches quoting bugs, unsafe patterns, and portability issues before you ever run the script. Worth installing early:
```bash
shellcheck script.sh
```

---

# Part 2 — Projects (Descriptions Only)

Ordered roughly by which chapters they need. Code comes in the next file — these are specs to work from, the way you'd read a ticket.

### 1. Password Generator
Generate a random password of a given length using allowed character sets. Should accept a `-l` length flag and a `-s` "include symbols" flag, and print a usage message if called with `-h` or bad input. Stretch: generate multiple passwords in one call via a `-n count` flag.

### 2. Number-Guessing Game
Script picks a random number in a range, player guesses, script says higher/lower, tracks attempt count, and congratulates on a correct guess. Good first place to actually feel a `while` loop combined with conditionals doing real work instead of toy output.

### 3. Countdown / Pomodoro Timer
Counts down from a given number of minutes, updating the terminal each second, and does something (bell, message) when it hits zero. Should handle Ctrl+C gracefully via `trap` rather than leaving the terminal in a weird state.

### 4. File Organizer
Given a messy directory (think Downloads), sorts files into subfolders by extension. Should support a `--dry-run` flag that prints what it *would* do without moving anything — a habit worth building early, since "script that moves your files" is exactly the kind of thing you want to test safely first.

### 5. Duplicate File Finder
Walks a directory tree, hashes each file, and reports groups of files with identical hashes. Introduces the idea of using an associative array as a lookup table (hash → first file seen with that hash) to detect the second and later occurrences.

### 6. Log Analyzer
Given a web server or application log, report: count of each HTTP status code (or log level), top N most frequent source IPs (or error messages), and requests per hour. This is the project that most resembles real DevOps work — heavy on `grep`/`awk`/`sort`/`uniq` piped together, light on bash-native logic. Worth revisiting against a real Nextack client log once comfortable.

### 7. Bulk Renamer
Renames a batch of files based on a pattern (e.g., strip a prefix, replace spaces with underscores, add a sequence number). Must support `--dry-run` for the same reason as the file organizer. Good place to practice parameter expansion (Ch. 8) instead of shelling out to `sed` for simple renames.

### 8. Backup Script with Rotation
Tars up a source directory into a timestamped archive, then deletes archives older than N days (or keeps only the most recent N). This is close to verbatim what you'll run in production — worth building it properly (error handling, cleanup on failure) rather than as a toy.

### 9. Health-Check / Retry Wrapper
Given a URL or host:port, polls it until it responds successfully or a max-attempt count is reached, with increasing delay between attempts (backoff). Exits non-zero on final failure so it can gate a deploy step. This is the shape of almost every "wait for the service to be ready" step in a CI/CD pipeline.

### 10. Environment/Secrets Sanity Checker
Given a list of required environment variable names, checks that each is set and non-empty, and reports exactly which ones are missing before a deploy proceeds. Small script, but it's the kind of guardrail that prevents an entire class of "deploy failed halfway through because of a missing var" incidents.

### 11. Git Repo Health Checker
Given a list of local repo paths (or all subdirectories of a folder), reports for each: uncommitted changes, unpushed commits, and current branch. Useful as a real tool, and forces looping over a collection while shelling out to another program (`git`) and parsing its output.

### 12. Deploy Script Skeleton
A parameterized script (environment name as an argument) that builds an image, tags it with the current git commit, pushes it, and updates a running deployment, with `trap`-based error reporting if any step fails. This is deliberately the "shape" of a real deploy script — building it teaches the error-handling discipline (Ch. 10) more than anything else on this list.

### 13. Simple CI-Style Task Runner
Define steps (lint, test, build) as functions, then run them in sequence from an array, stopping and reporting clearly on the first failure. A small step toward understanding what tools like Makefiles or CI YAML are automating under the hood.

### 14. Cert/Domain Expiry Checker
Given one or more domain names as arguments, reports each certificate's expiry date, and optionally flags any expiring within N days. Realistic on-call tooling, and a good excuse to practice looping over `"$@"` cleanly.

### 15. systemd Service Watcher
Runs indefinitely (or via cron), checks whether a named service is active, and restarts it (with a logged timestamp) if it isn't. Introduces the idea of a script meant to run continuously rather than once — different failure/logging considerations than a one-shot script.

### 16. JSON-Driven Task Runner (stretch)
Reads a `tasks.json` describing a sequence of named steps and their shell commands, executes them in order, and logs pass/fail per step. This is the "graduate project" — it's basically a tiny task runner, and building it surfaces most of what's in this book at once.

---

## Fun / Just-Because Projects

Not devops-useful, just good for building comfort with the language without the pressure of "this needs to be production-quality":

### 17. ASCII Conway's Game of Life
Simulate the classic cellular automaton in a 2D array, redrawing the grid in the terminal each generation. Forces you to think in nested loops and array indexing at the same time — a genuinely good workout, not just a novelty.

### 18. Terminal Fireworks / Screensaver
Randomly place and animate simple ASCII bursts or patterns across the terminal on a timer, clearing and redrawing each frame. Mostly an excuse to play with `sleep`, cursor control, and `$RANDOM`.

### 19. Mad Libs Generator
Prompt for a handful of words (noun, verb, adjective...) and slot them into a stored story template, printing the ridiculous result. Good, low-stakes `read` and string-substitution practice.

### 20. Terminal-Based Slot Machine / Dice Game
Spin three random symbols (or roll dice), check for matches, track a running "score" across multiple plays. Combines `$RANDOM`, arrays, and loops in a way that actually feels like a program rather than a script.

### 21. Fortune Cookie / Quote of the Day
Pick a random line out of a text file of quotes and print it, maybe styled with a simple ASCII border. Tiny, but a nice first script to stick in your shell's startup file just to see it run automatically.

### 22. Text-Based Adventure (mini)
A handful of "rooms" as functions, each printing a description and reading a choice (`go north`, `look`, `take key`) via `case`, branching to the next room function. This is genuinely how the top-down design chapter in *The Linux Command Line* frames a first real script — it just happens to be a game.

### 23. Matrix Rain Effect
Print streams of random characters scrolling down the terminal in green, looping until interrupted. Almost pure novelty, but a fun `trap`-on-Ctrl+C exercise, and a crowd-pleaser to show off once it works.

---

## Exercises for Common Tasks (small, single-sitting)

These aren't full projects — they're the kind of five-minute drills worth doing directly in a terminal to cement one concept each:

- Write a one-liner that prints every `.log` file in a directory older than 7 days.
- Write a function that takes a filename and prints "yes"/"no" for whether it's readable, writable, and executable, using three separate file tests.
- Given a colon-separated string (like `$PATH`), split it into an array and print each entry on its own line.
- Write a loop that pings a list of hosts (from a file, one per line) and prints which ones are unreachable.
- Take any of your existing scripts and add `set -euo pipefail` plus a `trap` cleanup — see what breaks, and fix it.
- Write a `getopts`-based argument parser for a script with three flags: `-v` (verbose, boolean), `-o file` (output path), `-h` (help, prints usage and exits).
- Given a CSV file, use `awk` to print only rows where column 3 is greater than a given threshold.


