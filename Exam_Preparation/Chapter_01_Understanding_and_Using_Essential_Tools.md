# Chapter 1: Understanding and Using Essential Tools

## Learning Objectives

By the end of this chapter, you will be able to:

- Access a shell prompt and issue commands with correct syntax
- Use input-output redirection (`>`, `>>`, `|`, `2>`, etc.)
- Use `grep` and regular expressions to analyze text
- Access remote systems securely using SSH
- Log in and switch users in multi-user targets
- Archive, compress, unpack, and uncompress files using `tar`, `gzip`, and `bzip2`
- Create and edit text files from the command line
- Create, delete, copy, and move files and directories
- Create hard and soft links
- List, set, and change standard `ugo/rwx` permissions
- Locate, read, and use system documentation including `man`, `info`, and files in `/usr/share/doc`

---

## Concepts and Theory

### The Linux Shell

The shell is a command-line interpreter that serves as your primary interface with the Linux operating system. It reads commands you type, interprets them, and passes them to the kernel for execution. In RHEL 10, the default shell is **Bash** (Bourne Again Shell), located at `/bin/bash`. Bash provides powerful features including command history, tab completion, variable expansion, scripting capabilities, and job control.

When you log into a RHEL 10 system, the PAM (Pluggable Authentication Modules) subsystem authenticates your credentials, and your configured shell starts as an interactive session. The shell reads configuration files in a specific order: `/etc/profile` (system-wide), then `~/.bash_profile` or `~/.bashrc` (user-specific). These files define environment variables, aliases, and prompt customization.

### Command Syntax

Every Linux command follows a consistent syntax structure:

```
command [options] [arguments]
```

- **Command**: The program or utility to execute (e.g., `ls`, `cp`, `grep`)
- **Options**: Modify command behavior, prefixed with `-` (short) or `--` (long)
- **Arguments**: Files, directories, or values the command operates on

Options can be combined: `ls -la` is equivalent to `ls -l -a`. Long options improve readability: `ls --all --long`.

### Input/Output/Redirection

Every process in Linux has three standard data streams:

- **Standard Input (stdin, file descriptor 0)**: Where a command reads input, typically from the keyboard
- **Standard Output (stdout, file descriptor 1)**: Where a command writes normal output, typically to the terminal
- **Standard Error (stderr, file descriptor 2)**: Where a command writes error messages

Redirection operators allow you to change where these streams go:

| Operator | Description |
|----------|-------------|
| `>` | Redirect stdout to a file (overwrites) |
| `>>` | Redirect stdout to a file (appends) |
| `2>` | Redirect stderr to a file |
| `&>` | Redirect both stdout and stderr to a file |
| `<` | Redirect a file as stdin |
| `\|` | Pipe stdout of one command to stdin of another |

### The Pipe Operator

The pipe (`|`) connects the stdout of one command directly to the stdin of the next command, enabling powerful command chains without intermediate files. For example, `cat /etc/passwd | grep bash` filters the passwd file for users with bash shells.

### Regular Expressions

Regular expressions (regex) are patterns used to match character combinations in text. Linux tools like `grep`, `sed`, and `awk` use regex for text processing.

**Basic Regular Expression Metacharacters:**

