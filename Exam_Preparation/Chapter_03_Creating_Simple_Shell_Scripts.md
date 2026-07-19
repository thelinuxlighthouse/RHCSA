# Chapter 3: Creating Simple Shell Scripts

## Objectives

After this chapter, you should be able to:

- create and execute a Bash script;
- process `$1`, `$2`, `$#`, and all arguments safely;
- use command substitution;
- make decisions with `if`, `test`, and `[ ]`;
- loop over arguments, command output, and files;
- read input and produce useful exit statuses;
- validate a script with Bash tools; and
- write small administrative scripts that fail clearly rather than silently.

## 1. A script is a saved command procedure

A shell script is a text file containing shell commands. Start with a shebang:

~~~bash
#!/usr/bin/bash
~~~

The kernel uses this line to select the interpreter when the file is executed directly.

Create and run:

~~~bash
$ vim hello.sh
$ chmod u+x hello.sh
$ ./hello.sh
~~~

Or run it explicitly through Bash:

~~~bash
$ bash hello.sh
~~~

When Bash reads a script, it starts a new shell process. A `cd` or variable assignment inside the executed script does not change the parent interactive shell after the script exits.

### A safe starting template

~~~bash
#!/usr/bin/bash

set -o nounset
set -o pipefail

main() {
    printf 'Host: %s\n' "$(hostname)"
    printf 'Date: %s\n' "$(date --iso-8601=seconds)"
}

main "$@"
~~~

`nounset` reports an unset variable. `pipefail` makes a pipeline fail if any component fails. `set -e` can be useful, but its exceptions surprise beginners; explicit error handling is often clearer for exam scripts.

## 2. Variables, quoting, and command substitution

No spaces are allowed around `=` in an assignment:

~~~bash
name="servera"
count=3
~~~

Read variables with `$name` or `${name}`:

~~~bash
printf 'Checking %s\n' "$name"
backup="${name}.tar.gz"
~~~

The braces separate the variable name from following text.

### Quoting rule

Quote variable expansions unless you intentionally want word splitting or filename expansion:

~~~bash
cp -- "$source" "$destination"
rm -- "$file"
~~~

This is unsafe:

~~~bash
rm $file
~~~

If `file` is empty or contains spaces or wildcard characters, the command may behave differently from what the author intended.

### Command substitution

`$(command)` runs a command and substitutes its stdout:

~~~bash
today=$(date +%F)
kernel=$(uname -r)
used=$(df --output=pcent / | tail -n 1 | tr -d ' %')
~~~

It does not capture stderr unless stderr is redirected.

## 3. Positional parameters

Given:

~~~bash
$ ./create-user.sh alice developers
~~~

the script sees:

| Parameter | Value |
| --- | --- |
| `$0` | script path |
| `$1` | alice |
| `$2` | developers |
| `$#` | 2 |
| `"$@"` | every argument, preserving each as a separate word |
| `"$*"` | all arguments combined according to the first IFS character |

Prefer `"$@"` when processing arguments:

~~~bash
for item in "$@"; do
    printf '%s\n' "$item"
done
~~~

Use `shift` to discard the first argument and move the remaining arguments down:

~~~bash
mode=$1
shift

for path in "$@"; do
    printf 'Mode %s, path %s\n' "$mode" "$path"
done
~~~

### Validate required arguments

~~~bash
#!/usr/bin/bash

if [ "$#" -ne 2 ]; then
    printf 'Usage: %s USER GROUP\n' "$0" >&2
    exit 2
fi

user=$1
group=$2
printf 'User=%s Group=%s\n' "$user" "$group"
~~~

Exit status zero means success. A nonzero status means failure. Status 2 is often used for incorrect command syntax, but the key point is consistency.

## 4. Tests and decisions

`test expression` and `[ expression ]` are equivalent. Spaces inside the brackets are required.

~~~bash
if [ -f /etc/hosts ]; then
    echo "hosts is a regular file"
fi
~~~

### File tests

| Test | True when |
| --- | --- |
| `-e path` | path exists |
| `-f path` | path is a regular file |
| `-d path` | path is a directory |
| `-L path` | path is a symbolic link |
| `-r path` | current process can read it |
| `-w path` | current process can write it |
| `-x path` | current process can execute or traverse it |
| `-s path` | file exists and is not empty |

