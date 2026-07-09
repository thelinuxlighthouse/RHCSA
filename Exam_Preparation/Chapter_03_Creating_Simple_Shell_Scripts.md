# Chapter 3: Creating Simple Shell Scripts

## Learning Objectives

By the end of this chapter, you will be able to:

- Write executable shell scripts with proper structure and shebang lines
- Use conditional statements (if, test, [], [[]]) to control script flow
- Implement looping constructs (for, while, until) to process files and data
- Process command-line arguments ($1, $2, $@, $#, $?)
- Capture and process the output of shell commands using command substitution
- Use variables, arithmetic operations, and string manipulation in scripts
- Validate script inputs and handle errors gracefully
- Debug shell scripts effectively

---

## Concepts and Theory

### What Is a Shell Script?

A shell script is a plain text file containing a sequence of commands that the shell executes sequentially, just as if you had typed them interactively at the command line. Shell scripts automate repetitive tasks, combine multiple commands into reusable workflows, and enable conditional logic and iteration that would be tedious to perform manually.

In RHEL 10, Bash (`/bin/bash`) is the default shell and the standard scripting environment. Bash provides variables, conditionals, loops, functions, arithmetic expansion, and powerful text processing capabilities.

### Script Execution Model

When you execute a script, Bash follows this process:

1. **Read**: Bash reads the script file line by line
2. **Parse**: Each line is parsed for syntax, variable expansion, command substitution, and redirection
3. **Execute**: Commands are executed in order
4. **Subshell**: Each pipeline runs in a subshell; loops and conditionals run in the current shell context

There are two ways to execute a script:

- **Direct execution**: `./script.sh` (requires execute permission and a valid shebang)
- **Interpreter invocation**: `bash script.sh` (explicitly calls Bash to interpret the file)

### The Shebang Line

The shebang (`#!`) is the first line of a script, specifying which interpreter should execute the script. When you run a script directly (e.g., `./script.sh`), the kernel reads the shebang and invokes the specified interpreter.

```bash
#!/bin/bash
#!/usr/bin/env bash
```

The `#!/bin/bash` form uses an absolute path and is preferred in production. The `#!/usr/bin/env bash` form searches for `bash` in `$PATH`, offering portability at the cost of a slight security concern.

### Variables in Bash

Variables store data that can be referenced and modified throughout a script.

**Variable Types:**

- **Shell variables**: Store text strings, numbers, or command output
- **Environment variables**: Inherited by child processes (exported with `export`)
- **Positional parameters**: Command-line arguments (`$1`, `$2`, etc.)
- **Special variables**: Predefined by Bash (`$?`, `$#`, `$@`, `$*`, `$$`, `$0`)

**Variable Naming Rules:**

- Names begin with a letter or underscore
- Contain only letters, numbers, and underscores
- Are case-sensitive (`MyVar` differs from `myvar`)
- No spaces around the `=` in assignment

**Variable Expansion:**

Variables are referenced with the `$` prefix. Use curly braces `${var}` for clarity, especially when adjacent to other characters:

```bash
name="world"
echo "Hello, $name"        # Works
echo "Hello, ${name}!"     # Preferred for clarity
```

### Conditional Execution

Conditional statements allow scripts to make decisions based on conditions. Bash provides several mechanisms:

**The `test` Command and `[ ]`:**

The `test` command evaluates an expression and returns an exit status of 0 (true) or 1 (false). The `[ ]` construct is a synonym for `test` and is more commonly used.

**The `[[ ]]` Construct:**

The double bracket `[[ ]]` is a Bash keyword that provides enhanced pattern matching, regex support, and safer evaluation (no word splitting or pathname expansion inside).

**Conditional Operators:**

| Operator | Description | Example |
|----------|-------------|---------|
| `-f` | File exists and is a regular file | `[ -f /etc/hosts ]` |
| `-d` | File exists and is a directory | `[ -d /tmp ]` |
| `-e` | File exists (any type) | `[ -e /var/log ]` |
| `-r` | File is readable | `[ -r file.txt ]` |
| `-w` | File is writable | `[ -w file.txt ]` |
| `-x` | File is executable | `[ -x script.sh ]` |
| `-s` | File exists and has size > 0 | `[ -s file.txt ]` |
| `-eq` | Equal (numeric) | `[ $a -eq $b ]` |
| `-ne` | Not equal (numeric) | `[ $a -ne $b ]` |
| `-gt` | Greater than (numeric) | `[ $a -gt $b ]` |
| `-lt` | Less than (numeric) | `[ $a -lt $b ]` |
| `-ge` | Greater than or equal | `[ $a -ge $b ]` |
| `-le` | Less than or equal | `[ $a -le $b ]` |
| `=` | Strings are equal | `[ "$a" = "$b" ]` |
| `!=` | Strings are not equal | `[ "$a" != "$b" ]` |
| `-z` | String is empty | `[ -z "$var" ]` |
| `-n` | String is not empty | `[ -n "$var" ]` |

**Logical Operators:**

| Operator | Description | Context |
|----------|-------------|---------|
| `-a` | AND | Inside `[ ]` |
| `-o` | OR | Inside `[ ]` |
| `&&` | AND | Between commands or `[[]]` |
| `\|\|` | OR | Between commands or `[[]]` |
| `!` | NOT | Before condition |

### Looping Constructs

Loops allow scripts to repeat actions multiple times.

**For Loop:** Iterates over a list of items. Commonly used for processing files, arrays, or sequences.

**While Loop:** Repeats as long as a condition is true. Useful for reading input line by line or waiting for a condition.

**Until Loop:** Repeats until a condition becomes true. The inverse of `while`.

### Command Substitution

Command substitution captures the output of a command and assigns it to a variable or embeds it in another command.

```bash
# Modern syntax (preferred)
variable=$(command)

# Legacy syntax (still valid)
variable=`command`
```

The modern `$()` syntax is preferred because it nests cleanly, is easier to read, and handles complex commands better.

### Exit Status and Error Handling

Every command returns an exit status: `0` means success, non-zero means failure. The special variable `$?` holds the exit status of the last executed command.

Scripts should check exit statuses to handle errors gracefully. The `set -e` option causes the script to exit immediately if any command fails.

---

## How It Works Internally

### Script Parsing and Execution

When Bash executes a script, it does not compile the code. Instead, it reads and interprets each line sequentially:

1. **Line Reading**: Bash reads one line (or logical line, if continued with `\`) at a time
2. **Tokenization**: The line is broken into tokens (words, operators, redirections)
3. **Expansions**: In order: brace expansion, tilde expansion, parameter/variable expansion, command substitution, arithmetic expansion, word splitting, pathname expansion (globbing)
4. **Redirection Processing**: Input/output redirections are set up
5. **Execution**: The command is executed, either as a builtin, function, alias, or external binary

This interpretation model means scripts are flexible but slower than compiled programs. For most administrative tasks, the performance difference is negligible.

### How Conditionals Work

The `if` statement works by executing the test command and checking its exit status:

```bash
if command; then
    # Runs if command exits with status 0
fi
```

The `[ ]` and `[[ ]]` constructs are themselves commands that evaluate expressions and return exit status 0 (true) or 1 (false). The `test` command, `[`, and `[[` all work this way—Bash does not have a native "boolean" type; truth is represented by exit codes.

The `[[ ]]` construct is a Bash keyword (not a command), which means:

- No word splitting occurs inside `[[ ]]`
- No pathname expansion occurs inside `[[ ]]`
- Pattern matching (`==`) uses glob patterns, not exact strings
- Regular expression matching (`=~`) is supported

### How Variable Expansion Works

When Bash encounters `$variable` or `${variable}`, it replaces the reference with the variable's value. This happens before the command executes.

**Quoting behavior:**

- Double quotes (`"$var"`): Variable expands, but no word splitting or globbing
- Single quotes (`'$var'`): No expansion; literal string
- Unquoted (`$var`): Variable expands, then word splitting and globbing occur

This is why best practice dictates always quoting variable expansions: `"$variable"`.

### How Loops Iterate

A `for` loop works by splitting the word list using the Internal Field Separator (`IFS`, default: space, tab, newline), then expanding any glob patterns. Each resulting word is assigned to the loop variable in turn.

For example:

```bash
for file in /tmp/*.txt; do
    # If /tmp has a.txt, b.txt, c.txt, the loop runs 3 times
done
```

If no files match the glob, the literal string `"/tmp/*.txt"` is used as the value unless `nullglob` is enabled.

### How Command Substitution Works

When Bash encounters `$(command)`, it:

1. Creates a subshell
2. Executes the command in the subshell
3. Captures the stdout of the command
4. Removes trailing newlines from the output
5. Substitutes the result into the original command line

The subshell has its own environment, so variable changes inside `$(...)` do not affect the parent shell.

---

## Commands and Administration Tasks

### Creating Your First Script

#### Script Structure

```bash
#!/bin/bash
#
# Description: Your first script
# Author: Administrator
# Date: 2024-01-15
#

# Script body goes here
echo "Hello, World!"
```

#### Making Scripts Executable

```bash
# Create script file
vi /home/admin/first_script.sh

# Add execute permission
chmod +x /home/admin/first_script.sh

# Execute the script
./first_script.sh
/home/admin/first_script.sh
```

#### Script Best Practices

```bash
#!/bin/bash
# Enable strict mode for robust scripts
set -euo pipefail

# set -e : Exit on any command failure
# set -u : Exit on undefined variable usage
# set -o pipefail : Pipeline fails if any command fails

# Your script logic here
```

### Working with Variables

#### Defining and Using Variables

```bash
#!/bin/bash

# Define variables
SERVER_NAME="web01"
SERVER_IP="192.168.1.10"
MAX_RETRIES=3

# Use variables (always quote them)
echo "Server: $SERVER_NAME"
echo "IP: ${SERVER_IP}"

# Define variable from command output
CURRENT_DATE=$(date +%Y-%m-%d)
CURRENT_USER=$(whoami)
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}')

# Define variable with default value
BACKUP_DIR="${BACKUP_DIR:-/var/backups}"

# Read variable from user input
read -p "Enter server name: " SERVER_NAME
read -s -p "Enter password: " PASSWORD
```

#### Variable Manipulation

```bash
#!/bin/bash

filename="report.tar.gz"

# Get length of string
echo ${#filename}               # 12

# Extract substring
echo ${filename:0:6}            # report
echo ${filename:7}              # tar.gz

# Replace pattern
echo ${filename/.tar.gz/.zip}   # report.zip

# Remove suffix
echo ${filename%.gz}            # report.tar
echo ${filename##*.}            # gz (file extension)

# Change case
echo ${filename^^}              # REPORT.TAR.GZ
echo ${filename,,}              # report.tar.gz
```

#### Arrays

```bash
#!/bin/bash

# Define array
SERVERS=("web01" "web02" "db01" "app01")

# Access elements (zero-indexed)
echo ${SERVERS[0]}              # web01
echo ${SERVERS[2]}              # db01

# Get all elements
echo ${SERVERS[@]}              # web01 web02 db01 app01

# Get array length
echo ${#SERVERS[@]}             # 4

# Iterate over array
for server in "${SERVERS[@]}"; do
    echo "Checking $server"
done

# Add element
SERVERS+=("cache01")

# Remove element
unset SERVERS[1]
```

### Conditional Statements

#### Simple if Statement

```bash
#!/bin/bash

# Check if a file exists
if [ -f /etc/hosts ]; then
    echo "/etc/hosts exists"
fi

# Check command exit status
if ping -c 1 -W 3 192.168.1.1 > /dev/null 2>&1; then
    echo "Gateway is reachable"
fi
```

#### if/else Statement

```bash
#!/bin/bash

# Check disk usage
DISK_USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$DISK_USAGE" -gt 90 ]; then
    echo "CRITICAL: Disk usage is ${DISK_USAGE}%"
    echo "Alert sent to administrator"
elif [ "$DISK_USAGE" -gt 75 ]; then
    echo "WARNING: Disk usage is ${DISK_USAGE}%"
else
    echo "OK: Disk usage is ${DISK_USAGE}%"
fi
```

#### Nested Conditionals

```bash
#!/bin/bash

# Check service health
SERVICE="nginx"

if systemctl is-active "$SERVICE" > /dev/null 2>&1; then
    if curl -sf http://localhost/health > /dev/null 2>&1; then
        echo "$SERVICE is healthy"
    else
        echo "$SERVICE is running but unhealthy"
    fi
else
    echo "$SERVICE is not running"
    echo "Attempting to start $SERVICE..."
    systemctl start "$SERVICE"
fi
```

#### Using [[ ]] for Advanced Testing

```bash
#!/bin/bash

filename="report_2024.pdf"

# Pattern matching
if [[ "$filename" == *.pdf ]]; then
    echo "This is a PDF file"
fi

# Regular expression matching
if [[ "2024-01-15" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]]; then
    echo "Valid date format"
fi

# Multiple conditions
if [[ -f "/etc/hosts" && -r "/etc/hosts" ]]; then
    echo "/etc/hosts exists and is readable"
fi

# String comparison (lexicographic)
if [[ "apple" < "banana" ]]; then
    echo "apple comes before banana"
fi
```

#### case Statement

```bash
#!/bin/bash

# Simple menu system
echo "Select an option:"
echo "1) Check disk space"
echo "2) Check memory usage"
echo "3) Check running services"
echo "4) Exit"
read -p "Enter choice [1-4]: " choice

case "$choice" in
    1)
        echo "=== Disk Space ==="
        df -h
        ;;
    2)
        echo "=== Memory Usage ==="
        free -h
        ;;
    3)
        echo "=== Running Services ==="
        systemctl list-units --type=service --state=running
        ;;
    4)
        echo "Goodbye!"
        exit 0
        ;;
    *)
        echo "Invalid option: $choice"
        exit 1
        ;;
esac
```

### Looping Constructs

#### for Loop with Lists

```bash
#!/bin/bash

# Iterate over a list of items
for color in red green blue yellow; do
    echo "Color: $color"
done

# Iterate over a range
for i in {1..5}; do
    echo "Number: $i"
done

# Iterate with step
for i in $(seq 1 2 10); do
    echo "Odd number: $i"
done

# Iterate over array
USERS=("alice" "bob" "charlie")
for user in "${USERS[@]}"; do
    echo "Processing user: $user"
done
```

#### for Loop with Files

```bash
#!/bin/bash

# Process all .log files in a directory
for logfile in /var/log/*.log; do
    # Check if glob matched actual files
    if [ -f "$logfile" ]; then
        FILESIZE=$(stat -c%s "$logfile")
        echo "$logfile: $FILESIZE bytes"
    fi
done

# Process files from find command
for file in $(find /tmp -name "*.tmp" -mtime +7); do
    echo "Old temp file: $file"
    rm -f "$file"
done
```

#### while Loop

```bash
#!/bin/bash

# Counting loop
counter=1
while [ "$counter" -le 10 ]; do
    echo "Counter: $counter"
    ((counter++))
done

# Read file line by line
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hosts

# Wait for a condition
MAX_WAIT=60
elapsed=0
while ! systemctl is-active nginx > /dev/null 2>&1; do
    if [ "$elapsed" -ge "$MAX_WAIT" ]; then
        echo "Timeout waiting for nginx"
        exit 1
    fi
    echo "Waiting for nginx to start... (${elapsed}s)"
    sleep 2
    ((elapsed+=2))
done
echo "nginx is running"
```

#### until Loop

```bash
#!/bin/bash

# Retry mechanism
RETRIES=0
MAX_RETRIES=5

until ping -c 1 -W 2 192.168.1.100 > /dev/null 2>&1; do
    ((RETRIES++))
    if [ "$RETRIES" -ge "$MAX_RETRIES" ]; then
        echo "Failed after $MAX_RETRIES attempts"
        exit 1
    fi
    echo "Connection failed, retry $RETRIES..."
    sleep 3
done
echo "Connected to 192.168.1.100"
```

#### Loop Control

```bash
#!/bin/bash

# break - exit loop early
for i in {1..100}; do
    if [ "$i" -eq 10 ]; then
        break
    fi
    echo "$i"
done

# continue - skip to next iteration
for file in /var/log/*.log; do
    if [ ! -f "$file" ]; then
        continue
    fi
    if [ "$(stat -c%s "$file")" -eq 0 ]; then
        echo "Skipping empty file: $file"
        continue
    fi
    echo "Processing: $file"
done
```

### Processing Script Inputs

#### Positional Parameters

```bash
#!/bin/bash

# $0 - Script name
# $1, $2, ... - Positional arguments
# $# - Number of arguments
# $@ - All arguments as separate words
# $? - Exit status of last command
# $$ - Script's process ID

echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "Number of arguments: $#"
echo "All arguments: $@"
echo "Process ID: $$"

# Example: backup script with arguments
SOURCE_DIR="$1"
DEST_DIR="$2"

if [ -z "$SOURCE_DIR" ] || [ -z "$DEST_DIR" ]; then
    echo "Usage: $0 <source_dir> <destination_dir>"
    exit 1
fi
```

#### Input Validation

```bash
#!/bin/bash

# Check minimum number of arguments
if [ "$#" -lt 2 ]; then
    echo "Usage: $0 <username> <group>"
    echo "  Creates a user and adds to specified group"
    exit 1
fi

USERNAME="$1"
GROUPNAME="$2"

# Validate username format (letters, numbers, hyphens, underscores only)
if [[ ! "$USERNAME" =~ ^[a-zA-Z0-9_-]+$ ]]; then
    echo "Error: Invalid username format"
    exit 1
fi

# Validate that username is not empty
if [ -z "$USERNAME" ]; then
    echo "Error: Username cannot be empty"
    exit 1
fi

# Validate numeric input
read -p "Enter port number: " PORT
if [[ ! "$PORT" =~ ^[0-9]+$ ]] || [ "$PORT" -lt 1 ] || [ "$PORT" -gt 65535 ]; then
    echo "Error: Port must be a number between 1 and 65535"
    exit 1
fi
```

#### Using getopts for Option Parsing

```bash
#!/bin/bash

# Parse command-line options
usage() {
    echo "Usage: $0 [-u username] [-g group] [-d home_dir] [-h]"
    exit 1
}

USERNAME=""
GROUP=""
HOMEDIR=""

while getopts "u:g:d:h" opt; do
    case "$opt" in
        u) USERNAME="$OPTARG" ;;
        g) GROUP="$OPTARG" ;;
        d) HOMEDIR="$OPTARG" ;;
        h) usage ;;
        *) usage ;;
    esac
done

if [ -z "$USERNAME" ]; then
    echo "Error: Username is required"
    usage
fi

echo "Creating user: $USERNAME"
echo "Group: ${GROUP:-default}"
echo "Home: ${HOMEDIR:-/home/$USERNAME}"
```

### Processing Command Output

#### Command Substitution

```bash
#!/bin/bash

# Capture command output
HOSTNAME=$(hostname)
DATE=$(date +%Y-%m-%d_%H:%M:%S)
USER_COUNT=$(wc -l < /etc/passwd)
DISK_FREE=$(df -h / | awk 'NR==2 {print $4}')

echo "Host: $HOSTNAME"
echo "Date: $DATE"
echo "Users: $USER_COUNT"
echo "Free disk: $DISK_FREE"

# Nested command substitution
LARGEST_FILE=$(ls -S /var/log/ 2>/dev/null | head -1)
echo "Largest log file: $LARGEST_FILE"

# Process output in a variable
FAILED_LOGINS=$(grep "Failed password" /var/log/secure 2>/dev/null | \
    awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -5)
echo "Top failed login sources:"
echo "$FAILED_LOGINS"
```

#### Processing Output with Pipes in Scripts

```bash
#!/bin/bash

# Generate a system report
generate_report() {
    echo "=== System Report ==="
    echo "Hostname: $(hostname)"
    echo "Kernel: $(uname -r)"
    echo "Uptime: $(uptime -p)"
    echo ""
    echo "=== Disk Usage ==="
    df -h | grep -v tmpfs
    echo ""
    echo "=== Top Memory Processes ==="
    ps aux --sort=-%mem | head -6
    echo ""
    echo "=== Listening Services ==="
    ss -tlnp
}

# Save report to file
generate_report > "/tmp/system_report_$(date +%Y%m%d).txt"
echo "Report saved"
```

#### Arithmetic Operations

```bash
#!/bin/bash

# Arithmetic expansion
a=10
b=5

echo $((a + b))     # 15
echo $((a - b))     # 5
echo $((a * b))     # 50
echo $((a / b))     # 2
echo $((a % b))     # 0

# Increment and decrement
counter=0
((counter++))
echo $counter       # 1

# Arithmetic in conditionals
if [ $((a * b)) -gt 40 ]; then
    echo "Product is greater than 40"
fi

# Using expr (legacy, avoid in new scripts)
result=$(expr 10 + 5)
```

---

## Configuration Examples

### System Health Check Script

File: `/usr/local/bin/health_check.sh`

```bash
#!/bin/bash
#
# System Health Check Script
# Checks disk, memory, processes, and services
#

set -euo pipefail

REPORT_FILE="/var/log/health_check_$(date +%Y%m%d_%H%M%S).log"
DISK_THRESHOLD=85
MEMORY_THRESHOLD=90
LOG_FILE="/var/log/health_check.log"

# Function to log messages
log_message() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message" | tee -a "$LOG_FILE"
}

# Function to check disk usage
check_disk() {
    log_message "INFO" "Checking disk usage..."
    while IFS= read -r line; do
        usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
        mount=$(echo "$line" | awk '{print $6}')
        if [ "$usage" -gt "$DISK_THRESHOLD" ]; then
            log_message "WARNING" "Disk usage on $mount is ${usage}%"
        else
            log_message "INFO" "Disk usage on $mount is ${usage}%"
        fi
    done < <(df -h | grep -v tmpfs)
}

# Function to check memory usage
check_memory() {
    log_message "INFO" "Checking memory usage..."
    mem_total=$(free -m | awk '/^Mem:/ {print $2}')
    mem_used=$(free -m | awk '/^Mem:/ {print $3}')
    mem_percent=$((mem_used * 100 / mem_total))

    if [ "$mem_percent" -gt "$MEMORY_THRESHOLD" ]; then
        log_message "WARNING" "Memory usage is ${mem_percent}%"
    else
        log_message "INFO" "Memory usage is ${mem_percent}%"
    fi
}

# Function to check critical services
check_services() {
    local services=("sshd" "cron" "firewalld")

    log_message "INFO" "Checking critical services..."
    for service in "${services[@]}"; do
        if systemctl is-active "$service" > /dev/null 2>&1; then
            log_message "INFO" "Service $service is running"
        else
            log_message "ERROR" "Service $service is NOT running"
        fi
    done
}

# Function to check for zombie processes
check_zombies() {
    local zombie_count
    zombie_count=$(ps aux | awk '$8 ~ /Z/ {count++} END {print count+0}')

    if [ "$zombie_count" -gt 0 ]; then
        log_message "WARNING" "Found $zombie_count zombie processes"
    else
        log_message "INFO" "No zombie processes found"
    fi
}

# Main execution
log_message "INFO" "=== Health check started ==="
check_disk
check_memory
check_services
check_zombies
log_message "INFO" "=== Health check completed ==="

exit 0
```

### User Account Management Script

File: `/usr/local/bin/create_user.sh`

```bash
#!/bin/bash
#
# User Account Creation Script
# Creates a user with specified group and sets initial password
#

set -euo pipefail

# Default values
DEFAULT_SHELL="/bin/bash"
DEFAULT_GROUP="users"
LOG_FILE="/var/log/user_creation.log"

usage() {
    echo "Usage: $0 -u <username> [-g <group>] [-s <shell>] [-c <comment>]"
    echo "  -u  Username (required)"
    echo "  -g  Group name (default: $DEFAULT_GROUP)"
    echo "  -s  Login shell (default: $DEFAULT_SHELL)"
    echo "  -c  Comment/description"
    exit 1
}

# Parse options
USERNAME=""
GROUP="$DEFAULT_GROUP"
SHELL="$DEFAULT_SHELL"
COMMENT=""

while getopts "u:g:s:c:h" opt; do
    case "$opt" in
        u) USERNAME="$OPTARG" ;;
        g) GROUP="$OPTARG" ;;
        s) SHELL="$OPTARG" ;;
        c) COMMENT="$OPTARG" ;;
        h) usage ;;
        *) usage ;;
    esac
done

# Validate input
if [ -z "$USERNAME" ]; then
    echo "Error: Username is required"
    usage
fi

# Check if user already exists
if id "$USERNAME" > /dev/null 2>&1; then
    echo "Error: User $USERNAME already exists"
    echo "[$(date)] FAILED: User $USERNAME already exists" >> "$LOG_FILE"
    exit 1
fi

# Check if group exists, create if not
if ! getent group "$GROUP" > /dev/null 2>&1; then
    echo "Creating group: $GROUP"
    groupadd "$GROUP"
fi

# Create user
echo "Creating user: $USERNAME"
if [ -n "$COMMENT" ]; then
    useradd -m -g "$GROUP" -s "$SHELL" -c "$COMMENT" "$USERNAME"
else
    useradd -m -g "$GROUP" -s "$SHELL" "$USERNAME"
fi

# Set password
read -sp "Enter password for $USERNAME: " PASSWORD
echo ""
echo "$USERNAME:$PASSWORD" | chpasswd

# Log success
echo "[$(date)] SUCCESS: Created user $USERNAME in group $GROUP" >> "$LOG_FILE"
echo "User $USERNAME created successfully"

exit 0
```

### Automated Backup Script

File: `/usr/local/bin/backup.sh`

```bash
#!/bin/bash
#
# Automated Backup Script
# Creates timestamped backups with rotation
#

set -euo pipefail

# Configuration
BACKUP_SOURCE="/etc"
BACKUP_DEST="/var/backups/config"
RETENTION_DAYS=30
MAX_BACKUPS=10
DATE_STAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="etc_backup_${DATE_STAMP}.tar.gz"
LOG_FILE="/var/log/backup.log"

# Logging function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Ensure backup directory exists
mkdir -p "$BACKUP_DEST"

log "Starting backup of $BACKUP_SOURCE"

# Create backup archive
if tar -czf "${BACKUP_DEST}/${BACKUP_FILE}" -C / "${BACKUP_SOURCE#/}" 2>/dev/null; then
    BACKUP_SIZE=$(du -sh "${BACKUP_DEST}/${BACKUP_FILE}" | cut -f1)
    log "Backup created successfully: ${BACKUP_FILE} (${BACKUP_SIZE})"
else
    log "ERROR: Backup creation failed"
    exit 1
fi

# Verify backup integrity
if tar -tzf "${BACKUP_DEST}/${BACKUP_FILE}" > /dev/null 2>&1; then
    log "Backup verification: OK"
else
    log "ERROR: Backup verification failed"
    rm -f "${BACKUP_DEST}/${BACKUP_FILE}"
    exit 1
fi

# Rotate old backups by age
DELETED_COUNT=$(find "$BACKUP_DEST" -name "etc_backup_*.tar.gz" -mtime +${RETENTION_DAYS} -delete -print | wc -l)
if [ "$DELETED_COUNT" -gt 0 ]; then
    log "Rotated $DELETED_COUNT old backup(s) (older than $RETENTION_DAYS days)"
fi

# Rotate old backups by count
BACKUP_COUNT=$(ls -1 "${BACKUP_DEST}"/etc_backup_*.tar.gz 2>/dev/null | wc -l)
if [ "$BACKUP_COUNT" -gt "$MAX_BACKUPS" ]; then
    EXCESS=$((BACKUP_COUNT - MAX_BACKUPS))
    ls -1t "${BACKUP_DEST}"/etc_backup_*.tar.gz | tail -n "$EXCESS" | xargs rm -f
    log "Removed $EXCESS excess backup(s) (limit: $MAX_BACKUPS)"
fi

# Generate backup summary
CURRENT_BACKUPS=$(ls -1 "${BACKUP_DEST}"/etc_backup_*.tar.gz 2>/dev/null | wc -l)
TOTAL_SIZE=$(du -sh "$BACKUP_DEST" | cut -f1)
log "Backup complete. Total backups: $CURRENT_BACKUPS, Total size: $TOTAL_SIZE"

exit 0
```

### Service Monitoring Script

File: `/usr/local/bin/monitor_services.sh`

```bash
#!/bin/bash
#
# Service Monitoring Script
# Monitors specified services and restarts if failed
#

set -euo pipefail

# Configuration
SERVICES=("sshd" "httpd" "named" "postfix")
CHECK_INTERVAL=60
MAX_RESTARTS=3
LOG_FILE="/var/log/service_monitor.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$LOG_FILE"
}

log "Service monitor started"

while true; do
    for service in "${SERVICES[@]}"; do
        if ! systemctl is-active "$service" > /dev/null 2>&1; then
            log "WARNING: Service $service is down"

            # Attempt restart
            RESTART_COUNT=0
            until systemctl start "$service" > /dev/null 2>&1; do
                ((RESTART_COUNT++))
                if [ "$RESTART_COUNT" -ge "$MAX_RESTARTS" ]; then
                    log "ERROR: Failed to restart $service after $MAX_RESTARTS attempts"
                    break
                fi
                log "Retry $RESTART_COUNT: Attempting to restart $service"
                sleep 5
            done

            if systemctl is-active "$service" > /dev/null 2>&1; then
                log "INFO: Service $service restarted successfully"
            fi
        fi
    done

    sleep "$CHECK_INTERVAL"
done
```

---

## Real-World Examples

### Scenario 1: Bulk User Creation from CSV

An administrator needs to create multiple user accounts from a CSV file.

```bash
#!/bin/bash
#
# Bulk User Creation from CSV File
# CSV format: username,fullname,group
#

set -euo pipefail

CSV_FILE="/tmp/users.csv"
LOG_FILE="/var/log/bulk_user_creation.log"

if [ ! -f "$CSV_FILE" ]; then
    echo "Error: CSV file not found: $CSV_FILE"
    exit 1
fi

SUCCESS_COUNT=0
FAIL_COUNT=0

# Skip header line, process each user
tail -n +2 "$CSV_FILE" | while IFS=',' read -r username fullname group; do
    # Skip empty lines
    if [ -z "$username" ]; then
        continue
    fi

    # Check if user exists
    if id "$username" > /dev/null 2>&1; then
        echo "SKIP: User $username already exists"
        echo "[$(date)] SKIP: $username already exists" >> "$LOG_FILE"
        continue
    fi

    # Create group if it does not exist
    if ! getent group "$group" > /dev/null 2>&1; then
        groupadd "$group"
        echo "CREATED GROUP: $group"
    fi

    # Create user
    if useradd -m -g "$group" -c "$fullname" -s /bin/bash "$username"; then
        echo "Created: $username ($fullname, $group)"
        echo "[$(date)] SUCCESS: Created $username" >> "$LOG_FILE"
    else
        echo "FAILED: Could not create $username"
        echo "[$(date)] FAILED: $username" >> "$LOG_FILE"
    fi
done

echo "Bulk user creation complete"
```

### Scenario 2: Log Rotation and Cleanup Script

A script that rotates logs, compresses old ones, and removes expired archives.

```bash
#!/bin/bash
#
# Custom Log Rotation Script
#

set -euo pipefail

LOG_DIR="/var/log/myapp"
MAX_SIZE_KB=10240        # 10 MB
KEEP_COUNT=5
MAX_AGE_DAYS=90

if [ ! -d "$LOG_DIR" ]; then
    echo "Log directory not found: $LOG_DIR"
    exit 1
fi

for logfile in "$LOG_DIR"/*.log; do
    if [ ! -f "$logfile" ]; then
        continue
    fi

    FILESIZE=$(stat -c%s "$logfile")
    MAX_SIZE_BYTES=$((MAX_SIZE_KB * 1024))

    if [ "$FILESIZE" -gt "$MAX_SIZE_BYTES" ]; then
        BASENAME=$(basename "$logfile")
        TIMESTAMP=$(date +%Y%m%d_%H%M%S)
        ROTATED="${logfile}.${TIMESTAMP}"

        # Rotate current log
        cp "$logfile" "$ROTATED"
        : > "$logfile"

        # Compress rotated log
        gzip "$ROTATED"

        echo "Rotated: $BASENAME (${FILESIZE} bytes)"
    fi

    # Remove old compressed logs
    find "$LOG_DIR" -name "${BASENAME}.*.gz" -mtime +${MAX_AGE_DAYS} -delete

    # Keep only the most recent N rotated logs
    ls -1t "$LOG_DIR"/${BASENAME}.*.gz 2>/dev/null | tail -n +$((KEEP_COUNT + 1)) | xargs rm -f 2>/dev/null || true
done

echo "Log rotation complete for $LOG_DIR"
```

### Scenario 3: Disk Space Alert Script

A script that monitors disk usage and sends alerts via email when thresholds are exceeded.

```bash
#!/bin/bash
#
# Disk Space Alert Script
#

set -euo pipefail

WARNING_THRESHOLD=75
CRITICAL_THRESHOLD=90
ADMIN_EMAIL="admin@example.com"

ALERT_NEEDED=false
ALERT_MESSAGE=""

# Check each mounted filesystem
while IFS= read -r line; do
    usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
    mount=$(echo "$line" | awk '{print $6}')
    available=$(echo "$line" | awk '{print $4}')

    if [ "$usage" -ge "$CRITICAL_THRESHOLD" ]; then
        ALERT_NEEDED=true
        ALERT_MESSAGE+="CRITICAL: $mount is ${usage}% full (${available} available)\n"
    elif [ "$usage" -ge "$WARNING_THRESHOLD" ]; then
        ALERT_NEEDED=true
        ALERT_MESSAGE+="WARNING: $mount is ${usage}% full (${available} available)\n"
    fi
done < <(df -h | grep -v tmpfs)

# Send alert if needed
if [ "$ALERT_NEEDED" = true ]; then
    echo -e "$ALERT_MESSAGE" | \
        mail -s "Disk Alert: $(hostname)" "$ADMIN_EMAIL" 2>/dev/null || \
        echo "Alert queued (mail not available):" | tee /tmp/disk_alert.txt
    echo -e "$ALERT_MESSAGE" > /tmp/disk_alert.txt
fi
```

---

## Troubleshooting

### Script Will Not Execute

**Problem**: Permission denied when running script

```bash
# Check file permissions
ls -la script.sh

# Add execute permission
chmod +x script.sh

# Verify shebang line
head -1 script.sh

# Check if specified shell exists
cat /etc/shells

# Run with explicit interpreter
bash script.sh
```

**Problem**: Bad interpreter error

```bash
# Check for Windows line endings (CRLF)
file script.sh
cat -A script.sh    # Look for ^M characters

# Convert to Unix line endings
sed -i 's/\r$//' script.sh
dos2unix script.sh

# Verify shebang points to valid shell
ls -la /bin/bash
```

### Variable and Expansion Issues

**Problem**: Unexpected behavior with variables

```bash
# Debug: print variable values
echo "DEBUG: variable='$variable'"

# Check for undefined variables
set -u    # Exit on undefined variable use

# Common mistake: spaces around =
WRONG= "value"     # Error
RIGHT="value"      # Correct

# Common mistake: unquoted variable with spaces
PATH_VAR="/path/with spaces"
ls $PATH_VAR       # Word splitting error
ls "$PATH_VAR"     # Correct

# Debug script execution
bash -x script.sh   # Show each command before execution
```

### Conditional Logic Errors

**Problem**: Conditional always evaluates to true/false

```bash
# Common mistake: using = instead of -eq for numbers
[ "5" = "5" ]      # Works (string comparison)
[ 5 -eq 5 ]        # Correct for numbers

# Common mistake: missing spaces in [ ]
[ "$a"-eq"$b" ]    # Error
[ "$a" -eq "$b" ]  # Correct - spaces required

# Common mistake: unquoted variable in string test
EMPTY=""
[ $EMPTY = "test" ]        # Error: missing operand
[ "$EMPTY" = "test" ]      # Correct

# Debug: check test result directly
[ -f /etc/hosts ]; echo $?   # 0 = true, 1 = false
```

### Loop Issues

**Problem**: for loop processes glob literally

```bash
# Problem: no .txt files exist, loop runs once with literal string
for file in /tmp/*.txt; do
    echo "$file"    # Prints "/tmp/*.txt"
done

# Solution: check if file exists
for file in /tmp/*.txt; do
    if [ -f "$file" ]; then
        echo "$file"
    fi
done

# Alternative: enable nullglob
shopt -s nullglob
for file in /tmp/*.txt; do
    echo "$file"    # Loop does not run if no matches
done
```

**Problem**: while read loop loses last line

```bash
# Problem: file without trailing newline loses last line
while read -r line; do
    echo "$line"
done < file.txt

# Solution: handle last line without newline
while IFS= read -r line || [ -n "$line" ]; do
    echo "$line"
done < file.txt
```

### Command Substitution Issues

**Problem**: Command substitution captures errors

```bash
# Problem: stderr mixed into variable
RESULT=$(ls /nonexistent)
echo "$RESULT"    # Shows error message in variable

# Solution: redirect stderr separately
RESULT=$(ls /nonexistent 2>/dev/null)
if [ $? -ne 0 ]; then
    echo "Command failed"
fi

# Debug: check exit status
RESULT=$(some_command)
EXIT_STATUS=$?
echo "Result: $RESULT, Exit: $EXIT_STATUS"
```

### Debugging Techniques

```bash
# Run script with debugging
bash -x script.sh              # Trace execution
bash -x script.sh 2>/tmp/debug.log   # Save trace to file

# Add debug output to script
set -x                         # Enable tracing
# ... script logic ...
set +x                         # Disable tracing

# Check syntax without executing
bash -n script.sh              # Syntax check only

# Add verbose logging
LOG.setLevel DEBUG
echo "DEBUG: Variable value is '$var'" >&2
```

---

## Verification Procedures

### Script Execution Verification

```bash
# Verify script syntax
bash -n script.sh

# Verify script is executable
ls -la script.sh
test -x script.sh && echo "Executable"

# Verify shebang line
head -1 script.sh
file script.sh

# Test script execution
./script.sh
echo "Exit status: $?"
```

### Variable Verification

```bash
# Create test script
cat > /tmp/test_vars.sh << 'EOF'
#!/bin/bash
NAME="test_user"
COUNT=5
ARRAY=("one" "two" "three")

echo "String: $NAME"
echo "Number: $COUNT"
echo "Array length: ${#ARRAY[@]}"
echo "Array contents: ${ARRAY[@]}"
echo "Substring: ${NAME:0:4}"
EOF

chmod +x /tmp/test_vars.sh
/tmp/test_vars.sh
```

### Conditional Verification

```bash
# Test conditional logic
cat > /tmp/test_cond.sh << 'EOF'
#!/bin/bash

# Test file checks
[ -f /etc/hosts ] && echo "hosts file exists" || echo "hosts file missing"

# Test numeric comparison
[ 42 -gt 10 ] && echo "42 > 10" || echo "42 <= 10"

# Test string comparison
[ "hello" = "hello" ] && echo "strings match" || echo "strings differ"

# Test empty string
EMPTY=""
[ -z "$EMPTY" ] && echo "string is empty" || echo "string has content"
EOF

chmod +x /tmp/test_cond.sh
/tmp/test_cond.sh
```

### Loop Verification

```bash
# Test for loop
for i in {1..3}; do echo "Iteration $i"; done

# Test while loop
counter=0; while [ "$counter" -lt 3 ]; do echo "Count: $counter"; ((counter++)); done

# Test file processing
echo -e "line1\nline2\nline3" > /tmp/test_lines.txt
while IFS= read -r line; do echo "Read: $line"; done < /tmp/test_lines.txt
```

### Script Input Verification

```bash
# Test positional parameters
cat > /tmp/test_args.sh << 'EOF'
#!/bin/bash
echo "Script: $0"
echo "Arg1: $1"
echo "Arg2: $2"
echo "Count: $#"
echo "All: $@"
EOF

chmod +x /tmp/test_args.sh
/tmp/test_args.sh hello world third
```

---

## RHCSA Exam Notes

### Exam Relevance

Shell scripting appears as a practical task on the RHCSA exam. You will likely be asked to write a script that performs a specific administrative task, such as creating users, checking services, or processing files.

### Key Exam Points

1. **Script Structure**: Always include the shebang (`#!/bin/bash`), make the script executable (`chmod +x`), and use meaningful variable names.

2. **Conditional Logic**: Know how to use `if/then/elif/else/fi` with `[ ]` for file tests, numeric comparisons, and string comparisons. The `[[ ]]` construct is acceptable and often safer.

3. **For Loops**: The `for` loop is the most commonly tested loop construct. Know how to iterate over lists, files, and command output.

4. **Positional Parameters**: Know `$1`, `$2`, `$#`, and `$@`. The exam may ask you to write a script that accepts arguments.

5. **Command Substitution**: Know the `$()` syntax for capturing command output. This is essential for storing command results in variables.

6. **Exit Codes**: Scripts should return appropriate exit codes. Use `exit 0` for success and `exit 1` (or higher) for failure.

### Common Exam Task Patterns

- Write a script that creates users from a list
- Write a script that checks if a service is running and restarts it
- Write a script that processes files in a directory
- Write a script that generates a report from system information
- Write a script that accepts command-line arguments for configuration

### Common Exam Traps

- Forgetting the shebang line (`#!/bin/bash`)
- Forgetting to add execute permission (`chmod +x`)
- Using `[` without spaces (`[ "$a"= "$b" ]` fails)
- Not quoting variables that may contain spaces
- Confusing `-eq` (numeric) with `=` (string) comparison
- Forgetting `fi` to close `if` statements
- Forgetting `done` to close loops
- Using `$0` for first argument (it is the script name; `$1` is the first argument)

### Time Management Tips

- Write the script structure first (shebang, variables, logic), then fill in details
- Test each section of the script as you write it
- Use `bash -n script.sh` to check syntax before submitting
- Keep scripts simple; the exam rewards correctness over cleverness
- Comment your scripts briefly to clarify intent

### Exam Environment Notes

- Scripts can be placed in any writable directory
- The exam specifies the exact script path and expected behavior
- Test data (files, user lists) will be provided
- Scripts must work when executed as specified in the task
- Use `bash` or `/bin/bash` as the interpreter

---

## Chapter Summary

This chapter covered the fundamentals of shell scripting in RHEL 10, including script structure, variables, conditional execution with `if/test/case`, looping with `for/while/until`, processing command-line arguments, and capturing command output through command substitution. You learned how to write robust scripts with input validation, error handling, and logging.

Shell scripting is one of the most practical skills for a Linux administrator. Scripts automate repetitive tasks, enforce consistency, and serve as documentation for operational procedures. On the RHCSA exam, scripting tasks test your ability to combine shell features into working solutions that solve real administrative problems.

---

## Quick Reference

### Script Structure

```bash
#!/bin/bash
set -euo pipefail       # Strict mode
# Your script here
exit 0                  # Success
```

### Variables

```bash
NAME="value"            # Define
echo "$NAME"            # Use
VALUE=$(command)        # Command substitution
READ -p "Prompt: " VAR # User input
${#VAR}                 # String length
${VAR:0:5}              # Substring
${VAR/pattern/repl}     # Replace
${VAR:-default}         # Default value
```

### Special Variables

```bash
$0      # Script name
$1-$9   # Positional arguments
$#      # Argument count
$@      # All arguments
$?      # Last exit status
$$      # Process ID
```

### Conditionals

```bash
if [ condition ]; then
    # true branch
elif [ condition ]; then
    # alternative
else
    # false branch
fi

# File tests
[ -f file ]   [ -d dir ]   [ -e path ]
[ -r file ]   [ -w file ]   [ -x file ]

# Numeric tests
[ a -eq b ]   [ a -ne b ]   [ a -gt b ]
[ a -lt b ]   [ a -ge b ]   [ a -le b ]

# String tests
[ "$a" = "$b" ]   [ "$a" != "$b" ]
[ -z "$a" ]       [ -n "$a" ]
```

### Loops

```bash
# For loop
for item in list; do
    # body
done

for i in {1..10}; do
    # body
done

# While loop
while [ condition ]; do
    # body
done

# Until loop
until [ condition ]; do
    # body
done

# Loop control
break      # Exit loop
continue   # Next iteration
```

### Arithmetic

```bash
$((a + b))          # Addition
$((a * b))          # Multiplication
$((a % b))          # Modulo
((counter++))       # Increment
((counter--))       # Decrement
```

### Debugging

```bash
bash -n script.sh   # Syntax check
bash -x script.sh   # Trace execution
set -x              # Enable tracing in script
set +x              # Disable tracing
```

### case Statement

```bash
case "$variable" in
    pattern1)
        # commands
        ;;
    pattern2)
        # commands
        ;;
    *)
        # default
        ;;
esac
```

---

## Review Questions

1. What is the purpose of the shebang line (`#!/bin/bash`) in a shell script, and what happens if it is missing?

2. What is the difference between `[ ]` and `[[ ]]` in Bash conditional statements?

3. How do you capture the output of a command into a variable, and why is `$()` preferred over backticks?

4. What do the special variables `$#`, `$@`, and `$?` represent?

5. Write a conditional that checks if a file exists AND is readable.

6. What is the difference between `for i in {1..5}` and `for i in $(seq 1 5)`?

7. How does the `break` statement differ from the `continue` statement in a loop?

8. What does `set -euo pipefail` do in a Bash script?

9. How would you write a script that accepts a username as its first argument and validates that it is not empty?

10. What is the purpose of `IFS= read -r line` when reading a file line by line?

11. How do you check the exit status of the last command executed in a script?

12. What would the following script output?
```bash
#!/bin/bash
for file in /nonexistent/*.txt; do
    echo "$file"
done
```

13. How do you make a shell script executable?

14. What is the difference between `$*` and `$@` when referencing script arguments?

15. How would you write a `while` loop that reads a file and skips empty lines?

---

## Answers

1. The shebang line (`#!/bin/bash`) tells the kernel which interpreter to use when executing the script directly (e.g., `./script.sh`). Without it, the script runs in the current shell, which may not be Bash, leading to unexpected behavior if the script uses Bash-specific features. If executed with `bash script.sh`, the shebang is not required since the interpreter is specified explicitly.

2. `[ ]` is the POSIX-compatible test command (also available as `test`). It performs basic file, numeric, and string tests but is subject to word splitting and pathname expansion. `[[ ]]` is a Bash keyword that provides enhanced features: pattern matching with `==`, regex matching with `=~`, logical operators `&&` and `||` inside the brackets, and no word splitting or pathname expansion. `[[ ]]` is safer and more powerful but is Bash-specific (not portable to other shells).

3. Use `variable=$(command)` to capture command output. The `$()` syntax is preferred over backticks (`` `command` ``) because it is easier to read, nests cleanly (`$(command $(inner))` vs `` `command \`inner\`` ``), and handles complex commands with pipes and redirections more reliably. Both syntaxes capture stdout, strip trailing newlines, and assign the result to the variable.

4. `$#` holds the number of positional arguments passed to the script. `$@` expands to all positional arguments as separate, quoted words (preserving spaces within arguments). `$?` holds the exit status of the most recently executed command (0 for success, non-zero for failure).

5. ```bash
   if [ -f "/path/to/file" ] && [ -r "/path/to/file" ]; then
       echo "File exists and is readable"
   fi
   ```
   Alternatively with `[[ ]]`: `if [[ -f "/path/to/file" && -r "/path/to/file" ]]; then`

6. Functionally, both produce the same output (numbers 1 through 5). However, `{1..5}` is a Bash brace expansion that happens before command execution (faster, no subshell). `$(seq 1 5)` runs the `seq` command in a subshell and captures its output (slightly slower, but more flexible for complex sequences with steps). For simple ranges, `{1..5}` is preferred.

7. `break` immediately exits the entire loop, continuing execution with the statement after `done`. `continue` skips the remaining statements in the current iteration and jumps to the next iteration of the loop. For example, `break` stops processing entirely when a condition is met, while `continue` skips one item and keeps processing the rest.

8. `set -euo pipefail` enables strict error handling: `-e` causes the script to exit immediately if any command returns a non-zero exit status; `-u` treats undefined variables as errors and exits; `-o pipefail` makes a pipeline fail if any command in the pipeline fails (not just the last one). Together, these options make scripts fail fast and catch errors early.

9. ```bash
   #!/bin/bash
   if [ "$#" -lt 1 ]; then
       echo "Usage: $0 <username>"
       exit 1
   fi
   USERNAME="$1"
   if [ -z "$USERNAME" ]; then
       echo "Error: Username cannot be empty"
       exit 1
   fi
   echo "Username: $USERNAME"
   ```

10. `IFS=` prevents leading and trailing whitespace from being trimmed from each line. `read -r` prevents backslash interpretation (so backslashes in the file are read literally). Together, `IFS= read -r line` ensures each line is read exactly as it appears in the file, preserving all whitespace and special characters.

11. Use the `$?` special variable immediately after the command. For example: `some_command; echo "Exit status: $?"`. The value of `$?` is overwritten by every command, so it must be checked or saved immediately after the command of interest.

12. The script would output: `/nonexistent/*.txt` (the literal glob pattern). Since no `.txt` files exist in `/nonexistent/`, the glob does not expand, and Bash passes the literal string to the loop. The loop runs exactly once with `file` set to the unexpanded glob pattern. To prevent this, check `[ -f "$file" ]` inside the loop or enable `nullglob`.

13. Use `chmod +x script.sh` to add execute permission. Alternatively, `chmod 755 script.sh` sets read, write, and execute for the owner, and read and execute for group and others. The script must also have a valid shebang line to execute directly.

14. Both `$*` and `$@` expand to all positional arguments, but differently. `$@` expands each argument as a separate word, preserving spaces and quoting (preferred in most cases). `$*` expands all arguments as a single word, separated by the first character of `IFS` (default: space). In practice, `"$@"` is almost always what you want when passing arguments to other commands.

15. ```bash
    while IFS= read -r line || [ -n "$line" ]; do
        if [ -z "$line" ]; then
            continue
        fi
        echo "Processing: $line"
    done < /path/to/file
    ```
    The `|| [ -n "$line" ]` handles files without a trailing newline. The `[ -z "$line" ]` test checks if the line is empty, and `continue` skips to the next iteration.