| Pattern | Description | Example |
|---------|-------------|---------|
| `.` | Any single character | `c.t` matches `cat`, `cut`, `cot` |
| `*` | Zero or more of preceding character | `colou*r` matches `color`, `colour` |
| `^` | Start of line | `^root` matches lines starting with `root` |
| `$` | End of line | `bash$` matches lines ending with `bash` |
| `[]` | Character class | `[aeiou]` matches any vowel |
| `[0-9]` | Numeric range | `[0-9][0-9]` matches two digits |
| `\` | Escape character | `\$` matches a literal dollar sign |

**Extended Regular Expressions (grep -E):**

| Pattern | Description | Example |
|---------|-------------|---------|
| `+` | One or more of preceding | `colou+r` matches `colour` |
| `?` | Zero or one of preceding | `colou?r` matches `color`, `colour` |
| `\|` | Alternation | `cat\|dog` matches `cat` or `dog` |
| `()` | Grouping | `(ab)+` matches `ab`, `abab` |
| `{n}` | Exact count | `[0-9]{3}` matches exactly 3 digits |

### SSH (Secure Shell)

SSH is an encrypted protocol for securely accessing remote systems. It replaces insecure protocols like Telnet and FTP. SSH encrypts all traffic between client and server, including authentication credentials and session data.

SSH supports two authentication methods:

1. **Password authentication**: Entering a password for each connection
2. **Key-based authentication**: Using cryptographic key pairs for passwordless, more secure login

The SSH client connects to the SSH daemon (`sshd`) running on the remote server, typically on port 22.

### Hard Links and Soft (Symbolic) Links

Linux provides two types of links that create alternative paths to files:

**Hard Links**: A hard link is an additional directory entry pointing to the same inode (the actual file data on disk). Hard links share the same inode number as the original file. They cannot cross filesystem boundaries and cannot link to directories. The file data persists as long as at least one hard link exists.

**Soft (Symbolic) Links**: A soft link is a special file containing a path reference to another file or directory. It has its own inode. If the target is deleted, the soft link becomes "broken." Soft links can cross filesystem boundaries and can point to directories.

### File Permissions

Linux uses a permission model based on three classes of users and three types of permissions:

**User Classes:**

- **User (u)**: The file owner
- **Group (g)**: Members of the file's group
- **Others (o)**: Everyone else

**Permission Types:**

- **Read (r, 4)**: View file contents / list directory contents
- **Write (w, 2)**: Modify file contents / create or delete files in directory
- **Execute (x, 1)**: Run file as program / enter directory

Permissions are represented symbolically (`rwxr-xr--`) or numerically (octal: `754`). The numeric system adds values: `7 = rwx`, `6 = rw-`, `5 = r-x`, `4 = r--`, `3 = -wx`, `2 = -w-`, `1 = --x`, `0 = ---`.

### System Documentation

RHEL 10 provides multiple documentation sources:

- **man pages**: Online reference manuals for commands and system calls, organized into numbered sections
- **info pages**: Hyperlinked documentation, often more detailed than man pages
- **/usr/share/doc**: Package-specific documentation, examples, and license files

---

## How It Works Internally

### Shell Command Processing

When you type a command and press Enter, the shell performs these steps:

1. **Read**: The shell reads the input line from stdin
2. **Parse**: The shell tokenizes the line, identifying the command, options, and arguments. It performs variable expansion, wildcard (glob) expansion, and handles redirection operators
3. **Fork**: The shell creates a child process using the `fork()` system call
4. **Exec**: The child process replaces itself with the requested program using `exec()`
5. **Search Path**: The shell searches for the command in directories listed in the `$PATH` environment variable, in order
6. **Wait**: The parent shell waits for the child process to complete, then displays the prompt again

Built-in commands (like `cd`, `echo`, `export`) are handled directly by the shell without forking a new process.

### How Redirection Works

Redirection manipulates file descriptors before the command executes. When the shell encounters `>`, it opens (or creates) the specified file, truncates it, and assigns it to file descriptor 1 (stdout). When the command runs, its output goes to this file instead of the terminal.

The `>>` operator opens the file in append mode, writing to the end rather than truncating. Error redirection `2>` works identically but targets file descriptor 2.

### How Pipes Work Internally

A pipe creates a unidirectional communication channel between two processes. The shell creates a pipe using the `pipe()` system call, which returns two file descriptors: one for reading and one for writing. The stdout of the first command is connected to the write end, and the stdin of the second command is connected to the read end. Data flows through a kernel buffer (typically 64 KB on Linux).

### How SSH Encryption Works

SSH uses a combination of encryption methods:

1. **Key Exchange**: The Diffie-Hellman algorithm establishes a shared secret without transmitting it, even over an unencrypted channel
2. **Server Authentication**: The server proves its identity using its host key (RSA, ECDSA, or Ed25519)
3. **User Authentication**: The client authenticates via password or public key
4. **Session Encryption**: All subsequent traffic is encrypted using symmetric encryption (AES, ChaCha20)

When you connect to a new server for the first time, SSH stores the server's host key fingerprint in `~/.ssh/known_hosts`. On subsequent connections, SSH verifies the fingerprint matches, protecting against man-in-the-middle attacks.

### How Links Work at the Filesystem Level

Every file on a Linux filesystem has an **inode** (index node), which stores metadata: permissions, ownership, timestamps, and pointers to data blocks. A directory entry maps a filename to an inode number.

A **hard link** simply adds another directory entry pointing to the same inode. The inode maintains a link count; the file data is only freed when this count reaches zero. Running `ls -i` displays inode numbers, revealing when two names share the same inode.

A **soft link** is a separate inode of type "symbolic link." Its data blocks contain the path string of the target. The kernel resolves this path when the link is accessed.

### How Permission Checks Work

When a process attempts to access a file, the kernel performs a permission check in this order:

1. If the process is running as **root** (UID 0), most permission checks are bypassed (read and execute are always allowed; write depends on file immutability)
2. If the process UID matches the file owner, the **user** permission bits are checked
3. If the process GID matches any of the file's group IDs, the **group** permission bits are checked
4. Otherwise, the **others** permission bits are checked

For directories, the execute bit (`x`) is required to access files within that directory, even if read permission is granted.

---

## Commands and Administration Tasks

### Accessing a Shell Prompt and Issuing Commands

#### Bash Shell

The default shell in RHEL 10 is Bash. After logging in, you receive an interactive prompt. The prompt format is controlled by the `PS1` environment variable.

```bash
# Display current shell
echo $SHELL

# Display shell version
bash --version

# List available shells
cat /etc/shells
```

#### Essential Command Syntax Rules

```bash
# Commands are case-sensitive
ls /home          # Correct
LS /home          # Error: command not found

# Use full paths when command is not in PATH
/usr/bin/ls -la

# View command type (builtin, alias, file)
type cd
type ls
which ls
```

#### Command History and Navigation

```bash
# View command history
history

# Search history (press Ctrl+R, then type)

# Use specific history entry
!123              # Run command at history number 123
!ls               # Run most recent command starting with 'ls'

# Navigation shortcuts
# Ctrl+A - Move to beginning of line
# Ctrl+E - Move to end of line
# Ctrl+U - Clear line before cursor
# Ctrl+K - Clear line after cursor
# Ctrl+W - Delete word before cursor
# Tab - Auto-complete commands and filenames
```

#### Tab Completion

```bash
# Press Tab to complete commands, filenames, and paths
ls /et<Tab>       # Completes to /etc/
us<Tab>eradd     # Completes to useradd
```

### Input/Output Redirection

#### Redirecting Standard Output

```bash
# Overwrite file with command output
ls -la > /tmp/filelist.txt

# Append to file
echo "backup completed" >> /var/log/backup.log

# Redirect multiple commands
{ ls -la; df -h; free -m; } > /tmp/system_report.txt
```

#### Redirecting Standard Error

```bash
# Redirect errors only
ls /nonexistent 2> /tmp/errors.txt

# Redirect errors to /dev/null (discard)
ls /nonexistent 2>/dev/null

# Redirect both stdout and stderr
command > /tmp/output.txt 2>&1
# Or using the shorthand in Bash
command &> /tmp/output.txt
```

#### Redirecting Standard Input

```bash
# Read input from file
sort < /tmp/unsorted.txt > /tmp/sorted.txt

