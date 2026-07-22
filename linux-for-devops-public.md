# Linux for DevOps — A Practical Field Guide

> **Learning in Public** — This guide was built as part of my DevOps learning sprint. It's a living reference, not a tutorial. The goal is to have every command I actually reach for in one place, with enough context to understand *why* — not just *what*.
>
---


## Table of Contents

1. [Mental Model — What Linux IS](#part-1-mental-model--what-linux-is)
2. [The Filesystem — The Map You Need](#part-2-the-filesystem--the-map-you-need)
3. [Navigation — Moving Around](#part-3-navigation--moving-around)
4. [Exploring Files](#part-4-exploring-files)
5. [Files — Create, Move, Delete](#part-5-files--create-move-delete)
6. [Working With Commands](#part-6-working-with-commands)
7. [Redirection — The Most Important Concept](#part-7-redirection--the-most-important-concept)
8. [Shell Expansions — What the Shell Does Before Running](#part-8-shell-expansions--what-the-shell-does-before-running)
9. [Advanced Keyboard Tricks](#part-9-advanced-keyboard-tricks)
10. [History](#part-10-history)
11. [Users and Permissions](#part-11-users-and-permissions)
12. [Processes](#part-12-processes)
13. [Text Manipulation — Daily Tools](#part-13-text-manipulation--daily-tools)
14. [Finding Things](#part-14-finding-things)
15. [Networking](#part-15-networking)
16. [Archiving and Compression](#part-16-archiving-and-compression)
17. [Package Management](#part-17-package-management)
18. [Environment Variables](#part-18-environment-variables)
19. [Scheduling](#part-19-scheduling)
20. [System Information](#part-20-system-information)
21. [File Editing from the Terminal](#part-21-file-editing-from-the-terminal)
22. [Shell — bash vs zsh in DevOps](#part-22-shell--bash-vs-zsh-in-devops)
23. [Quick Reference Card](#quick-reference-card)

---

## Part 1: Mental Model — What Linux IS

Linux is a **kernel** — the brain that talks to hardware. What you actually use is a **distribution** (distro): the kernel + tools + package manager bundled together.

For DevOps work, you'll mostly live on:
- **Ubuntu/Debian** — most cloud servers, Docker images, CI runners
- **RHEL/CentOS/Fedora** — enterprise environments, AWS EC2 defaults

The shell you'll use 99% of the time: **bash**.

The dollar sign `$` in examples = "type what comes after this." Don't type the `$`.

---

## Part 2: The Filesystem — The Map You Need

Everything in Linux is a file. Devices, processes, config — all files.

```
/                   ← root, the trunk of the tree
├── bin/            ← essential binaries (ls, cp, cat, rm)
├── sbin/           ← system admin binaries (fsck, ip)
├── usr/
│   ├── bin/        ← user programs (git, python, go)
│   ├── sbin/       ← more system admin programs
│   ├── local/bin/  ← things YOU install (your Go binaries land here)
│   └── share/doc/  ← documentation for installed packages
├── etc/            ← ALL system-wide config files (nginx.conf, ssh, cron)
├── var/
│   ├── log/        ← log files (tail -f this constantly)
│   └── lib/        ← persistent app data
├── tmp/            ← temporary files, wiped on reboot
├── home/
│   └── username/   ← your home directory (~)
├── root/           ← root user's home
├── proc/           ← virtual FS, live kernel/process data (not on disk)
├── dev/            ← device files (disks, terminals)
├── boot/           ← kernel + bootloader
├── lib/            ← shared library files used by core programs
├── opt/            ← optional/commercial software installs
└── srv/            ← service data (web roots, FTP)
```

**DevOps files you'll touch constantly:**
- `/etc/nginx/` — nginx config
- `/etc/systemd/system/` — service unit files
- `/var/log/` — application and system logs
- `/etc/hosts` — local DNS overrides (consulted before DNS)
- `/etc/crontab` — scheduled tasks
- `~/.ssh/` — SSH keys and config
- `/etc/passwd` — user accounts (not passwords)
- `/etc/fstab` — filesystem mount table

---

## Part 3: Navigation — Moving Around

```bash
pwd                     # where am I right now? (print working directory)
ls                      # list files in current directory
ls -l                   # long format: permissions, owner, size, date
ls -la                  # long format including hidden files (starting with .)
ls -lh                  # human-readable sizes (KB, MB)
ls -lt                  # sort by modification time
ls -ltr                 # sort by time, oldest first (reversed)
ls -lS                  # sort by file size
ls -F                   # append / to dirs, * to executables, @ to symlinks
ls -d */                # list only directories
ls /usr /bin            # list multiple directories at once
ls -R                   # recursive listing
```

### cd Shortcuts — The Ones People Forget

```bash
cd /etc             # go to /etc (absolute path — always starts with /)
cd nginx            # go into nginx/ from current dir (relative path)
cd ..               # go up one level
cd ../..            # go up two levels
cd ~                # go home
cd                  # also goes home (same as cd ~)
cd -                # go back to where you JUST were (toggle between two dirs)
cd ~bob             # go to bob's home directory
```

**Absolute vs Relative paths:**
- Absolute always starts with `/` → `/home/username/projects`
- Relative starts from where you are → `projects/myapp`
- `.` = current directory, `..` = parent directory

### pushd / popd — Directory Stack

When you need to bounce between more than two directories:

```bash
pushd /etc/nginx        # go to /etc/nginx AND save current dir on stack
pushd /var/log          # go to /var/log AND save /etc/nginx on stack
dirs                    # show the stack: /var/log /etc/nginx ~/
popd                    # go back to /etc/nginx (pop from stack)
popd                    # go back to wherever you started
```

---

## Part 4: Exploring Files

### Viewing File Contents

```bash
cat file.txt            # print whole file
cat -n file.txt         # print with line numbers
cat -A file.txt         # show non-printing chars (^I=tab, $=line end) — debug config files
tac file.txt            # print file in REVERSE line order
less file.txt           # page through it (space=next page, b=back, q=quit, /=search)
head file.txt           # first 10 lines (default)
head -n 20 file.txt     # first 20 lines
tail file.txt           # last 10 lines
tail -n 20 file.txt     # last 20 lines
tail -f /var/log/syslog # FOLLOW a log in real time ← use this constantly
```

**less keyboard commands** (you will use these daily):
```
space / page down   → next page
b / page up         → previous page
G                   → jump to END of file
g / 1G              → jump to BEGINNING
/pattern            → search forward for pattern
n                   → next match
N                   → previous match
q                   → quit
```

### Determining File Types

```bash
file picture.jpg        # → JPEG image data, JFIF standard...
file /bin/ls            # → ELF 64-bit LSB executable...
file /etc/passwd        # → ASCII text
file unknown_file       # tell me what this actually is
```

Linux doesn't care about file extensions. `file` inspects the actual content.

---

## Part 5: Files — Create, Move, Delete

### Creating Files

```bash
touch file.txt              # create empty file (or update timestamp if exists)
echo "hello" > file.txt     # create file with content (OVERWRITES)
echo "more" >> file.txt     # append to file

# Multiline file creation with heredoc:
cat > file.txt << EOF
line 1
line 2
line 3
EOF

# Truncate (empty) an existing file:
> existing_file.txt
```

### Wildcards — Shell Globbing

Before `rm`, `cp`, `mv` or any command takes filenames, the shell expands wildcards:

```bash
*           # matches any characters (including none)
?           # matches exactly ONE character
[abc]       # matches a, b, or c
[!abc]      # matches anything EXCEPT a, b, or c
[a-z]       # matches any lowercase letter
[[:upper:]] # POSIX class: uppercase letters
[[:digit:]] # POSIX class: digits 0-9
[[:alnum:]] # POSIX class: letters and digits
[[:space:]] # POSIX class: whitespace

# Examples:
ls *.txt            # all .txt files
ls file?.txt        # file1.txt, fileA.txt — exactly one char between file and .txt
ls [abc]*.txt       # files starting with a, b, or c
ls BACKUP.[0-9][0-9][0-9]   # BACKUP.001, BACKUP.099, etc.
```

**Golden rule:** test your wildcard with `ls` before using with `rm`:
```bash
ls *.log        # see what will be affected
rm *.log        # then delete — never delete blind
```

### Copying, Moving, Deleting

```bash
# cp — Copy
cp file.txt backup.txt          # copy (overwrites silently)
cp -i file.txt backup.txt       # interactive: asks before overwriting
cp -v file.txt backup.txt       # verbose: shows what it's doing
cp -r dir/ backup_dir/          # copy directory RECURSIVELY
cp -a dir/ backup_dir/          # archive: preserves permissions, ownership, timestamps
cp -u *.html /destination/      # only copy if source is NEWER than destination

# mv — Move/Rename
mv file.txt newname.txt         # rename
mv file.txt /tmp/               # move to /tmp
mv -i file.txt newname.txt      # interactive: asks before overwriting

# rm — Delete (NO UNDO)
rm file.txt                     # delete file silently
rm -i file.txt                  # interactive (asks first) ← habit to build
rm -r dir/                      # delete directory recursively
rm -rf dir/                     # force delete, no prompting ← DANGEROUS, double-check path
rm -f nonexistent               # no error if doesn't exist

# mkdir
mkdir mydir                     # create directory
mkdir -p a/b/c                  # create nested directories at once
mkdir dir1 dir2 dir3            # create multiple directories at once
```

### Links

**Hard links** — two names pointing to the same file data (inode):
```bash
ln file1 file2              # file1 and file2 are now the SAME file
ls -li file1 file2          # -i shows inode numbers — they'll match
# Limitations: can't cross filesystems, can't link directories
```

**Symbolic links (symlinks)** — pointer/shortcut to another file:
```bash
ln -s /var/log/nginx/access.log ~/logs/nginx-access  # create symlink
ls -l                           # shows: nginx-access -> /var/log/nginx/access.log
# If original deleted, symlink breaks (dangling link)
# Can cross filesystems, can link directories
```

### Dashed Filenames — Files Starting with `-`

A file named `-filename` causes problems because the shell interprets the leading dash as a command flag. You'll hit this in OverTheWire Bandit and on real systems with oddly named files.

**Two solutions that always work:**

```bash
# 1. ./ prefix — prepend current directory path
cat ./-filename
rm ./-filename
touch ./-filename

# 2. -- end-of-options delimiter
#    tells the command: stop reading flags, everything after is a filename
cat -- -filename
rm -- -filename
touch -- -filename
```

**All common operations:**
```bash
# Create:
touch ./-filename

# Read/view:
cat ./-filename
cat < -filename         # redirection workaround

# Delete:
rm ./-filename
rm -- -filename

# Rename/move:
mv ./-filename ./newname
mv -- -filename newname

# Find dashed files in current directory:
find . -maxdepth 1 -name '-*'
```

**Why `--` works everywhere:** POSIX standard — almost every Unix command stops parsing flags when it sees `--`. Anything after is treated as a literal argument. Useful beyond dashed files:

```bash
grep -- -v file.txt         # search for literal "-v" in file (not the flag)
git checkout -- file.txt    # restore file (-- separates branch from filename)
```

### Spaces in Filenames

Spaces are legal in filenames but the shell uses spaces as delimiters between arguments — so `my file.txt` looks like two arguments: `my` and `file.txt`.

**Three ways to handle them:**

```bash
# 1. Quotes — wrap the whole name
cat "my project report.txt"
cat 'my project report.txt'
rm "my file.txt"
mv "old name.txt" "new name.txt"

# 2. Backslash escape — escape each space individually
cat my\ project\ report.txt
cd /home/user/my\ documents/

# 3. Tab completion — the easiest way
# type: cat my<TAB>
# shell fills in: cat my\ project\ report.txt  (automatically escaped)
```

**In scripts — always quote your variables:**
```bash
filename="my report.txt"

cat $filename           # BREAKS — shell sees: cat my report.txt (two args)
cat "$filename"         # CORRECT — shell sees one argument

# Common script bug with find:
for f in $(find . -name "*.txt"); do
    cat "$f"            # quotes required — $f may contain spaces
done

# Better: use find with -exec or -print0/-0:
find . -name "*.txt" -exec cat {} \;
find . -name "*.txt" -print0 | xargs -0 cat
```

**Why professionals avoid spaces in filenames:**
1. **Breaks scripts** — unquoted variables with spaces silently do the wrong thing
2. **Web URLs** — spaces become `%20` (`my%20file.pdf`), making links fragile
3. **Tool incompatibility** — Docker, compilers, CI systems, older databases choke on them
4. **Harder to type** — requires escaping or quoting every single time

**Convention:** use underscores (`my_file.txt`) or hyphens (`my-file.txt`) — universally safe across every tool, OS, URL, and script.

### Files with Both Dashes and Spaces

The worst case — a filename that starts with a dash AND has spaces. Both problems hit at once: the dash gets parsed as a flag, the spaces split it into multiple arguments.

```bash
# filename is literally: -- space in file
# or: -my weird file.txt
```

Solve both problems simultaneously — `./ prefix` + `quotes`:

```bash
# ./ prefix + quotes — the reliable combo for any weird filename
cat "./-- space in file"
rm "./-- space in file"
mv "./-- space in file" ./newname.txt

# -- delimiter + quotes also works
cat -- "-- space in file"
rm -- "-- space in file"

# ./ prefix + backslash escaping
cat ./--\ space\ in\ file
```

**The default rule for any weird filename:**
```bash
cat "./<filename>"      # ./ handles the dash, quotes handle the spaces
```

Tab completion works here too — type `./--<TAB>` and the shell fills in the rest automatically. Fastest approach when you can see the filename on screen.

### Identifying Commands

```bash
type ls             # → ls is aliased to 'ls --color=tty'
type cd             # → cd is a shell builtin
type python3        # → python3 is /usr/bin/python3
which go            # → /usr/local/go/bin/go (only works for executables)
```

Commands are one of four things:
1. Executable binary (`/usr/bin/ls`)
2. Shell builtin (`cd`, `echo`, `type`)
3. Shell function
4. Alias

### Getting Help

```bash
man ls              # manual page for ls
man 5 passwd        # man page for SECTION 5 (file format) of passwd
man -f socket       # list ALL man pages named "socket"
man -k partition    # search man pages containing "partition" (same as apropos)
apropos partition   # same as man -k

ls --help           # brief usage info
help cd             # help for shell BUILTINS specifically
```

### Aliases

```bash
alias               # list all current aliases
alias ll='ls -l --color=tty'
alias la='ls -la'
alias grep='grep --color=auto'
alias ..='cd ..'
alias ...='cd ../..'

# Remove an alias:
unalias ll

# To make permanent: add to ~/.bashrc
```

---

## Part 7: Redirection — The Most Important Concept

Every command has three streams:
- `stdin` (0) — keyboard input by default
- `stdout` (1) — screen output by default
- `stderr` (2) — error messages, also screen by default

### Redirecting Output

```bash
# stdout to file (OVERWRITES):
ls -l /usr/bin > ls-output.txt

# Truncate/create empty file:
> ls-output.txt

# stdout to file (APPENDS):
ls -l /usr/bin >> ls-output.txt

# stderr to file:
ls /nonexistent 2> errors.txt

# stdout AND stderr to SAME file:
ls /bin/usr > output.txt 2>&1      # old way (order matters!)
ls /bin/usr &> output.txt          # bash 4+ shorthand

# Append both stdout and stderr:
ls /bin/usr &>> output.txt

# Silence everything:
command > /dev/null 2>&1
```

**Critical:** order matters with `2>&1`:
```bash
> file.txt 2>&1     # CORRECT: stderr goes to wherever stdout goes (the file)
2>&1 > file.txt     # WRONG: stderr goes to screen, stdout goes to file
```

### Redirecting Input

```bash
sort < unsorted.txt > sorted.txt    # read from file, write to file
```

### Pipelines `|` — The Unix Superpower

The `|` sends stdout of one command as stdin to the next:

```bash
ls /usr/bin | less                      # page through a long listing
ls /bin /usr/bin | sort | uniq | less   # combined sorted unique list
ls /usr/bin | sort | uniq | grep zip    # find zip-related programs
ls /usr/bin | wc -l                     # count files in /usr/bin

# Real-world DevOps examples:
ps aux | grep nginx | awk '{print $2}' | xargs kill
cat /var/log/nginx/access.log | grep "POST" | tail -n 50
```

**`>` vs `|`:** The `>` redirects to a FILE. The `|` connects to another COMMAND.

### tee — Split the Stream

```bash
ls /usr/bin | tee ls.txt | grep zip
./build.sh 2>&1 | tee build.log     # log a build while watching it live
```

---

## Part 8: Shell Expansions — What the Shell Does Before Running

### Brace Expansion — Huge Timesaver

```bash
echo Front-{A,B,C}-Back         # → Front-A-Back Front-B-Back Front-C-Back
echo Number_{1..5}              # → Number_1 Number_2 Number_3 Number_4 Number_5
echo {01..15}                   # → 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15
echo {Z..A}                     # → Z Y X W V U T S R Q P O N M L K J I H G F E D C B A

# Real use: create directory structure instantly:
mkdir -p Photos/{2023..2025}-{01..12}
mkdir {logs,config,data,scripts}
```

### Arithmetic Expansion

```bash
echo $((2 + 2))         # → 4
echo $((5**2))          # → 25 (exponent)
echo $((5/2))           # → 2 (integer division)
echo $((5%2))           # → 1 (remainder/modulo)
```

### Command Substitution vs Pipelines

Both use the output of a command — but they hand it to different things.

**Pipeline `|`** feeds output into another command's **stdin**:
```bash
ls /usr/bin | grep zip
# grep reads filenames as input text, line by line
```

**Command substitution `$()`** feeds output in as **arguments** (words on the command line):
```bash
ls -l $(which cp)
# which cp outputs: /bin/cp
# shell replaces $() with that string → becomes: ls -l /bin/cp
```

The difference is where the output lands — **stdin vs arguments**.

Some commands read from stdin (`grep`, `sort`, `wc`, `less`). Others only take arguments, not stdin (`ls`, `file`, `cd`):

```bash
file $(ls /usr/bin | grep zip)   # file doesn't read stdin — needs arguments
ls /usr/bin | grep zip | wc -l   # wc reads stdin fine — pipeline works
```

You can combine them — a pipeline inside `$()`:
```bash
file $(ls -d /usr/bin/* | grep zip)
```

Gotcha: when filenames might have spaces, quote it:
```bash
echo "$(cat file.txt)"     # preserves newlines, treats as one argument
echo $(cat file.txt)       # word-splits on whitespace
```

### Quoting

```bash
# Double quotes — suppress most expansion but allow $, $(), arithmetic:
echo "The total is $((5+5))"    # → The total is 10
echo "two words.txt"            # treat as one argument with space

# Single quotes — suppress ALL expansion:
echo 'text ~/*.txt {a,b} $(echo foo) $((2+2)) $USER'
# → exactly that, no expansion

# Backslash — escape a single character:
echo "The balance is \$5.00"    # → The balance is $5.00
```

---

## Part 9: Advanced Keyboard Tricks

### Cursor Movement (Readline)

```bash
ctrl+A          # move to BEGINNING of line
ctrl+E          # move to END of line
alt+F           # move forward one WORD
alt+B           # move backward one WORD
ctrl+L          # clear screen
```

### Cut and Paste (Kill and Yank)

```bash
ctrl+K          # kill (cut) from cursor to END of line
ctrl+U          # kill from cursor to BEGINNING of line
alt+D           # kill from cursor to end of current word
alt+backspace   # kill from cursor to beginning of current word
ctrl+Y          # yank (paste) killed text at cursor
```

### Tab Completion

```bash
ls Do<TAB>          # → ls Documents  (if unambiguous)
ls D<TAB><TAB>      # → shows all matches starting with D
$HO<TAB>            # → $HOME  (variable completion)
```

---

## Part 10: History

```bash
history             # show command history (numbered)
history | grep go   # search history for "go"

!!                  # repeat LAST command
!88                 # repeat command number 88
!nginx              # repeat LAST command starting with "nginx"

ctrl+R              # reverse incremental search (start typing to search)
                    # ctrl+R again for next match
                    # ctrl+J to copy to command line without executing

ctrl+P              # previous history entry (same as up arrow)
ctrl+N              # next history entry (same as down arrow)
```

---

## Part 11: Users and Permissions

### Users

```bash
whoami              # who am I
id                  # uid, gid, groups
who                 # who is logged in
sudo command        # run as root
su -                # switch to root (needs root password)
```

### File Permissions

Every file has: **owner | group | others** × **read/write/execute**

```bash
ls -la
# -rwxr-xr-- 1 username devs 1234 Jun 19 app.sh
#  ↑↑↑↑↑↑↑↑↑
#  type, owner(u), group(g), others(o)
```

File type first character: `-`=regular file, `d`=directory, `l`=symlink

```
r = read    (4)
w = write   (2)
x = execute (1)
```

For directories: `r` = can list contents, `w` = can create/delete files, `x` = can enter (cd into it)

### chmod — Change Permissions

#### Octal Notation — Understanding It, Not Memorising It

Permissions are 3 bits — read, write, execute — either on (1) or off (0). Three bits gives 8 possible values (0–7), which is why octal fits perfectly: one digit per entity, each digit representing three permission flags.

```
r w x
1 1 1 = 7   rwx  all three on
1 1 0 = 6   rw-  read + write
1 0 1 = 5   r-x  read + execute
1 0 0 = 4   r--  read only
0 0 0 = 0   ---  nothing
```

So `chmod 755` means:
```
7    5    5
rwx  r-x  r-x
↑    ↑    ↑
own  grp  oth
```

You can derive any combination yourself:
```
owner read+write, group read, others nothing → 6 4 0 → chmod 640
everyone reads, owner+group write, no execute → 6 6 4 → chmod 664
```

**Permissions you'll use constantly:**
```
755  rwxr-xr-x  directories, scripts, executables
644  rw-r--r--  regular files, config files
600  rw-------  private files: SSH keys, .env, secrets
700  rwx------  private directories
664  rw-rw-r--  shared files in a team environment (with setgid dir)
777  rwxrwxrwx  everyone can do everything — almost always wrong
```

```bash
chmod 755 script.sh     # rwxr-xr-x
chmod 644 config.txt    # rw-r--r--
chmod 600 ~/.ssh/id_rsa # rw------- (SSH refuses to use keys if permissions are looser)
chmod 640 file.txt      # rw-r-----
```

**Why 777 is almost always wrong** — the real fix is usually ownership:
```bash
# Wrong: throw permissions at the problem
chmod 777 /var/www/uploads/

# Right: find what user actually needs access and fix ownership
ps aux | grep nginx           # nginx runs as www-data
chown www-data:www-data /var/www/uploads/
chmod 755 /var/www/uploads/
```

**Symbolic notation:**
```bash
chmod u+x script.sh          # add execute for owner (u)
chmod go-w file.txt          # remove write from group (g) and others (o)
chmod a+r file.txt           # add read for everyone (a)
chmod u=rwx,g=rx,o=r file    # set all three explicitly
chmod go= file.txt           # set group and others to nothing
```

Symbolic = better when adding/removing one bit. Octal = better when setting everything from scratch.

### chmod -R — The Recursive Trap

`-R` applies the permission to a directory and everything inside it. The problem: files and directories need different permissions, and `-R` treats them the same.

```
755 on a directory = rwxr-xr-x  ← correct, x means "can enter"
755 on a file      = rwxr-xr-x  ← makes every file executable — usually wrong
```

What you almost always actually want:
```bash
find /var/www/ -type d -exec chmod 755 {} \;   # directories get 755
find /var/www/ -type f -exec chmod 644 {} \;   # files get 644
```

**The execute bit means different things on files vs directories:**
```
On a FILE:       x = this file can be run as a program
On a DIRECTORY:  x = you can cd into it / access files inside it
                 r = you can ls and see what's in it
                 w = you can create, delete, rename files inside it
```

Without `x` on a directory, you can't enter it — even knowing the exact path:
```bash
chmod 600 /var/www/
cat /var/www/index.html    # → Permission denied (even though you know the path)
```

**How to read `ls -l` permissions:**
```bash
ls -la /etc/passwd
# -rw-r--r-- 1 root root 2847 Jun 10 /etc/passwd
```

```
- rw- r-- r--
↑ ↑↑↑ ↑↑↑ ↑↑↑
│  │   │   └── others: r-- = 4 = read only
│  │   └────── group:  r-- = 4 = read only
│  └────────── owner:  rw- = 6 = read + write
└───────────── type: - = file, d = directory, l = symlink
```

**Real examples:**
```
drwxr-xr-x   /etc/            → 755 dir, everyone can read and enter
drwx------    /root/           → 700 dir, root only
-rwsr-xr-x   /usr/bin/passwd  → 4755 setuid: runs as root for anyone who calls it
-rw-------   ~/.ssh/id_ed25519 → 600 SSH private key
lrwxrwxrwx   symlink          → 777 on symlinks is cosmetic, perms live on the target
```

### Special Permissions

Three extra bits that sit above the normal nine rwx permissions.

**setuid (4000) — "borrow the owner's identity"**

Normally a program runs **as you**. setuid changes that: the program runs as whoever **owns the file**, regardless of who launched it.

```bash
ls -l $(which passwd)
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
#     ↑ s instead of x = setuid is set
```

`passwd` is owned by root, needs to write to `/etc/shadow` (root-only). setuid means it runs as root for anyone who calls it — does its specific job, exits. High-value attack target: a bug in a setuid root program can mean root access.

```bash
chmod u+s program       # set setuid
chmod 4755 program      # octal form
```

```
root owns file + setuid → runs as root for everyone who calls it
```

**setgid (2000) — two different things**

*On a file:* runs as the file's group. Uncommon.

*On a directory (the DevOps case):* new files inside inherit the **directory's group**, not the creator's primary group.

Without setgid:
```
Alice creates file.txt → alice:alice
Bob   creates data.txt → bob:bob
# neither can write to the other's files
```

With setgid (directory group = `devs`):
```
Alice creates file.txt → alice:devs  ← inherits directory group
Bob   creates data.txt → bob:devs    ← inherits directory group
# everyone in devs can access both
```

```bash
sudo mkdir /shared/project
sudo chown :devs /shared/project       # set directory group to devs
sudo chmod 2775 /shared/project        # 2=setgid, 775=rwxrwxr-x

ls -ld /shared/project
# drwxrwsr-x  ← s in group execute slot = setgid on directory
```

Also needs `umask 0002` — setgid sets the *group*, umask controls the *permissions*. If umask strips the group write bit, it won't work.

**Sticky bit (1000) — "you can only delete your own stuff"**

On a directory: even with write permission, you can only delete files **you own**.

Without sticky: Bob can delete Alice's file in `/tmp` because he has write on the directory.
With sticky: Bob can only delete his own files. Deleting Alice's → Permission denied.

```bash
ls -ld /tmp
# drwxrwxrwt  ← t in others execute slot = sticky bit

chmod +t /shared/uploads
chmod 1777 /tmp             # 1=sticky, 777=everyone reads/writes/enters the dir
```

**Reading them in `ls -l`:**
```
-rwsr-xr-x   setuid on file       (s in owner execute slot)
drwxrwsr-x   setgid on directory  (s in group execute slot)
drwxrwxrwt   sticky bit           (t in others execute slot)

# Capital = special bit set but execute NOT set (usually a mistake):
-rwSr--r--   setuid but not executable
drwxrwxrwT   sticky but others have no execute
```

**Side by side:**
```
setuid on file  → runs as FILE'S OWNER (not who ran it)
setgid on dir   → new files inherit DIRECTORY'S GROUP
sticky on dir   → you can only delete files YOU OWN
```

**Octal with all four digits:**
```bash
chmod 4755 program      # setuid + rwxr-xr-x
chmod 2775 /shared/dir  # setgid + rwxrwxr-x
chmod 1777 /tmp         # sticky + rwxrwxrwx
# chmod 755 is shorthand for chmod 0755 — leading 0 implied
```

### umask — Default Permissions

`umask` controls what permissions are **removed** from newly created files and directories.

**The starting maximums:**
- Files start at `0666` (rw-rw-rw-) — never executable by default
- Directories start at `0777` (rwxrwxrwx)

**The mask subtracts from those maximums:**
```
Default max for files:    0666  →  rw-rw-rw-
umask 0022 removes:       0022  →  ----w--w-
Result:                   0644  →  rw-r--r--

Default max for dirs:     0777  →  rwxrwxrwx
umask 0022 removes:       0022  →  ----w--w-
Result:                   0755  →  rwxr-xr-x
```

```bash
umask               # show current mask
umask 0022          # standard: files=644, dirs=755
umask 0002          # team-friendly: files=664, dirs=775 (group can write)
umask 0077          # paranoid: files=600, dirs=700 (owner only)
umask -S            # symbolic form — shows what's ALLOWED not what's removed
```

**Common masks:**
```
umask   files   dirs    use case
0022    644     755     standard — others can read, nobody else can write
0002    664     775     shared dev environment — group can write
0027    640     750     sensitive — group can read, others get nothing
0077    600     700     private — owner only
```

**The shared directory gotcha** — setgid does nothing useful if umask strips the group write bit:
```bash
# With umask 0022:
touch /shared/file      # → -rw-r--r--  (group can't write)

# With umask 0002:
umask 0002
touch /shared/file      # → -rw-rw-r--  (group can write — works as intended)
```

### chown / chgrp

```bash
sudo chown username file.txt            # change owner
sudo chown username:devs file.txt       # change owner AND group
sudo chown :devs file.txt               # change group only
sudo chown -R username /var/www/        # recursive
passwd                                  # change your own password
```

---

## Part 12: Processes

```bash
ps                  # processes in current terminal session
ps x                # all your processes regardless of terminal
ps aux              # ALL processes from ALL users
ps -ef | grep nginx # is nginx running?
pstree              # processes in tree format

top                 # live process monitor (q=quit, h=help, 1=per-CPU, M=sort by memory)
```

### Background/Foreground

```bash
long_command &      # run in background immediately
jobs                # list background jobs
jobs -l             # list with PIDs
fg %1               # bring job 1 to foreground
bg %1               # resume stopped job in background

ctrl+Z              # STOP (pause) current foreground process
ctrl+C              # KILL (terminate) current foreground process
```

### Signals and kill

```bash
kill PID            # send SIGTERM (polite stop) to process
kill -9 PID         # send SIGKILL (force kill) — last resort
kill -1 PID         # send SIGHUP (reload config — used by nginx, etc.)
kill -l             # list all signal names

killall nginx       # kill all processes named "nginx"
pkill -f processname # kill by name pattern

kill -TERM PID      # graceful stop (default)
kill -KILL PID      # force kill
kill -HUP PID       # reload config
```

### Load Average

```bash
uptime              # how long running + load averages
# output: load average: 0.45, 0.17, 0.12
#                       1min  5min  15min
# On single-core: 1.0 = 100% busy, >1.0 = overloaded
# On 4 cores: 4.0 = 100%, >4.0 = overloaded
```

---

## Part 13: Text Manipulation — Daily Tools

### Regular Expressions — Crash Course

Regex is a pattern language for matching text. Used in `grep`, `sed`, `awk`, `find`, and every programming language.

**Literal characters** — match exactly this text:
```
cat     matches anywhere c-a-t appears: cat, concatenate, scat
```

**The dot `.`** — any single character:
```
c.t     matches: cat, cot, cut, c3t — NOT ct or coat
```

**Anchors:**
```
^       start of line       ^error   lines starting with "error"
$       end of line         error$   lines ending with "error"
^$      empty lines
^error$ lines that are EXACTLY "error" and nothing else
```

**Character classes `[]`:**
```
[abc]      a, b, or c
[a-z]      any lowercase letter
[0-9]      any digit
[^abc]     anything EXCEPT a, b, c  (^ inside [] means NOT)

[bg]zip    bzip or gzip
[^bg]zip   anything but bzip/gzip before "zip"
```

**POSIX classes** (inside `[]`):
```
[[:alpha:]]   any letter       [[:digit:]]   any digit
[[:alnum:]]   letter or digit  [[:space:]]   whitespace
[[:upper:]]   uppercase        [[:lower:]]   lowercase
```

**Quantifiers:**
```
*        zero or more    (BRE + ERE)
+        one or more     (ERE / BRE: \+)
?        zero or one     (ERE / BRE: \?)
{n,m}    n to m times    (ERE / BRE: \{n,m\})

ab*c     matches: ac, abc, abbc     (zero or more b's)
ab+c     matches: abc, abbc NOT ac  (one or more b's)
ab?c     matches: ac or abc only    (b optional)
.+       any non-empty line
```

**Alternation `|` and grouping `()`** (ERE):
```
cat|dog        matches: cat or dog
jpg|jpeg|png   matches any of the three
(cat)+         matches: cat, catcat, catcatcat
(cat|dog)s     matches: cats or dogs
```

Groups capture matched text — used in sed back-references:
```bash
echo "2024-01-15" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# → 15/01/2024  (\1=2024, \2=01, \3=15)
```

**BRE vs ERE:**
```
BRE (default for grep/sed): + ? | () {} are LITERAL unless escaped with \
ERE (use -E flag):          + ? | () {} are SPECIAL by default
```
Rule: use `-E` whenever your pattern has `+`, `?`, `|`, or `()`.

**Escaping:**
```bash
grep '192\.168\.1\.1' file.txt   # \. = literal dot, not "any character"
```

**Real patterns:**
```bash
grep '^$' file.txt                           # empty lines
grep -v '^#' file.txt                        # non-comment lines
grep -E '^[0-9]+$' file.txt                  # lines that are ALL digits
grep -E 'TODO|FIXME' *.go                    # find TODO or FIXME in code
grep -E '={3,}' data.txt                     # 3+ = signs (Bandit!)
grep -E '" (4|5)[0-9]{2} ' access.log        # HTTP 4xx/5xx in nginx logs
```

**Quick reference:**
```
.     any char    ^  start    $  end    [abc]  one of    [^abc]  not
*     zero+       +  one+     ?  0or1   {n,m}  n to m    |  OR   ()  group
\     escape
```

---

### grep — Search

```bash
grep "error" /var/log/syslog        # find "error" in file
grep -r "TODO" ./src/               # recursive search in directory
grep -n "func main" main.go         # show line numbers
grep -i "error" logs.txt            # case insensitive
grep -v "DEBUG" app.log             # lines that DON'T match (invert)
grep -l "pattern" *.txt             # only show FILENAMES that match
grep -L "pattern" *.txt             # only show filenames that DON'T match
grep -c "error" app.log             # count matching lines
grep -A2 "error" app.log            # 2 lines AFTER each match
grep -B2 "error" app.log            # 2 lines BEFORE each match
grep -C2 "error" app.log            # 2 lines context either side

# ERE — use -E whenever pattern has + ? | ()
grep -E 'AAA|BBB' file.txt          # match AAA OR BBB
grep -E '^[[:upper:]]' dirlist.txt  # lines starting with uppercase
grep -E '={3,}' data.txt            # 3 or more = signs

# BRE anchors (no -E needed):
grep '^error' file.txt      # "error" at START of line
grep 'error$' file.txt      # "error" at END of line
grep '^$' file.txt          # empty lines
grep '[bg]zip' file.txt     # "bzip" or "gzip"
grep '[^bg]zip' file.txt    # NOT bzip/gzip
```

### cut — Extract Columns

`cut` removes sections from each line of a file. Works well on structured text with consistent delimiters — `/etc/passwd`, CSVs, TSVs.

**The three modes:**
```bash
# -f : select by FIELD (requires a delimiter with -d)
cut -d: -f1 /etc/passwd             # usernames (field 1, colon-delimited)
cut -d: -f1,7 /etc/passwd           # fields 1 AND 7 (username + shell)
cut -d: -f1-4 /etc/passwd           # fields 1 THROUGH 4
cut -d, -f2,3 data.csv              # fields 2 and 3 of a CSV
cut -f3 distros.txt                 # field 3, tab-delimited (default)
cut --complement -d: -f3 /etc/passwd # everything EXCEPT field 3

# -c : select by CHARACTER POSITION
cut -c1-5 file.txt                  # characters 1 through 5
cut -c7-10 file.txt                 # characters 7 through 10
cut -c-5 file.txt                   # first 5 characters
cut -c5- file.txt                   # character 5 to end of line
```

**The big limitation of cut:** it only works on single-character delimiters. It can't handle multiple spaces as one delimiter — that's where `awk` is better.

```bash
# cut fails on multi-space output like ps:
ps aux | cut -d' ' -f1             # broken — empty fields from multiple spaces

# awk handles it correctly:
ps aux | awk '{print $1}'          # works — awk treats runs of whitespace as one delimiter
```

### sed — Stream Editor

sed reads input line by line, applies editing commands, then outputs the result. Think of it as scriptable find-and-replace that also understands line addresses, deletions, and insertions.

**Substitution:**
```bash
sed 's/foo/bar/' file.txt           # replace FIRST occurrence per line
sed 's/foo/bar/g' file.txt          # replace ALL occurrences (g = global)
sed 's/foo/bar/gi' file.txt         # global + case insensitive
sed -i 's/localhost/0.0.0.0/g' app.conf  # edit file IN PLACE
sed -i.bak 's/foo/bar/g' file.txt   # in-place + save backup as file.txt.bak
```

**Addresses — which lines to operate on:**
```bash
sed '2s/foo/bar/' file.txt          # only line 2
sed '1,5s/foo/bar/' file.txt        # lines 1 through 5
sed '$d' file.txt                   # delete LAST line
sed '1d' file.txt                   # delete first line (strip headers)
sed '/^#/d' config.txt              # delete comment lines
sed '/^$/d' file.txt                # delete empty lines
sed '/pattern/s/foo/bar/' file.txt  # substitute only on lines containing "pattern"
sed '/start/,/end/d' file.txt       # delete from line matching "start" to "end"
sed '/^#/!d' config.txt             # delete everything EXCEPT comment lines
```

**The `-n` flag + `p` command — print selectively:**
```bash
sed -n '5p' file.txt                # print only line 5
sed -n '1,5p' file.txt             # print lines 1 through 5
sed -n '/error/p' file.txt          # print only lines containing "error"
```

**Multiple commands:**
```bash
sed -e 's/foo/bar/' -e 's/baz/qux/' file.txt
sed 's/foo/bar/; s/baz/qux/' file.txt
```

**Changing the delimiter — when your pattern contains slashes:**
```bash
sed 's|/usr/bin|/usr/local/bin|g' script.sh    # use | instead of /
```

**Useful real-world patterns:**
```bash
sed 's/^[[:space:]]*//' file.txt    # strip leading whitespace
sed 's/[[:space:]]*$//' file.txt    # strip trailing whitespace
sed 's/\r//' file.txt              # remove Windows carriage returns
sed 's/  */ /g' file.txt           # collapse multiple spaces into one
sed '/pattern/a\new line content' file.txt  # add line AFTER match
sed '/pattern/i\new line content' file.txt  # add line BEFORE match
sed -E 's/(\w+) (\w+)/\2 \1/' file.txt     # swap two words (back-references)
```

### awk — Field Processing and Reporting

awk is a full programming language designed around processing structured text. Reads input line by line, splits each line into fields, runs your program against each line.

**Built-in variables:**
```bash
$0      # entire current line
$1      # first field
$NF     # LAST field (NF = number of fields)
NR      # current line number
NF      # number of fields on current line
FS      # field separator (default: whitespace)
OFS     # output field separator (default: space)
```

**Field extraction:**
```bash
awk '{print $1}' file.txt           # first field of every line
awk '{print $1, $3}' file.txt       # fields 1 and 3
awk '{print $NF}' file.txt          # last field
awk '{print NR, $0}' file.txt       # line number + whole line
awk -F: '{print $1}' /etc/passwd    # colon delimiter — usernames
awk -F: '{print $1, $7}' /etc/passwd # username and shell
awk 'BEGIN{FS=","} {print $2}' data.csv  # set FS in BEGIN block
```

**Pattern matching:**
```bash
awk '/error/ {print}' app.log           # lines containing "error"
awk '!/error/ {print}' app.log          # lines NOT containing "error"
awk '$3 > 100 {print}' data.txt         # lines where field 3 is > 100
awk '$1 == "GET" {print $7}' access.log # print URL for GET requests
awk 'NR==5' file.txt                    # only line 5
awk 'NR>=5 && NR<=10' file.txt          # lines 5 through 10
awk 'NF > 3' file.txt                   # only lines with more than 3 fields
```

**BEGIN and END blocks:**
```bash
awk 'BEGIN{print "Header"} {print} END{print "Footer"}' file.txt
awk '/error/{count++} END{print count " errors found"}' app.log
awk '{sum += $1} END{print "Total:", sum}' numbers.txt
awk '{sum += $1; count++} END{print "Average:", sum/count}' numbers.txt
awk 'NR==1 || /pattern/' file.txt       # header line plus matching lines
```

**Real-world patterns:**
```bash
# Extract IPs from nginx access log:
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Count HTTP status codes:
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Disk usage alert:
df -h | awk 'NR>1 && $5+0 > 80 {print $6, "is", $5, "full"}'

# Find lines where field 3 matches a pattern:
awk '$3 ~ /^[0-9]+$/' file.txt      # field 3 is all digits
```

**sed vs awk — when to use which:**
```
sed → transforming text: substitutions, deletions, insertions across a stream
awk → extracting and computing from structured data: fields, arithmetic, reports

find-and-replace → sed
column extraction or arithmetic → awk
not sure → awk (it can do everything sed does, just more verbosely)
```

### sort, uniq, wc, tr

```bash
# sort
sort file.txt                       # alphabetical sort
sort -r file.txt                    # reverse
sort -n numbers.txt                 # NUMERIC sort (not lexicographic)
sort -nr numbers.txt                # numeric reverse (largest first)
sort -u file.txt                    # sort and remove duplicates
sort -t: -k3 -n /etc/passwd         # sort by 3rd field, colon-delimited, numeric
du -s /usr/share/* | sort -nr | head  # find largest directories

# uniq (must sort first — only removes adjacent duplicates)
sort file.txt | uniq                # collapses duplicates to one copy each
sort file.txt | uniq -c             # count occurrences of each line
sort file.txt | uniq -d             # only lines that occur 2+ times
sort file.txt | uniq -u             # only lines that occur EXACTLY once ← easy to forget

# wc
wc -l file.txt                      # count lines
wc -w file.txt                      # count words
ls | wc -l                          # how many files in this directory?
grep "error" app.log | wc -l        # how many errors?

# tr
echo "hello" | tr 'a-z' 'A-Z'      # lowercase to UPPERCASE
cat file.txt | tr -d '\r'           # remove Windows carriage returns
tr -d '[:digit:]' < file.txt        # delete all digits
```

**ROT13 — rotate letters by 13 positions:**

ROT13 shifts each letter 13 positions forward in the alphabet, wrapping past Z. Since the alphabet has 26 letters, applying ROT13 twice returns the original — it's self-inverse. Not real encryption (no key) — historically used to hide spoilers from casual glance.

```
hello → uryyb       HELLO → URYYB (case preserved)       uryyb → hello (decode = encode again)
```

```bash
tr 'A-Za-z' 'N-ZA-Mn-za-m' < data.txt        # decode/encode (same operation either way)
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'    # pipe form, identical result
```

Reading the mapping: `A-Z` → `N-ZA-M` means uppercase A–Z maps to N–Z then wraps to A–M; `a-z` → `n-za-m` is the identical pattern for lowercase. `tr` translates character-by-character — wherever it sees A, output N; wherever it sees Z, wrap to M.

**Worked example — find the one line that appears only once in a file:**
```bash
sort data.txt | uniq -u
```
`sort` groups identical lines together (`uniq` only catches *adjacent* duplicates, so this step is required). `-u` then prints only lines with no matching neighbor — i.e. lines occurring exactly once. To sanity-check first: `sort data.txt | uniq -c | sort -n` shows counts, with the count-of-1 line at the top.

### strings and xxd — Inspecting Binary / Unknown Files

When a file isn't plain text — binary data, a compiled program, something encoded or compressed — `cat` and `less` produce garbled output or terminal-breaking control characters. These two tools are how you read it safely.

**`strings` — extract readable text from a binary file:**
```bash
strings data.txt                    # print all printable text runs (default min length: 4 chars)
strings -n 8 data.txt                # only runs of 8+ printable characters (fewer false positives)
strings -a data.txt                  # scan the WHOLE file
strings data.txt | grep "===="       # extract text, then filter for a pattern
strings data.txt | grep -E "={3,}"   # filter for 3+ consecutive = characters
```

What it does: scans the file for sequences of printable ASCII characters at least N characters long (default 4) and prints each as a line. Everything else — binary headers, machine code, non-printable bytes — is silently skipped. This is how you "read" a binary file without garbage flooding your terminal.

**`xxd` — full hex dump, every byte, nothing hidden:**
```bash
xxd data.txt | less         # paged hex dump, for larger files
xxd -l 16 data.txt           # only the first 16 bytes (-l = length)
xxd -l 4 data.txt            # check just the first 4 bytes — often a file signature
xxd data.txt | grep "pattern"  # search the hex/ASCII dump for something specific
xxd -r hexdump.txt > restored.bin  # reverse: hex dump back into binary
```

**Reading xxd output:**
```
00000000: 4865 6c6c 6f2c 2057 6f72 6c64 210a 4279  Hello, World!.By
          ↑ offset            ↑ bytes (hex)         ↑ same bytes as ASCII
```

**Common file signatures (magic bytes) — useful for `xxd -l 4`:**
```
89504e47 → PNG    504b0304 → ZIP/JAR    1f8b → gzip    425a68 → bzip2    7f454c46 → ELF
```

**The relationship between the three inspection tools:**
```
file     → tells you WHAT the file is (its type, in one line)
strings  → extracts the readable TEXT, ignoring binary
xxd      → shows EVERYTHING, raw, byte by byte — use when strings isn't enough
```

**Typical workflow on an unknown file:**
```bash
file data.txt                       # step 1: what is this?
strings data.txt | grep "pattern"   # step 2: try to find readable text directly
xxd data.txt | less                 # step 3: if that fails, look at the raw bytes
```

### base64 — Encode/Decode

`base64` represents binary or arbitrary data using only printable ASCII characters (A–Z, a–z, 0–9, `+`, `/`, with `=` padding). Used wherever binary data needs to travel through text-only channels — email attachments, JSON, URLs, embedded HTML/CSS images. Recognizable by the character set and trailing `=`/`==` padding.

```bash
base64 file.txt               # ENCODE: text/binary → base64 text
base64 -d file.txt            # DECODE: base64 text → original
base64 -d -i file.txt         # decode, ignoring invalid characters (stray whitespace etc.)
echo "aGVsbG8=" | base64 -d   # decode a string directly via pipe — prints: hello

base64 file.txt > encoded.txt     # save encoded output
base64 -d encoded.txt > original  # save decoded output
```

**If decoding produces garbage** — usually a stray newline breaking strict decoders:
```bash
base64 -d -i data.txt                     # -i ignores invalid characters
cat data.txt | tr -d '\n' | base64 -d     # strip newlines first, then decode
```

### Layered Encoding — Files Wrapped in Multiple Formats

Some files are compressed and/or encoded multiple times — base64 wrapping a gzip archive wrapping a tar file, for example. Peel one layer at a time, checking `file` after each step.

```bash
file data.txt                    # ASCII text → likely base64 or similar
base64 -d data.txt > step1       # decode it
file step1                       # gzip? bzip2? tar archive?

gzip -d step1       # if gzip
bzip2 -d step1      # if bzip2
tar -xf step1        # if tar archive — extracts contained file(s)

file step1                       # check again — repeat as needed
```

**Decompression cheat sheet:**
```bash
gzip -d file.gz          # or: gunzip file.gz
bzip2 -d file.bz2        # or: bunzip2 file.bz2
tar -xf file.tar         # extract a tar archive
tar -xzf file.tar.gz     # extract gzip-compressed tar in one step
tar -xjf file.tar.bz2    # extract bzip2-compressed tar in one step
```

`file` inspects actual bytes (magic numbers), not extensions — exactly what you need when a file has no extension or a misleading one.

---

## Part 14: Finding Things

### find — Search Live Filesystem

```bash
find /etc -name "*.conf"                    # all .conf files under /etc
find . -type f -name "*.go"                 # only FILES named *.go
find . -type d -name "logs"                 # only DIRECTORIES named logs
find . -type l                              # only symbolic links
find /home -user username                   # files owned by a user
```

**Size — exact match vs ranges:**
```bash
# Size units: c=bytes, k=KB, M=MB, G=GB
find . -size 1033c                          # EXACTLY 1033 bytes (no + or -)
find . -size +10M                           # MORE than 10MB
find . -size -10M                           # LESS than 10MB
find . -size +10M -size -1G                 # between 10MB and 1GB
```

**Time:**
```bash
# -mtime (modified), -atime (accessed), -ctime (changed inode)
# +n = more than n days ago, -n = within last n days
find . -mtime -1                            # modified in last 24 hours
find . -newer reference_file                # newer than reference_file
```

**Combining conditions — stacking filters is AND by default:**
```bash
find . -type f -size 1033c -not -executable
# → must be a regular file AND exactly 1033 bytes AND not executable

# Explicit AND / OR if you want it spelled out:
find . \( -name "*.go" -o -name "*.sh" \)   # -o = OR, needs \( \) grouping
```

**Permissions and executability:**
```bash
find . -not -executable                     # none of the execute bits are set
find . -executable                          # at least one execute bit is set
find . -perm 644                            # exactly 644
```

**Hidden files — dotfiles:**
```bash
# Hidden files start with a dot: .bashrc, .ssh, .gitignore
# find sees them like any other file, but ls doesn't show them without -a

find . -name ".*" -not -name "." -not -name ".."  # hidden files/dirs, excluding . and ..
find . -maxdepth 1 -name ".*" -type f       # hidden files only, current dir, no recursion
find . -name ".*" -type f                   # hidden files, any depth
find ~ -maxdepth 1 -name ".*"               # hidden files directly in home directory

# ls equivalent (faster for browsing, not scripting):
ls -a            # show all, including hidden
ls -A            # show all except . and ..
```

**Recursion and depth** — `find` recurses by default, unlike `ls` which needs `-R`:
```bash
find . -name "filename.txt"          # current dir AND everything beneath it
find . -maxdepth 1 -name "*.txt"     # current dir only, no recursion
find . -maxdepth 2 -name "*.txt"     # current dir + one level down
find / -name "nginx.conf" 2>/dev/null  # whole filesystem, suppress permission errors
find . -iname "*readme*"             # case-insensitive partial match, any depth
```

**Acting on results (`-exec`):**
```bash
find . -name "*.log" -exec rm {} \;        # find and delete
find . -name "*.go" -exec grep -l "TODO" {} \;  # Go files containing TODO
find . -name "*.log" -ok rm {} \;          # like -exec but prompts before each
```

**xargs — more efficient than -exec for many files:**
```bash
find . -name "*.txt" | xargs grep "pattern"
find . -name "*.log" -print0 | xargs -0 rm  # handles filenames with spaces
```

### locate — Fast Database Search

```bash
locate nginx.conf                   # search pre-built database (instant)
locate -i nginx.conf                # case insensitive
locate --regex 'bin/(bz|gz|zip)'    # with extended regex
locate "*.go"                       # find all Go files on the system

sudo updatedb                       # refresh the database (runs daily automatically)
```

### locate vs find — When to Use Which

```
"Where is this file?" → locate     (exploring, you know it exists)
"Which files match these criteria?" → find   (filtering, scripting, acting on results)
```

**Use locate when:**
- You're at the terminal exploring interactively
- You know the file exists, you just want to find it fast
- `locate nginx.conf` returns instantly vs `find / -name nginx.conf` taking 30+ seconds

**You must use find when:**
1. **File was just created** — locate's database is rebuilt nightly; won't know about new files
2. **You need to act on results** — `find . -name "*.log" -exec rm {} \;`
3. **You're filtering by attributes** — size, time, owner, permissions
4. **You're writing a script** — scripts need reliable, current results

In DevOps: `locate` for interactive exploration at the terminal, `find` for everything in scripts or when you need current results.

---

## Part 15: Networking

### Check Network Status

```bash
ip addr show                # show IP addresses (modern)
ip a                        # shorthand
ip route show               # routing table
ifconfig                    # older equivalent (may need: apt install net-tools)

ping google.com             # is internet reachable?
ping -c 4 google.com        # send exactly 4 packets then stop
traceroute google.com       # trace network path to host

# See open connections and listening ports:
ss -tulpn                   # modern: tcp, udp, listening, processes, numeric
netstat -tulpn              # older equivalent
```

### DNS

```bash
nslookup google.com         # DNS lookup
dig google.com              # detailed DNS lookup
dig +short google.com       # just the IP
host google.com             # simple lookup
cat /etc/resolv.conf        # DNS server config
cat /etc/hosts              # local DNS overrides (checked BEFORE DNS)
```

### scp — Simple Remote Copy

scp works like `cp` but over SSH. Straightforward but always copies everything — no sync, no resume.

```bash
scp file.txt user@server:/tmp/          # copy file TO remote
scp user@server:/var/log/app.log ./     # copy file FROM remote
scp -r dir/ user@server:/app/           # copy directory recursively
scp -i ~/.ssh/mykey.pem file user@server:/tmp/  # with specific key
scp -P 2222 file.txt user@server:/tmp/  # non-standard port
```

### rsync — Smart Sync

rsync only transfers what has changed — dramatically faster than scp for large or repeated transfers.

```bash
rsync -av src/ user@server:/dst/        # sync directory (only changed files)
rsync -av --delete src/ dst/            # sync and DELETE files not in source
rsync -av --dry-run src/ dst/           # preview what WOULD be transferred (test first)
rsync -avz src/ user@server:/dst/       # -z compresses during transfer
rsync -avP src/ dst/                    # -P shows progress + allows resume

# The trailing slash matters:
rsync src/ dst/     # copies CONTENTS of src into dst
rsync src dst/      # copies src DIRECTORY itself into dst → dst/src/
```

**rsync vs scp — when to use which:**
```
scp → one-off copy of a file or small directory you've never copied before
rsync → anything you'll repeat, large directories, keeping two places in sync
```

Four cases where rsync beats scp:
1. **Repeated syncs** — deploying the same codebase repeatedly, only diffs transfer
2. **Large directories** — rsync skips files that haven't changed
3. **Resuming** — `rsync --partial` picks up where it left off
4. **Keeping dirs in sync** — `--delete` removes stale files on the destination

### wget — Download Files

wget is a downloader. Give it a URL, it saves the file to disk.

```bash
wget https://example.com/file.tar.gz    # download file
wget -O output.tar.gz https://example.com/file.tar.gz  # specific name
wget -c https://example.com/file.tar.gz # resume interrupted download
wget -r https://example.com/            # recursive download
wget -q https://example.com/file        # quiet mode
wget -i urls.txt                        # download list of URLs from a file
```

### curl — Transfer Data, Inspect HTTP

curl outputs to stdout by default — pipeline-friendly and great for API work.

```bash
curl https://example.com               # fetch URL, print to stdout
curl -o output.html https://example.com # save to file
curl -O https://example.com/file.tar.gz # save with original filename
curl -L https://example.com            # follow redirects (use almost always)
curl -I https://example.com            # headers ONLY (HEAD request)
curl -i https://example.com            # response headers + body
curl -s https://example.com            # silent (no progress)
curl -v https://example.com            # verbose: full request + response

# HTTP methods:
curl -X POST https://api.example.com/endpoint
curl -d '{"key":"value"}' -H "Content-Type: application/json" https://api.example.com

# Headers and auth:
curl -H "Authorization: Bearer TOKEN" https://api.example.com
curl -u username:password https://example.com

# Test an API and pretty-print JSON:
curl -s https://api.example.com/data | python3 -m json.tool
curl -s https://api.example.com/data | jq .   # with jq installed
```

**wget vs curl — when to use which:**
```
wget → downloading files to disk, mirroring sites, resuming downloads
curl → API calls, inspecting HTTP, piping output, scripting HTTP interactions

wget saves to a file by default.
curl outputs to stdout by default.
```

In DevOps: `curl` for API calls and HTTP debugging. `wget` for downloading large files. `rsync` for any deployment or backup you'll repeat.

### SSH

**Basic usage — username, password, port:**
```bash
ssh username@server_ip              # prompts for password if server allows it
ssh -p 2222 username@server_ip      # non-standard port (-p before the host)

# With all three explicit:
ssh -p 2222 username@server_ip
# then type password when prompted

# Non-interactive (scripting) — sshpass:
# install: apt install sshpass
sshpass -p 'yourpassword' ssh -p 2222 username@server_ip
sshpass -p 'yourpassword' scp -P 2222 file.txt username@server_ip:/tmp/
# note: scp uses -P (uppercase) for port, ssh uses -p (lowercase)
```

**Force password auth** (useful when you have keys configured but want to test):
```bash
ssh -o PreferredAuthentications=password username@server_ip
```

**If password auth is blocked** (`Permission denied (publickey)`) — server has password auth disabled. Default on AWS, GCP, DigitalOcean. Check on the server:
```bash
grep PasswordAuthentication /etc/ssh/sshd_config
# PasswordAuthentication yes   ← allowed
# PasswordAuthentication no    ← blocked, keys only

# To enable (requires root):
sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

Don't use sshpass in production — the password appears in shell history and the process list. Use key-based auth for anything automated.

**Key-based auth — the right way for DevOps:**
```bash
# Generate a key (ed25519 is modern and fast):
ssh-keygen -t ed25519 -C "your@email.com"
# → creates ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)

# Copy your public key to the server (prompts for password this ONE time):
ssh-copy-id username@server_ip
ssh-copy-id -p 2222 username@server_ip      # with non-standard port

# Now SSH with no password ever again:
ssh username@server_ip

# With a specific key file (AWS PEM keys):
ssh -i ~/.ssh/mykey.pem username@server_ip
ssh-copy-id -i ~/.ssh/mykey.pub username@server_ip
```

**Other useful SSH options:**
```bash
ssh -l someone some-server              # specify username with -l flag
ssh user@server 'df -h'                 # run single command remotely
ssh user@server 'bash -s' < script.sh  # run local script on remote server
ssh -L 8080:localhost:80 user@server    # local port forwarding
ssh -N -f -L 8080:localhost:80 user@server  # port forward in background

# ~/.ssh/config — avoid typing flags every time:
# Host myserver
#     HostName 192.168.1.100
#     User ubuntu
#     IdentityFile ~/.ssh/mykey.pem
#     Port 2222
#
# Then just: ssh myserver
# Works for scp too: scp file.txt myserver:/tmp/
```
#     IdentityFile ~/.ssh/mykey.pem
#     Port 2222
# Then just: ssh myserver
```

---

## Part 16: Archiving and Compression

```bash
# gzip — compress (replaces original file):
gzip file.txt               # → file.txt.gz
gunzip file.txt.gz          # decompress
gzip -c file.txt > file.txt.gz  # keep original
zcat file.txt.gz            # view compressed file without extracting
zgrep "pattern" file.gz     # grep inside compressed file

# bzip2 — better compression, slower:
bzip2 file.txt              # → file.txt.bz2
bunzip2 file.txt.bz2

# tar — archive (bundle multiple files):
tar -czf archive.tar.gz dir/    # create gzip-compressed archive
tar -cjf archive.tar.bz2 dir/   # create bzip2-compressed archive
tar -xzf archive.tar.gz         # extract gzip archive
tar -xjf archive.tar.bz2        # extract bzip2 archive
tar -tzf archive.tar.gz         # list contents without extracting
tar -xzf archive.tar.gz -C /dst/ # extract to specific directory
tar -czf backup.tar.gz /etc /home /usr/local  # backup multiple paths

# Options: c=create, x=extract, t=list, z=gzip, j=bzip2, f=file, v=verbose

# zip/unzip (for Windows compatibility):
zip -r archive.zip directory/   # create zip recursively
unzip archive.zip               # extract
unzip -l archive.zip            # list contents
```

---

## Part 17: Package Management

**Ubuntu/Debian (apt):**
```bash
sudo apt update                     # refresh package list
sudo apt upgrade                    # upgrade installed packages
sudo apt install nginx              # install
sudo apt remove nginx               # remove (keep config files)
sudo apt purge nginx                # remove including config files
apt show nginx                      # info about a package
dpkg -l | grep nginx                # is nginx installed?
dpkg -L nginx                       # list files installed by package
dpkg -S /usr/bin/ls                 # which package installed this file?
```

**RHEL/CentOS/Fedora (dnf/yum):**
```bash
sudo dnf install nginx
sudo dnf remove nginx
sudo dnf update
rpm -qa | grep nginx                # query all installed packages
rpm -ql nginx                       # list files in package
```

---

## Part 18: Environment Variables

```bash
echo $HOME              # your home dir
echo $PATH              # where shell looks for executables
echo $SHELL             # your current shell
echo $USER              # your username
printenv                # show all environment variables

export MY_VAR="hello"           # set variable for this session + child processes
MY_VAR=hello command            # set variable only for this one command

# Make permanent — add to ~/.bashrc:
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
source ~/.bashrc                # reload without restarting terminal

# Modifying PATH:
export PATH=$PATH:/usr/local/go/bin     # add to end
export PATH=/my/tools:$PATH             # add to beginning (takes priority)
```

### Startup Files

```bash
# Login shells read (in order, first found wins):
# /etc/profile → ~/.bash_profile → ~/.bash_login → ~/.profile

# Non-login shells (new terminal windows) read:
# /etc/bash.bashrc → ~/.bashrc

# ~/.bashrc is read most often — put your customizations here
```

---

## Part 19: Scheduling

```bash
crontab -e              # edit your cron jobs
crontab -l              # list your cron jobs

# Cron syntax:
# MIN  HOUR  DOM  MON  DOW  command
# 0    2     *    *    *    /scripts/backup.sh    # every day at 2am
# 30   6     *    *    1    /scripts/report.sh    # Monday at 6:30am
# */5  *     *    *    *    /scripts/check.sh     # every 5 minutes
# 0    9     *    *    1-5  /scripts/standup.sh   # weekdays at 9am

# at — run once at a specific time:
at 2am tomorrow
atq                      # list pending at jobs
atrm 3                   # remove at job #3

# sleep — pause execution:
sleep 10                 # wait 10 seconds
sleep 2m                 # wait 2 minutes
sleep 1h                 # wait 1 hour
```

---

## Part 20: System Information

```bash
uname -a                # kernel version + architecture
hostname                # machine name
df -h                   # disk usage by filesystem
du -sh /var/log/        # size of a directory
du -sh *                # size of everything in current directory
free -h                 # RAM usage
uptime                  # how long running + load averages
lscpu                   # CPU info
lsblk                   # block devices (disks, partitions)
dmesg | tail            # kernel messages (hardware events)
```

---

## Part 21: File Editing from the Terminal

**Nano** (beginner-friendly):
```bash
nano file.txt
# ctrl+O → save
# ctrl+X → exit
# ctrl+W → search
# ctrl+K → cut line
# ctrl+U → uncut/paste
```

**Vim** (on every server, learn the basics):
```bash
vim file.txt
# MODES: Normal (default) → insert mode (i) → back to Normal (Esc)

# Saving and quitting (in Normal mode):
# :w    → save
# :q    → quit (fails if unsaved changes)
# :wq   → save and quit
# :q!   → quit WITHOUT saving
# ZZ    → shorthand for :wq

# Navigation (Normal mode):
# h j k l = left down up right
# w = next word, b = previous word
# 0 = beginning of line, $ = end of line
# G = end of file, gg = beginning of file
# :42   = go to line 42

# Editing:
# dd    = delete (cut) line
# yy    = yank (copy) line
# p     = paste after cursor
# u     = undo
# ctrl+R = redo

# Search and replace:
# /pattern  = search forward
# n = next match, N = previous match
# :%s/old/new/g    = replace all in file
# :%s/old/new/gc   = replace all with confirmation

# vimtutor → run this from terminal for an interactive tutorial
```

---

## Part 22: Shell — bash vs zsh in DevOps

**bash is the default on servers. That's what matters.**

When you SSH into an Ubuntu EC2 instance, a Docker container, a CI runner, or any Linux server — bash is there. Always. No configuration needed, guaranteed to be present.

zsh is a developer workstation shell. It's the macOS default since 2019, popular locally because of Oh My Zsh, better autocomplete, and plugins. But it's not installed on servers and nobody installs it there.

**What this means practically:**
- Shell scripts: always `#!/bin/bash`, never `#!/bin/zsh` — portability requires it
- CI/CD pipelines (GitHub Actions, GitLab CI): bash
- Docker containers: bash (or sh for minimal Alpine images)
- Ansible, cloud-init, userdata scripts: bash
- Your local machine: whatever makes you productive — zsh is fine

The distinction is between your **interactive shell** (local, can be zsh) and your **scripting shell** (always bash or sh for portability).

One thing to watch: zsh and bash have subtle differences in arrays, globbing, and string handling. If you write something in your zsh session and it works, don't assume it'll work on a server. Test scripts explicitly with bash, or use `bash -n script.sh` to syntax-check before deploying.

**Bottom line:** learn `.bashrc` deeply — it applies to every machine you touch at work. Use zsh locally if you want, but treat it as a personal preference, not a career skill.

---

## Part 15: DevOps Practical — Real Server Scenarios

Real workflows you'll use on the job and get asked about in interviews.

---

### Diagnosing a Slow Server

The most common interview question. Structured investigation: CPU → memory → disk → network → processes. Don't guess, measure.

```bash
# 1. Quick overview
uptime
# load average: 4.23, 2.10, 1.50
# 4-core machine: 4.23 ≈ 100% utilised. 2-core: overloaded.

# 2. What's using CPU?
top                              # live, sorted by CPU (press M for memory, 1 for per-CPU)
ps aux --sort=-%cpu | head -15   # snapshot, sorted by CPU

# 3. Memory — are we swapping?
free -h
# swap "used" climbing = system paging to disk = ~100x slower than RAM

# 4. Disk I/O — is disk the bottleneck?
iostat -x 1                      # %util near 100% = disk saturated
df -h                            # is a filesystem 100% full?
du -sh /var/log/* | sort -hr | head   # find what's filling disk

# 5. Network
ss -tulpn                        # what's listening
ss -s                            # socket statistics

# 6. Find the culprit
ps aux --sort=-%cpu | head -15
ps aux --sort=-%mem | head -15
lsof -p PID                      # what files/sockets does this process have open?
```

**60-second checklist:**
```bash
uptime && free -h && df -h && ps aux --sort=-%cpu | head -10
```

---

### Service Down

```bash
systemctl status nginx           # is it running? last error?
journalctl -u nginx -n 50        # last 50 lines of service logs
journalctl -u nginx --since "10 min ago"
tail -f /var/log/nginx/error.log # traditional log file

systemctl start nginx            # try to start it
nginx -t                         # test config syntax before restarting

# Is something else on the port?
ss -tulpn | grep :80
lsof -i :80
kill $(lsof -ti :80)             # kill whatever's blocking the port
```

---

### Disk Space Problems

```bash
df -h                            # overview
du -sh /* 2>/dev/null | sort -hr | head    # largest at root
find / -type f -size +500M 2>/dev/null     # files over 500MB

# Clean up:
journalctl --vacuum-size=500M    # trim systemd journal
journalctl --vacuum-time=7d
apt autoremove && apt clean      # old packages (Debian/Ubuntu)
docker system prune              # if Docker is installed — often the culprit
docker system df                 # see how much Docker is using
```

---

### Permission Denied Errors

```bash
ls -la /path/to/file             # what are current permissions?
ls -la /path/to/                 # check the directory too
ps aux | grep nginx              # what user is the process running as?
whoami && id                     # who am I?

# Fix:
sudo chown www-data:www-data /var/www/uploads/
sudo chmod 755 /var/www/uploads/

# Debug by acting as that user:
sudo -u www-data ls /var/www/uploads/
```

---

### Finding What's Using a Port

```bash
ss -tulpn | grep :8080           # modern, preferred
lsof -i :8080                    # shows process name + PID
kill $(lsof -ti :8080)           # kill whatever's on that port
fuser -k 8080/tcp                # alternative
```

---

### Log Analysis

```bash
tail -f /var/log/nginx/access.log          # follow in real time
journalctl -f                              # follow systemd journal
journalctl --since "1 hour ago" | grep -i error

# Count HTTP status codes:
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Top 10 IPs hitting your server:
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Find all 500 errors:
grep '" 500 ' /var/log/nginx/access.log
```

---

### User and Group Management

```bash
# Create a user with home dir and shell:
sudo useradd -m -s /bin/bash username
sudo passwd username

# Add user to a group (without removing from others):
sudo usermod -aG sudo username
sudo usermod -aG docker,devs username

# Remove user (with home directory):
sudo userdel -r username

# See groups a user belongs to:
groups username
id username

# Create a group:
sudo groupadd devs

# Lock/unlock account:
sudo usermod -L username        # lock
sudo usermod -U username        # unlock

# Switch user:
su - username                   # login shell (loads their env)
sudo -u username command        # run single command as that user
```

---

### Sudo Configuration

```bash
# Always use visudo — never edit /etc/sudoers directly:
sudo visudo

# Give full sudo:         username ALL=(ALL:ALL) ALL
# No password prompt:     username ALL=(ALL) NOPASSWD: ALL
# Specific commands only: username ALL=(ALL) /usr/bin/systemctl restart nginx

# Check what a user can sudo:
sudo -l -U username

# Forgot sudo on last command:
sudo !!
```

---

### File Ownership — Practical Scenarios

```bash
# Web server files:
sudo chown -R deploy:www-data /var/www/myapp
sudo find /var/www/myapp -type d -exec chmod 750 {} \;
sudo find /var/www/myapp -type f -exec chmod 640 {} \;

# Shared team directory (everyone in devs can write):
sudo mkdir /shared/project
sudo chown root:devs /shared/project
sudo chmod 2775 /shared/project      # setgid: new files inherit devs group
# team members also need: umask 0002 in ~/.bashrc

# Secret config file (only the app user reads it):
sudo chown appuser:appuser /etc/myapp/config.env
sudo chmod 600 /etc/myapp/config.env

# SSH key permissions (wrong permissions = SSH refuses to use the key):
chmod 600 ~/.ssh/id_ed25519          # private key: owner only
chmod 644 ~/.ssh/id_ed25519.pub      # public key: world-readable is fine
chmod 700 ~/.ssh/                    # directory: owner only
chmod 644 ~/.ssh/authorized_keys
```

---

### systemd — Managing Services

```bash
# Lifecycle:
sudo systemctl start|stop|restart|reload nginx
sudo systemctl status nginx

# Boot behaviour:
sudo systemctl enable nginx           # start on boot
sudo systemctl disable nginx
sudo systemctl enable --now nginx     # enable AND start immediately

# Check state:
systemctl is-enabled nginx
systemctl is-active nginx
systemctl list-units --type=service --state=failed

# Logs:
journalctl -u nginx -n 100            # last 100 lines
journalctl -u nginx -f                # follow
journalctl -u nginx --since "1 hour ago"

# Write a service unit:
# /etc/systemd/system/myapp.service
# [Unit]
# Description=My App
# After=network.target
# [Service]
# User=appuser
# WorkingDirectory=/opt/myapp
# ExecStart=/opt/myapp/myapp
# Restart=on-failure
# [Install]
# WantedBy=multi-user.target

sudo systemctl daemon-reload          # reload unit files after changes
sudo systemctl enable --now myapp
```

---

### Network Configuration

```bash
# Interfaces and IPs:
ip a
ip route show default             # default gateway

# DNS:
dig google.com                    # detailed lookup
cat /etc/resolv.conf              # which DNS server
cat /etc/hosts                    # local overrides (checked before DNS)
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts

# Firewall (ufw):
sudo ufw status
sudo ufw allow 22                 # SSH
sudo ufw allow 80                 # HTTP
sudo ufw allow 443                # HTTPS
sudo ufw deny 3306                # block MySQL from outside
sudo ufw allow from 192.168.1.0/24 to any port 5432  # subnet to postgres
```

---

### Common Interview Questions

**"Walk me through debugging a slow server"**
→ structured: `uptime` (load) → `top` (CPU/memory) → `free -h` (swap) → `df -h` (disk full?) → `iostat` (disk I/O) → `ps aux --sort=-%cpu` (culprit process). Check logs throughout.

**"Site is down after a deploy. What do you do?"**
→ `systemctl status` → `journalctl -u service` → roll back if needed → verify fix.

**"How do you find what's using port 8080?"**
→ `ss -tulpn | grep :8080` or `lsof -i :8080`.

**"Disk is 100% full and the app is down."**
→ `df -h` confirm → `du -sh /* | sort -hr` find culprit → clear logs/old deploys → `df -h` verify → restart service.

**"Difference between `kill` and `kill -9`?"**
→ `kill` = SIGTERM: polite, process cleans up (closes files, flushes buffers). `kill -9` = SIGKILL: immediate, OS does it directly, no cleanup. Always try SIGTERM first.

**"How do you check if a remote server is reachable?"**
```bash
ping server_ip                   # basic reachability
curl -I https://example.com      # HTTP (checks through to the app)
nc -zv server_ip 80              # check a specific port
traceroute server_ip             # where is it failing in the network?
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Where am I? | `pwd` |
| What's here? | `ls -la` |
| Go back where I was | `cd -` |
| Save/restore directory | `pushd /path` / `popd` |
| Find a file (fast) | `locate nginx.conf` |
| Find a file (precise) | `find / -name "nginx.conf"` |
| Search in files | `grep -r "error" /var/log/` |
| Who owns port 80? | `sudo ss -tulpn \| grep :80` |
| Is service running? | `ps aux \| grep nginx` |
| Disk space? | `df -h` |
| RAM? | `free -h` |
| Follow a log | `tail -f /var/log/syslog` |
| What type is this file? | `file unknown_file` |
| Count lines matching | `grep "error" log.txt \| wc -l` |
| Replace in file | `sed -i 's/old/new/g' file.txt` |
| Extract column | `cut -d: -f1 /etc/passwd` |
| Sort + deduplicate | `sort file.txt \| uniq` |
| Copy to server | `scp file.txt user@host:/path/` |
| Sync directory | `rsync -avz src/ user@host:/dest/` |
| Kill by name | `pkill -f processname` |
| Load averages | `uptime` |
| What is this command? | `type command` / `which command` |
| Documentation | `man command` / `command --help` |
| Reload bash config | `source ~/.bashrc` |

---

## What Comes Next

This guide covers the foundation. The natural progression from here:

### Where to Practice

**Linux fundamentals:**

| Resource | What it teaches | Why it's good |
|----------|----------------|---------------|
| [OverTheWire Bandit](https://overthewire.org/wargames/bandit/) | Linux CLI, file permissions, encoding, SSH | Progressive, hands-on — this guide was built alongside it |
| [OverTheWire Natas](https://overthewire.org/wargames/natas/) | Web + Linux server side | Natural next step after Bandit |
| [SadServers](https://sadservers.com) | Broken Linux server troubleshooting | Fix real broken servers — closest to actual DevOps work |
| [TryHackMe](https://tryhackme.com) | Linux, networking, security | Guided paths, good for structured learning |

**DevOps and infrastructure:**

| Resource | What it teaches | Why it's good |
|----------|----------------|---------------|
| [KodeKloud](https://kodekloud.com) | Docker, Kubernetes, Ansible, Linux | Best for DevOps specifically, browser-based labs |
| [Killercoda](https://killercoda.com) | Kubernetes, Linux, Docker | Free, no setup, scenario-based |
| [Game of Pods](https://kodekloud.com/courses/game-of-pods/) | Kubernetes debugging | Debug real broken K8s clusters interactively — concepts stick |

**Cloud:**

| Resource | What it teaches | Why it's good |
|----------|----------------|---------------|
| [AWS Escape Room](https://aws.amazon.com/gameday/) | AWS concepts | Puzzle-based revision, surprisingly fun |
| [Google Cloud Arcade](https://cloudskillsboost.google/arcade) | GCP hands-on | Labs + badges, good for GCP certification prep |
| [flaws.cloud](http://flaws.cloud) | AWS security misconfigurations | Real vulnerable AWS environment to exploit and fix safely |
| [flaws2.cloud](http://flaws2.cloud) | AWS attacker + defender | Covers both attacking and defending — excellent for DevSecOps |

### Learning Path

1. **OverTheWire Bandit** — levels 0–20, muscle memory for CLI fundamentals
2. **SadServers** — apply the sysadmin and debugging skills against real broken servers
3. **Bash scripting** — loops, conditionals, functions. The commands become programs.
4. **Go CLI tools** — `os/exec`, `bufio`, `flag`, `cobra`. Shell knowledge transfers directly.
5. **KodeKloud / Killercoda** — Docker and Kubernetes, now that you have the Linux foundation
6. **flaws.cloud** — cloud security, once you have AWS basics

---