### String tests

~~~bash
[ "$answer" = "yes" ]
[ "$answer" != "yes" ]
[ -n "$answer" ]
[ -z "$answer" ]
~~~

### Integer tests

| Test | Meaning |
| --- | --- |
| `-eq` | equal |
| `-ne` | not equal |
| `-lt` | less than |
| `-le` | less than or equal |
| `-gt` | greater than |
| `-ge` | greater than or equal |

~~~bash
if [ "$used" -ge 90 ]; then
    printf 'CRITICAL: root filesystem is %s%% full\n' "$used"
elif [ "$used" -ge 75 ]; then
    printf 'WARNING: root filesystem is %s%% full\n' "$used"
else
    printf 'OK: root filesystem is %s%% full\n' "$used"
fi
~~~

Do not use `>` for numeric comparison inside `[ ]`; use `-gt`.

### Test a command directly

An `if` condition is any command. It checks the command's exit status:

~~~bash
if systemctl is-active --quiet sshd; then
    echo "sshd is active"
else
    echo "sshd is not active" >&2
fi
~~~

This is clearer than parsing the printed output of `systemctl status`.

### AND, OR, and NOT

~~~bash
if [ -f "$file" ] && [ -r "$file" ]; then
    cat -- "$file"
fi

if [ ! -e "$path" ]; then
    mkdir -p -- "$path"
fi

command1 && command2
command1 || error_handler
~~~

The second command after `&&` runs only if the first succeeds. The command after `||` runs only if the first fails.

## 5. Loops

### Loop over a list

~~~bash
for service in sshd chronyd firewalld; do
    systemctl is-active "$service"
done
~~~

### Loop over arguments

~~~bash
for file in "$@"; do
    if [ -f "$file" ]; then
        printf '%s: %s bytes\n' "$file" "$(stat -c %s -- "$file")"
    else
        printf 'Not a regular file: %s\n' "$file" >&2
    fi
done
~~~

### Loop over matched files