# Pass input to a command
wc -l < /etc/passwd
```

#### Using Pipes

```bash
# Chain multiple commands
cat /etc/passwd | grep bash | cut -d: -f1

# Count lines containing a pattern
grep -c "error" /var/log/messages

# View large files page by page
cat /var/log/audit/audit.log | less

# Sort and remove duplicates
cat /etc/group | sort -u
```

### Using grep and Regular Expressions

#### Basic grep Usage

```bash
# Search for pattern in file
grep "root" /etc/passwd

# Case-insensitive search
grep -i "error" /var/log/messages

# Show line numbers
grep -n "WARNING" /var/log/syslog

# Count matching lines
grep -c "Failed password" /var/log/secure

# Invert match (show non-matching lines)
grep -v "^#" /etc/fstab

# Search recursively in directory
grep -r "listen_address" /etc/
```

#### grep with Regular Expressions

```bash
# Basic regex: find lines starting with root
grep "^root" /etc/passwd

# Find lines ending with /bin/bash
grep "/bin/bash$" /etc/passwd

# Match any single character
grep "r..t" /etc/words

# Match character class
grep "[0-9][0-9][0-9]" /etc/hosts

# Extended regex with -E flag
grep -E "^(root|admin)" /etc/passwd
grep -E "[0-9]{3}\.[0-9]{3}" /var/log/messages
grep -E "warning|error|critical" /var/log/messages
```

#### Combining grep with Pipes

```bash
# Filter process list
ps aux | grep sshd

# Search configuration files
cat /etc/ssh/sshd_config | grep -v "^#" | grep -v "^$"

# Analyze disk usage
df -h | grep -v "tmpfs"

# Find failed SSH attempts
grep "Failed password" /var/log/secure | grep -oP "(\d+\.){3}\d+" | sort | uniq -c | sort -rn
```

### Accessing Remote Systems Using SSH

#### Basic SSH Connection

```bash
# Connect to remote system
ssh username@remote_host

# Connect with specific port
ssh -p 2222 username@remote_host

# Run a single command remotely
ssh username@remote_host "df -h"

# Verbose mode for troubleshooting
ssh -v username@remote_host
```

#### SSH Key-Based Authentication

```bash
# Generate SSH key pair (Ed25519 recommended)
ssh-keygen -t ed25519 -C "admin@rhel10"

# Copy public key to remote server
ssh-copy-id username@remote_host

# Test key-based login
ssh username@remote_host
```

#### SSH Configuration

```bash
# Edit SSH client configuration
vi ~/.ssh/config

# Example ~/.ssh/config content:
# Host webserver
#     HostName 192.168.1.100
#     User admin
#     Port 22
#     IdentityFile ~/.ssh/webserver_key
```

#### SSH Security Options

```bash
# Limit authentication attempts
ssh -o NumberOfPasswordPrompts=2 username@remote_host

# Set connection timeout
ssh -o ConnectTimeout=10 username@remote_host

# Disable X11 forwarding
ssh -x username@remote_host
```

### Logging In and Switching Users

#### Switching Users with su

```bash
# Switch to another user (requires target user's password)
su - username

# Switch to root
su -

# Run a single command as another user
su - username -c "ls -la /root"
```

#### Switching Users with sudo

```bash
# Run command as root
sudo command

# Switch to root shell
sudo -i

# Run command as specific user
sudo -u username command

# List sudo privileges
sudo -l
```

#### Multi-User Targets

```bash
# Check current runlevel/target
systemctl get-default

# List available targets
systemctl list-units --type=target --state=active

# Switch to multi-user target (CLI mode)
sudo systemctl isolate multi-user.target

# Switch to graphical target
sudo systemctl isolate graphical.target
```

### Archiving and Compressing Files

#### Using tar

```bash
# Create a tar archive
tar -cf archive.tar /path/to/files

# Create compressed archive with gzip
tar -czf archive.tar.gz /path/to/files

# Create compressed archive with bzip2
tar -cjf archive.tar.bz2 /path/to/files

# Extract archive
tar -xf archive.tar

# Extract to specific directory
tar -xf archive.tar -C /destination/path

# List archive contents
tar -tf archive.tar

# Extract with verbose output
tar -xvf archive.tar.gz

# Add files to existing archive
tar -rf archive.tar newfile.txt

# Exclude patterns
tar -czf backup.tar.gz /home --exclude='*.cache' --exclude='*.tmp'
```

**tar Option Reference:**

| Option | Description |
|--------|-------------|
| `-c` | Create archive |
| `-x` | Extract archive |
| `-t` | List contents |
| `-f` | Specify archive file |
| `-v` | Verbose output |
| `-z` | Compress/decompress with gzip |
| `-j` | Compress/decompress with bzip2 |
| `-C` | Change to directory before operation |

#### Using gzip and bzip2 Directly

```bash
# Compress a file with gzip
gzip filename.txt        # Creates filename.txt.gz, removes original

# Compress keeping original
gzip -k filename.txt

# Decompress gzip file
gunzip filename.txt.gz
zcat filename.txt.gz     # View without decompressing

# Compress with bzip2
bzip2 filename.txt       # Creates filename.txt.bz2

# Decompress bzip2 file
bunzip2 filename.txt.bz2
bzcat filename.txt.bz2   # View without decompressing

# Compression level (1-9, default 6)
gzip -9 filename.txt
bzip2 -9 filename.txt
```

### Creating and Editing Text Files

#### Creating Files

```bash
# Create empty file
touch filename.txt

# Create file with content using echo
echo "Hello World" > filename.txt

# Create file with multiple lines using heredoc
cat > filename.txt << EOF
Line 1
Line 2
Line 3
EOF
```

#### Editing with vi/vim

```bash
# Open file in vi
vi filename.txt

# Essential vi commands:
# i          - Enter insert mode
# Esc        - Exit insert mode
# :w         - Save file
# :q         - Quit
# :wq        - Save and quit
# :q!        - Quit without saving
# dd         - Delete current line
# yy         - Yank (copy) current line
# p          - Paste after cursor
# /pattern   - Search for pattern
# n          - Next search result
# u          - Undo
```

#### Editing with nano

```bash
# Open file in nano
nano filename.txt

# Essential nano shortcuts:
# Ctrl+O - Save file
# Ctrl+X - Exit
# Ctrl+W - Search
# Ctrl+K - Cut line
# Ctrl+U - Paste
# Ctrl+_  - Go to line number
```

### File and Directory Operations

#### Creating Files and Directories

```bash
# Create a single directory
mkdir /path/to/newdir

# Create nested directories
mkdir -p /path/to/nested/directory

# Create directory with specific permissions
mkdir -m 750 /path/to/secure/dir

# Create multiple directories
mkdir dir1 dir2 dir3
```

#### Copying Files and Directories

```bash
# Copy a file
cp source.txt /destination/

# Copy with verbose output
cp -v source.txt /destination/

# Copy directory recursively
cp -r /source/dir /destination/

# Preserve attributes (permissions, timestamps, ownership)
cp -p source.txt /destination/
cp -a /source/dir /destination/   # Archive mode (preserves all)

# Copy multiple files
cp file1.txt file2.txt file3.txt /destination/

# Interactive copy (prompt before overwrite)
cp -i source.txt /destination/
```

#### Moving and Renaming

```bash
# Move a file
mv source.txt /new/location/

# Rename a file
mv oldname.txt newname.txt

# Move directory
mv /source/dir /new/location/

# Interactive mode
mv -i source.txt /destination/
```

#### Deleting Files and Directories

```bash
# Delete a file
rm filename.txt

# Force delete without confirmation
rm -f filename.txt

# Delete directory and contents recursively
rm -r /path/to/directory

# Force delete directory recursively
rm -rf /path/to/directory

# Delete with interactive prompt
rm -i filename.txt
```

#### Viewing File Contents

```bash
# Display entire file
cat filename.txt

# View file page by page
less filename.txt

# View first/last lines
head -n 20 filename.txt
tail -n 20 filename.txt

# Follow file in real time
tail -f /var/log/messages

# Display file with line numbers
nl filename.txt
cat -n filename.txt
```

### Creating Hard and Soft Links

#### Hard Links

```bash
# Create a hard link
ln source.txt hardlink.txt

# Verify hard link (same inode number)
ls -li source.txt hardlink.txt

# Hard links share inode and link count increases
stat source.txt
```

#### Soft (Symbolic) Links

```bash
# Create a symbolic link
ln -s /path/to/target linkname

# Create symbolic link to directory
ln -s /var/log /home/user/loglink

# Verify symbolic link
ls -l linkname
ls -li /path/to/target linkname

# Readlink shows where symlink points
readlink linkname
```

### Setting and Changing Permissions

#### Using chmod (Symbolic Mode)

```bash
# Add execute permission for owner
chmod u+x script.sh

# Remove write permission for group and others
chmod go-w file.txt

# Set read-only for group and others
chmod go=r file.txt

# Add execute for owner and group
chmod ug+x script.sh

# Remove all permissions for others
chmod o-rwx file.txt
```

#### Using chmod (Numeric Mode)

```bash
# Set permissions to rwxr-xr-- (754)
chmod 754 file.txt

# Set permissions to rw-r--r-- (644)
chmod 644 file.txt

# Set permissions to rwxr-xr-x (755)
chmod 755 directory/

# Set restrictive permissions (owner only)
chmod 600 sensitive.txt

# Recursively change directory permissions
chmod -R 750 /path/to/directory
```

#### Using chown and chgrp

```bash
# Change file owner
chown newuser file.txt

# Change file group
chgrp newgroup file.txt

# Change both owner and group
chown newuser:newgroup file.txt

# Change owner only (keep existing group)
chown newuser: file.txt

# Recursively change ownership
chown -R user:group /path/to/directory
```

### Using System Documentation

#### man Pages

```bash
# View man page
man command_name

# Search within man page (press /, type pattern, press Enter)

# Search man page names and descriptions
man -k keyword
apropos keyword

# View specific section
man 2 open         # Section 2: System calls
man 5 fstab        # Section 5: File formats

# man page sections:
# 1 - User commands
# 2 - System calls
# 3 - Library functions
# 4 - Special files (devices)
# 5 - File formats and conventions
# 6 - Games
# 7 - Miscellaneous
# 8 - System administration tools

# View man page in another language
man -L de command_name
```

#### info Pages

```bash
# View info documentation
info command_name

# info navigation:
# Enter - Follow menu item
# Space - Next page
# b - Previous page
# d - Go to directory
# u - Go up one level
# q - Quit
```

#### Package Documentation

```bash
# List documentation for installed packages
ls /usr/share/doc/

# View package-specific documentation
ls /usr/share/doc/package-name/

# View changelog
cat /usr/share/doc/package-name/CHANGELOG
```

---

## Configuration Examples

### SSH Client Configuration File

File: `~/.ssh/config`

```
# SSH client configuration
# Apply to all hosts
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    HashKnownHosts yes

# Web server shortcut
Host web01
    HostName 192.168.1.10
    User admin
    Port 22
    IdentityFile ~/.ssh/web01_ed25519

# Database server
Host db01
    HostName 192.168.1.20
    User dbadmin
    Port 2222
    IdentityFile ~/.ssh/db01_ed25519