~~~bash
for file in /var/log/*.log; do
    [ -e "$file" ] || continue
    printf '%s\n' "$file"
done
~~~

The guard handles the case where the wildcard matches nothing and Bash leaves the literal pattern unchanged.

### Loop over command output carefully

This is acceptable when output is guaranteed not to contain whitespace:

~~~bash
for user in $(cut -d: -f1 /etc/passwd); do
    printf '%s\n' "$user"
done
~~~

For arbitrary file names, do not use `for file in $(find ...)`; whitespace and wildcard characters break it. A safer form is:

~~~bash
while IFS= read -r -d '' file; do
    printf '%s\n' "$file"
done < <(find /srv/data -type f -print0)
~~~

The current objectives emphasize simple loops, but learning the safe form prevents real damage.

## 6. Read input

~~~bash
read -r -p "Account name: " username
printf 'You entered %s\n' "$username"
~~~

`-r` prevents backslashes from being treated as escapes.

Read a colon-delimited file:

~~~bash
while IFS=: read -r user _ uid gid comment home shell; do
    printf '%-16s UID=%s shell=%s\n' "$user" "$uid" "$shell"
done < /etc/passwd
~~~

The underscore receives a field that the script does not need.

## 7. Arithmetic

~~~bash
count=$((count + 1))
free_mib=$((free_kib / 1024))
~~~

Inside arithmetic expansion, ordinary shell variable names can be used without `$`.

## 8. Functions and reusable logic

~~~bash
log() {
    printf '%s %s\n' "$(date --iso-8601=seconds)" "$*"
}

die() {
    printf 'ERROR: %s\n' "$*" >&2
    exit 1
}

require_root() {
    [ "$(id -u)" -eq 0 ] || die "Run this script as root"
}
~~~

Functions make the main procedure easier to read. Arguments inside a function have their own `$1`, `$2`, and `"$@"`.

## 9. Administrative examples

### Example 1: report service state

~~~bash
#!/usr/bin/bash

if [ "$#" -eq 0 ]; then
    printf 'Usage: %s SERVICE...\n' "$0" >&2
    exit 2
fi

failed=0

for service in "$@"; do
    if systemctl is-active --quiet "$service"; then
        printf 'ACTIVE   %s\n' "$service"
    else
        printf 'INACTIVE %s\n' "$service"
        failed=1
    fi
done

exit "$failed"
~~~

Natural reasoning:

1. Reject an empty request.
2. Test state with a command designed for scripting.
3. Report every requested service rather than stopping at the first failure.
4. Return nonzero if at least one service was inactive.

### Example 2: create backups of readable configuration files

~~~bash
#!/usr/bin/bash

if [ "$#" -lt 2 ]; then
    printf 'Usage: %s DESTINATION FILE...\n' "$0" >&2
    exit 2
fi

destination=$1
shift

mkdir -p -- "$destination" || exit 1

for source in "$@"; do
    if [ -f "$source" ] && [ -r "$source" ]; then
        cp -a -- "$source" "$destination/" ||
            printf 'Copy failed: %s\n' "$source" >&2
    else
        printf 'Skipped unreadable file: %s\n' "$source" >&2
    fi
done
~~~

The destination is processed once, then `shift` leaves only source files in `"$@"`.

### Example 3: disk threshold

~~~bash
#!/usr/bin/bash

threshold=${1:-80}
used=$(df -P / | awk 'NR == 2 { gsub(/%/, "", $5); print $5 }')

if ! [[ "$threshold" =~ ^[0-9]+$ ]]; then
    echo "Threshold must be an integer" >&2
    exit 2
fi

if [ "$used" -ge "$threshold" ]; then
    printf 'ALERT: / is %s%% full; threshold is %s%%\n' "$used" "$threshold"
    exit 1
fi

printf 'OK: / is %s%% full\n' "$used"
~~~

`${1:-80}` uses 80 when argument 1 is unset or empty. `[[ ... =~ ... ]]` is Bash pattern matching and is used here to ensure numeric input before an integer comparison.

## 10. Debug and validate

Check syntax without running:

~~~bash
$ bash -n script.sh
~~~

Trace commands after expansion:

~~~bash
$ bash -x script.sh argument
~~~

Enable tracing for only part of a script:

~~~bash
set -x
commands_to_trace
set +x
~~~

Do not trace a script that handles passwords, tokens, or other secrets; expanded values can be exposed.

Inspect exit status immediately:

~~~bash
$ ./script.sh
$ echo $?
~~~

Useful debugging checklist:

1. Run `bash -n`.
2. Confirm the shebang and execute permission.
3. Print the number and values of arguments.
4. Quote expansions.
5. Confirm spaces around `[ ]`.
6. Test commands directly and inspect `$?`.
7. Run with `bash -x` when no secret is involved.

## 11. Exam lab

Write `/usr/local/bin/user-report` with these requirements:

- it accepts one or more account names;
- if no name is supplied, it prints usage to stderr and exits 2;
- for every existing account, it prints the name, UID, home, and shell;
- it reports nonexistent accounts to stderr;
- it exits 1 if any requested account does not exist, otherwise 0.

One correct solution:

~~~bash
#!/usr/bin/bash

if [ "$#" -eq 0 ]; then
    printf 'Usage: %s USER...\n' "$0" >&2
    exit 2
fi

result=0

for user in "$@"; do
    if entry=$(getent passwd "$user"); then
        IFS=: read -r name _ uid _ _ home shell <<< "$entry"
        printf '%s UID=%s HOME=%s SHELL=%s\n' \
            "$name" "$uid" "$home" "$shell"
    else
        printf 'No such account: %s\n' "$user" >&2
        result=1
    fi
done

exit "$result"
~~~

Install and verify:

~~~bash
# chmod 755 /usr/local/bin/user-report
# bash -n /usr/local/bin/user-report
# /usr/local/bin/user-report root nobody missing-user
# echo $?
~~~

## Quick check

- Can you explain the difference between `"$@"` and unquoted `$@`?
- Can you validate the number of arguments and print usage to stderr?
- Can you use string, integer, and file tests correctly?
- Can you test service state without parsing decorative status output?
- Can you loop over every argument safely?
- Can you capture command output and still check whether the command succeeded?
- Can you make a script return a meaningful exit status?

## Official references

- [EX200 objectives](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam)
- Local pages: `man bash`, `help test`, `help if`, `help for`, and `help read`