# Jump host configuration
Host internal-*
    ProxyJump web01
    User admin
```

### SSH Server Configuration

File: `/etc/ssh/sshd_config`

```
# SSH daemon configuration for RHEL 10
Port 22
ListenAddress 0.0.0.0
ListenAddress ::

# Key-based authentication
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Disable root login
PermitRootLogin no

# Disable password authentication (after keys work)
PasswordAuthentication no

# Security settings
MaxAuthTries 3
MaxSessions 5
LoginGraceTime 60

# Disable empty passwords
PermitEmptyPasswords no

# Logging
LogLevel VERBOSE

# Allowed users (optional, restrict access)
# AllowUsers admin deployer
```

### Bash Profile Configuration

File: `~/.bashrc`

```bash
# User-specific bash configuration

# History settings
HISTSIZE=10000
HISTFILESIZE=20000
HISTCONTROL=ignoreboth
PROMPT_COMMAND="history -a"

# Aliases for safety
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
alias ll='ls -la --color=auto'
alias gs='grep --color=auto -rni'

# Custom prompt
PS1='[\u@\h \W]\$ '

# Export common paths
export PATH=$PATH:/usr/local/bin
```

### tar Backup Script

File: `/usr/local/bin/backup-config.sh`

```bash
#!/bin/bash
# Backup system configuration files

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/backups/config"
ARCHIVE="config_backup_${DATE}.tar.gz"

# Create backup directory if it does not exist
mkdir -p "$BACKUP_DIR"

# Create compressed archive of configuration
tar -czf "${BACKUP_DIR}/${ARCHIVE}" \
    /etc/ssh/ \
    /etc/hosts \
    /etc/fstab \
    /etc/crontab \
    --exclude='*.bak' \
    --exclude='*.old'

# Verify archive was created
if [ -f "${BACKUP_DIR}/${ARCHIVE}" ]; then
    echo "Backup created: ${ARCHIVE}"
    ls -lh "${BACKUP_DIR}/${ARCHIVE}"
else
    echo "ERROR: Backup failed"
    exit 1
fi

# Remove backups older than 30 days
find "$BACKUP_DIR" -name "config_backup_*.tar.gz" -mtime +30 -delete
```

---

## Real-World Examples

### Scenario 1: System Information Gathering Script

A system administrator needs to quickly gather information about a RHEL 10 server for documentation.

```bash
#!/bin/bash
# gather_info.sh - Collect system information

OUTPUT="/tmp/system_info_$(date +%Y%m%d).txt"

echo "=== System Information Report ===" > "$OUTPUT"
echo "Generated: $(date)" >> "$OUTPUT"
echo "" >> "$OUTPUT"

# System details
echo "--- System Details ---" >> "$OUTPUT"
hostname >> "$OUTPUT"
cat /etc/os-release | grep PRETTY_NAME >> "$OUTPUT"
uname -r >> "$OUTPUT"
echo "" >> "$OUTPUT"

# Disk usage
echo "--- Disk Usage ---" >> "$OUTPUT"
df -h | grep -v tmpfs >> "$OUTPUT"
echo "" >> "$OUTPUT"

# Memory usage
echo "--- Memory Usage ---" >> "$OUTPUT"
free -h >> "$OUTPUT"
echo "" >> "$OUTPUT"

# Running services
echo "--- Active Services ---" >> "$OUTPUT"
systemctl list-units --type=service --state=running --no-pager | head -20 >> "$OUTPUT"
echo "" >> "$OUTPUT"

# Listening ports
echo "--- Listening Ports ---" >> "$OUTPUT"
ss -tlnp >> "$OUTPUT"
echo "" >> "$OUTPUT"

# Failed logins
echo "--- Recent Failed Logins ---" >> "$OUTPUT"
grep "Failed password" /var/log/secure 2>/dev/null | tail -10 >> "$OUTPUT"

echo "Report saved to: $OUTPUT"
```

### Scenario 2: Log Analysis for Security Audit

An administrator needs to identify potential security issues from system logs.

```bash
# Find all failed SSH login attempts
grep "Failed password" /var/log/secure | \
    grep -oP "from \K[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" | \
    sort | uniq -c | sort -rn | head -20

# Find successful root logins
grep "session opened.*root" /var/log/secure

# Find all user account changes
grep "useradd\|usermod\|userdel" /var/log/secure

# Find permission denied errors
grep "Permission denied" /var/log/secure

# Find SELinux denials
grep "avc:  denied" /var/log/audit/audit.log | \
    grep -oP "type=\K[^ ]+" | sort | uniq -c | sort -rn
```

### Scenario 3: Remote Server Health Check

An administrator needs to check the health of multiple remote servers.

```bash
#!/bin/bash
# check_servers.sh - Health check for remote servers

SERVERS="web01 web02 db01"

for SERVER in $SERVERS; do
    echo "=== Checking $SERVER ==="

    # Check if server is reachable
    if ssh -o ConnectTimeout=5 -o BatchMode=yes "$SERVER" "exit" 2>/dev/null; then
        echo "Status: ONLINE"

        # Check disk usage
        ssh "$SERVER" "df -h / | tail -1" | awk '{print "Disk: " $5 " used"}'

        # Check memory
        ssh "$SERVER" "free -m" | grep Mem | awk '{printf "Memory: %.0f%% used\n", $3/$2*100}'

        # Check load average
        ssh "$SERVER" "uptime" | awk -F'average: ' '{print "Load: " $2}'
    else
        echo "Status: OFFLINE"
    fi
    echo ""
done
```

### Scenario 4: File Permission Audit

An administrator needs to find files with overly permissive settings.

```bash
# Find world-writable files in /var/www
find /var/www -type f -perm -o+w -ls

# Find SUID files (potential security risk)
find / -type f -perm -4000 -ls 2>/dev/null

# Find SGID files
find / -type f -perm -2000 -ls 2>/dev/null

# Find files with no owner
find / -type f -nouser -ls 2>/dev/null

# Find world-writable directories
find / -type d -perm -o+w ! -perm -1000 -ls 2>/dev/null

# Find files owned by root in user home directories
find /home -user root -ls 2>/dev/null
```

---

## Troubleshooting

### SSH Connection Issues

**Problem**: SSH connection refused

```bash
# Check if SSH service is running on remote host
ssh -v username@remote_host

# Check SSH port
ss -tlnp | grep :22

# Verify SSH service status on remote host
ssh username@remote_host "systemctl status sshd"

# Check firewall rules
firewall-cmd --list-all

# Test specific port connectivity
nc -zv remote_host 22
```

**Problem**: SSH key authentication fails

```bash
# Check key permissions (must be restrictive)
ls -la ~/.ssh/
chmod 700 ~/.ssh
chmod 600 ~/.ssh/private_key
chmod 644 ~/.ssh/private_key.pub

# Verify authorized_keys on server
ssh username@remote_host "ls -la ~/.ssh/authorized_keys"
ssh username@remote_host "stat ~/.ssh"

# Check SSH server logs
ssh username@remote_host "journalctl -u sshd --no-pager -n 20"

# Debug key authentication
ssh -vvv username@remote_host
```

**Problem**: SSH connection timeout

```bash
# Check DNS resolution
nslookup remote_host
dig remote_host

# Try connecting by IP address
ssh username@192.168.1.100

# Check known_hosts for key mismatch
ssh-keygen -R remote_host

# Test with reduced security for debugging
ssh -o StrictHostKeyChecking=no -o PasswordAuthentication=yes username@remote_host
```

### Permission Denied Errors

**Problem**: Cannot access a file despite having permissions

```bash
# Check the entire path - every directory needs execute permission
namei -l /path/to/file

# Check for SELinux context issues
ls -Z /path/to/file
getenforce

# Check if file has immutable attribute
lsattr /path/to/file

# Check ACLs
getfacl /path/to/file
```

**Problem**: Cannot execute a script

```bash
# Check if execute permission is set
ls -la script.sh

# Check file type
file script.sh

# Check if shell exists
head -1 script.sh
cat /etc/shells

# Add execute permission
chmod +x script.sh
```

### Redirection and Pipe Issues

**Problem**: Command output not captured as expected

```bash
# Check what goes to stdout vs stderr
command 1>/tmp/stdout.txt 2>/tmp/stderr.txt

# Verify both streams are captured
command &> /tmp/combined.txt

# Test pipe chain step by step
cat /etc/passwd | head -5
cat /etc/passwd | grep bash
cat /etc/passwd | grep bash | cut -d: -f1
```

**Problem**: grep returns no results

```bash
# Verify file exists and has content
ls -la filename.txt
wc -l filename.txt

# Test with simpler pattern
grep "exacttext" filename.txt

# Check for hidden characters
cat -A filename.txt

# Verify regex syntax
grep -E "pattern" filename.txt
```

### tar Archive Issues

**Problem**: Cannot extract archive

```bash
# Check archive integrity
tar -tzf archive.tar.gz

# Check file type
file archive.tar.gz

# Try different compression flag
tar -xzf archive.tar.gz    # gzip
tar -xjf archive.tar.bz2  # bzip2
tar -xf archive.tar       # uncompressed

# Check disk space
df -h /destination/path
```

### Documentation Lookup Issues

**Problem**: man page not found

```bash
# Check if package providing man pages is installed
dnf search man-pages
dnf install man-pages

# Search across all sections
man -a command_name

# Use apropos for fuzzy search
apropos keyword

# Check info pages
info command_name

# Search online documentation
lynx https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/10/
```

---

## Verification Procedures

### Shell Access Verification

```bash
# Verify shell is accessible and functional
echo "Shell test: $(date)"
whoami
pwd
echo $SHELL
bash --version | head -1
```

### Redirection Verification

```bash
# Test stdout redirection
echo "test output" > /tmp/test_redirect.txt
cat /tmp/test_redirect.txt
rm /tmp/test_redirect.txt

# Test append redirection
echo "line 1" > /tmp/test_append.txt
echo "line 2" >> /tmp/test_append.txt
cat /tmp/test_append.txt
rm /tmp/test_append.txt

# Test stderr redirection
ls /nonexistent 2> /tmp/test_stderr.txt
cat /tmp/test_stderr.txt
rm /tmp/test_stderr.txt

# Test pipe
echo -e "apple\nbanana\ncherry" | grep "ban"
```

### grep and Regex Verification

```bash
# Test basic grep
echo -e "hello\nworld\nhello again" | grep "hello"

# Test case-insensitive
echo -e "Hello\nWORLD" | grep -i "hello"

# Test regex anchoring
echo -e "root:x:0:0\nnobody:x:65534" | grep "^root"

# Test extended regex
echo -e "192.168.1.1\n10.0.0.1\nabc" | grep -E "^[0-9]+\.[0-9]+"
```

### SSH Verification

```bash
# Test SSH connectivity
ssh -o ConnectTimeout=5 username@remote_host "echo connected"

# Verify key-based authentication
ssh -o PasswordAuthentication=no username@remote_host "echo key_auth_works"

# Verify SSH configuration syntax
sshd -t

# Check SSH service status
systemctl is-active sshd
```

### File Operations Verification

```bash
# Test file creation
touch /tmp/testfile.txt
ls -la /tmp/testfile.txt

# Test directory creation
mkdir -p /tmp/test/nested/dir
ls -la /tmp/test/nested/

# Test copy with attribute preservation
cp -p /etc/hosts /tmp/hosts_backup
diff /etc/hosts /tmp/hosts_backup

# Test move
touch /tmp/movetest.txt
mv /tmp/movetest.txt /tmp/moved.txt
ls -la /tmp/moved.txt
```

### Link Verification

```bash
# Test hard link
touch /tmp/hardlink_source.txt
echo "content" > /tmp/hardlink_source.txt
ln /tmp/hardlink_source.txt /tmp/hardlink_link.txt
ls -li /tmp/hardlink_source.txt /tmp/hardlink_link.txt
diff /tmp/hardlink_source.txt /tmp/hardlink_link.txt

# Test symbolic link
ln -s /tmp/hardlink_source.txt /tmp/softlink.txt
ls -la /tmp/softlink.txt
readlink /tmp/softlink.txt
```

### Permission Verification

```bash
# Test permission setting
touch /tmp/permtest.txt
chmod 640 /tmp/permtest.txt
stat -c "%a %A %U:%G" /tmp/permtest.txt

# Test ownership change
chown root:root /tmp/permtest.txt
stat -c "%U:%G" /tmp/permtest.txt

# Verify numeric to symbolic conversion
# 755 = rwxr-xr-x
# 644 = rw-r--r--
# 600 = rw-------
```

### Archive Verification

```bash
# Test tar creation and extraction
mkdir -p /tmp/tartest
echo "file1" > /tmp/tartest/file1.txt
echo "file2" > /tmp/tartest/file2.txt
tar -czf /tmp/testarchive.tar.gz -C /tmp tartest
tar -tzf /tmp/testarchive.tar.gz

# Extract and verify
mkdir -p /tmp/tartest_extract
tar -xzf /tmp/testarchive.tar.gz -C /tmp/tartest_extract
diff /tmp/tartest/file1.txt /tmp/tartest_extract/tartest/file1.txt
```

### Documentation Verification

```bash
# Test man page access
man ls | head -5
man 1 ls > /dev/null 2>&1 && echo "man works"

# Test apropos
apropos "copy file" | head -3

# Test info
info ls > /dev/null 2>&1 && echo "info works"

# Test documentation directory
ls /usr/share/doc/ | head -5
```

---

## RHCSA Exam Notes

### Exam Relevance

This chapter covers one of the most heavily weighted exam objectives. Expect 3-5 questions testing these skills directly, and these skills are prerequisites for nearly every other exam task.

### Key Exam Points

1. **Shell Proficiency**: You will work entirely in the command line during the exam. Know your shell shortcuts, tab completion, and command history inside and out. Speed matters.

2. **Redirection**: Exam questions may ask you to redirect command output to specific files. Know the difference between `>` and `>>` on an exam question. A common trap is using `>` when `>>` is needed, which overwrites existing data.

3. **grep Mastery**: You will need to use `grep` to find information in configuration files and logs. Know both basic and extended regex. The exam may present a log file and ask you to extract specific information.

4. **SSH**: You will SSH between exam systems (server and desktop). Know how to use key-based authentication. The exam environment provides pre-configured keys; you just need to know how to connect.

5. **tar Archives**: Expect to create and extract archives during the exam. Know the difference between `-z` (gzip) and `-j` (bzip2) flags. The exam may specify which compression to use.

6. **Permissions**: This is a high-frequency exam topic. Know both symbolic and numeric chmod. Understand that directory permissions work differently than file permissions (execute bit on directories means "can enter").

7. **Links**: Know when to use hard vs. soft links. Exam questions may specify which type to create. Remember: hard links cannot cross filesystems and cannot link to directories.

8. **Documentation**: You have access to man pages during the exam. Use `man` and `apropos` when you forget command syntax. This is an allowed resource.

### Common Exam Traps

- Using `>` instead of `>>` when asked to append to a file
- Forgetting the `-p` flag with `mkdir` for nested directories
- Confusing `cp -r` (recursive) with `cp -p` (preserve attributes)
- Setting incorrect permissions (e.g., 777 when 755 is appropriate)
- Creating a hard link when a soft link is needed (or vice versa)
- Forgetting that `chmod` numeric values are octal, not decimal
- Not using `su -` (with dash) which loads the target user's environment

### Time Management Tips

- Use tab completion to save time and avoid typos
- Use command history (`!`, `Ctrl+R`) to repeat and modify previous commands
- Keep a scratchpad: redirect important outputs to `/tmp/` for reference
- Verify your work immediately after completing each task
- Use `man` quickly when unsure about command options

### Exam Environment Notes

- You will have access to two systems: a desktop and a server
- Use SSH to switch between systems as needed
- `man` pages are available and encouraged
- No internet access; rely on local documentation
- Tasks must persist across reboots unless stated otherwise

---

## Chapter Summary

This chapter covered the foundational tools every RHEL 10 administrator must master. You learned how to interact with the shell, redirect input and output between commands and files, search and filter text with `grep` and regular expressions, connect to remote systems securely via SSH, manage archives with `tar`, perform file and directory operations, create both hard and soft links, control access with permissions, and find answers in system documentation.

These skills form the bedrock of Linux administration. Every advanced task builds upon these fundamentals. During the RHCSA exam, you will use these tools repeatedly, often combining them in creative ways to solve problems efficiently.

---

## Quick Reference

### Shell and Commands

```bash
echo $SHELL                    # Current shell
type command                   # Command type
which command                  # Command location
history                        # Command history
!number                        # Run history entry
```

### Redirection

```bash
cmd > file                    # Overwrite with stdout
cmd >> file                   # Append stdout
cmd 2> file                   # Redirect stderr
cmd &> file                   # Redirect both
cmd < file                    # Redirect stdin
cmd1 | cmd2                   # Pipe
```

### grep Patterns

```bash
grep "pattern" file           # Basic search
grep -i "pattern" file       # Case-insensitive
grep -n "pattern" file       # Show line numbers
grep -c "pattern" file       # Count matches
grep -v "pattern" file       # Invert match
grep -r "pattern" dir/       # Recursive
grep -E "regex" file        # Extended regex
```

### SSH

```bash
ssh user@host                 # Connect
ssh -p port user@host        # Custom port
ssh-keygen -t ed25519        # Generate key
ssh-copy-id user@host        # Deploy key
ssh -v user@host             # Verbose/debug
```

### tar Archives

```bash
tar -czf archive.tar.gz dir/ # Create gzip archive
tar -cjf archive.tar.bz2 dir/# Create bzip2 archive
tar -xzf archive.tar.gz      # Extract gzip
tar -xjf archive.tar.bz2     # Extract bzip2
tar -tf archive.tar          # List contents
tar -xf archive.tar -C dir/  # Extract to directory
```

### File Operations

```bash
touch file                   # Create empty file
mkdir -p dir/nested          # Create directories
cp -a source dest           # Copy preserving all
mv source dest              # Move/rename
rm -rf directory            # Remove recursively
ls -la                      # Detailed listing
```

### Links

```bash
ln source link              # Hard link
ln -s target link           # Symbolic link
ls -li file                 # Show inode
readlink link               # Show symlink target
```

### Permissions

```bash
chmod 755 file              # Numeric mode
chmod u+x file              # Symbolic mode
chown user:group file       # Change ownership
chgrp group file            # Change group
ls -la file                 # View permissions
stat file                   # Detailed info
```

### Documentation

```bash
man command                 # Man page
man -k keyword             # Search man pages
apropos keyword            # Search man pages
info command               # Info page
ls /usr/share/doc/         # Package docs
```

### Permission Reference

| Octal | Symbolic | Meaning |
|-------|----------|---------|
| 7 | rwx | Read, write, execute |
| 6 | rw- | Read, write |
| 5 | r-x | Read, execute |
| 4 | r-- | Read only |
| 3 | -wx | Write, execute |
| 2 | -w- | Write only |
| 1 | --x | Execute only |
| 0 | --- | No permissions |

---

## Review Questions

1. What is the difference between `>` and `>>` redirection operators?

2. How do you redirect both stdout and stderr to the same file?

3. What does the command `grep -E "^(root|admin)" /etc/passwd` do?

4. What is the difference between a hard link and a symbolic link?

5. What numeric permission value corresponds to `rwxr-xr--`?

6. How do you create a gzip-compressed tar archive of the `/etc` directory?

7. What command would you use to find all man pages related to "copy"?

8. What is the effect of the execute permission on a directory versus a file?

9. How do you switch to another user's shell environment completely (including their environment variables)?

10. What command displays the inode number of a file?

11. How do you extract a bzip2-compressed tar archive to a specific directory?

12. What is the purpose of the `-p` flag in `mkdir -p /a/b/c`?

13. How would you search for the word "error" case-insensitively in all `.log` files under `/var/log`?

14. What does `chmod 600 file.txt` do?

15. How do you verify where a symbolic link points?

---

## Answers

1. `>` redirects stdout to a file, overwriting any existing content. `>>` redirects stdout to a file, appending to existing content without overwriting.

2. Use `command &> file` (Bash shorthand) or `command > file 2>&1` (POSIX-compatible). Both send stdout and stderr to the same file.

3. This command uses extended regex (`-E`) to search `/etc/passwd` for lines that start with (`^`) either `root` or (`|`) `admin`. It finds user accounts whose usernames begin with "root" or "admin."

4. A hard link is an additional directory entry pointing to the same inode as the original file. Both names are equally valid references to the same data. Hard links cannot cross filesystem boundaries or link to directories. A symbolic link is a separate file containing a path reference to the target. If the target is deleted, the symbolic link becomes broken. Symbolic links can cross filesystems and link to directories.

5. The numeric value is `754`. Breaking it down: `7` (rwx) for owner, `5` (r-x) for group, `4` (r--) for others.

6. Use `tar -czf archive.tar.gz /etc`. The `-c` creates an archive, `-z` compresses with gzip, and `-f` specifies the output filename.

7. Use `man -k copy` or `apropos copy`. Both search man page names and descriptions for the keyword "copy."

8. On a file, execute permission allows the file to be run as a program or script. On a directory, execute permission allows the user to enter (traverse) the directory and access files within it. Without execute permission on a directory, you cannot access its contents even if you have read permission.

9. Use `su - username` or `sudo -i -u username`. The dash (`-`) simulates a full login, loading the target user's environment variables, shell profile, and working directory.

10. Use `ls -li filename` to display the inode number in the first column, or `stat filename` for detailed file information including the inode.

11. Use `tar -xjf archive.tar.bz2 -C /target/directory`. The `-x` extracts, `-j` handles bzip2 compression, and `-C` changes to the target directory before extracting.

12. The `-p` flag creates parent directories as needed. Without it, `mkdir /a/b/c` would fail if `/a` or `/a/b` do not exist. With `-p`, all missing parent directories are created automatically.

13. Use `grep -ri "error" /var/log/*.log`. The `-r` flag searches recursively, `-i` makes the search case-insensitive, and the glob pattern `*.log` limits the search to log files.

14. `chmod 600 file.txt` sets permissions to `rw-------`, meaning only the file owner can read and write the file. Group members and all other users have no permissions.

15. Use `readlink symlink_name` to display the target path, or `ls -l symlink_name` which shows the link target after the `->` arrow.
