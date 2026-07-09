# Chapter 9: Managing Users and Groups

## Learning Objectives

By the end of this chapter, you will be able to:

- Create, delete, and modify local user accounts
- Change passwords and adjust password aging for local user accounts
- Create, delete, and modify local groups and group memberships
- Configure privileged access using sudo
- Understand user account security and best practices
- Manage user shells and home directories
- Configure user authentication methods

---

## Concepts and Theory

### User Account Structure

Each user account in Linux has associated data stored in system files:

**/etc/passwd:**
- User account information
- Format: `username:x:UID:GID:GECOS:home_directory:shell`
- Contains login information for all users
- Readable by all users (contains only `x` for password field)

**/etc/shadow:**
- Password and account security information
- Format: `username:password_hash:last_change:min:max:warn:inactive:expire:flag`
- Contains encrypted passwords and aging information
- Accessible only by root

**/etc/group:**
- Group information
- Format: `group_name:x:GID:user_list`
- Defines groups and their members

**/etc/gshadow:**
- Group security information
- Format: `group_name:password_admins:last_change:min:max:warn:inactive:flag:user_list`
- Contains encrypted group passwords and member lists
- Accessible only by root

### User Identifiers

**UID (User ID):**
- Unique identifier for each user
- **0**: Root (superuser)
- **1-999**: System accounts
- **1000+**: Regular user accounts
- **1000**: First regular user account

**GID (Group ID):**
- Unique identifier for each group
- **0**: Root group
- **1-999**: System groups
- **1000+**: Regular user groups

### User Account Creation

**useradd Command:**
- Creates user account
- Sets up home directory
- Creates mail spool directory
- Sets default shell
- Creates group if specified

**Options:**
- `-m`: Create home directory
- `-d`: Specify home directory
- `-s`: Specify shell
- `-g`: Specify primary group
- `-G`: Specify supplementary groups
- `-c`: Add comment/GECOS field
- `-e`: Set account expiration date
- `-G`: Add to supplementary groups
- `-u`: Specify UID

### Group Management

**Groups:**
- **Primary group**: User's default group (first group in /etc/passwd)
- **Supplementary groups**: Additional groups user belongs to

**groupadd Command:**
- Creates new group
- Options: `-g` for GID, `-f` to allow duplicate GIDs

**groupmod Command:**
- Modifies existing group
- Options: `-g` for new GID, `-n` for new name

**groupdel Command:**
- Deletes group

### Password Management

**passwd Command:**
- Changes passwords
- Enforces password policies
- Supports password aging

**Password Aging:**
- **Last change**: When password was last changed
- **Minimum days**: Minimum days before change
- **Maximum days**: Maximum days before change required
- **Warning days**: Days before expiry to warn
- **Inactive days**: Days after expiry before account locks
- **Expire date**: Account lock date

**chage Command:**
- Modifies password aging
- Options: `-l` list, `-M` max days, `-m` min days, `-W` warn days, `-I` inactive days, `-E` expire date

### Privileged Access

**sudo:**
- Allows users to run commands as another user (usually root)
- Configured in /etc/sudoers
- Logs all commands
- Can limit commands by user

**su (Switch User):**
- Switches to another user account
- `su -` loads target user's environment
- Requires password (unless in sudoers)

### User Security

**Account Locking:**
- Password expired
- Account disabled
- Lock file created

**Password Policies:**
- Minimum length
- Complexity requirements
- History requirements
- Expiration requirements

**File Permissions:**
- Home directory: 700 (drwx------)
- Personal files: 600 (rw-------)
- Executables: 755 (rwxr-xr-x)

---

## How It Works Internally

### User Account Creation Process

When a user is created:

1. **UID/GID assigned** from system databases
2. **Home directory created** at /home/username
3. **Mail spool created** at /var/spool/mail/username
4. **Shell created** with default permissions
5. **Login scripts executed** (if configured)
6. **User entry added** to /etc/passwd
7. **Password placeholder added** to /etc/shadow
8. **Group entries added** to /etc/group

### Password Hashing

Linux uses various password hashing algorithms:

- **SHA-512**: Default, most secure
- **SHA-256**: Alternative
- **MD5**: Deprecated
- **crypt**: Legacy

Password hashing process:
1. User enters password
2. Password is hashed with salt
3. Hash stored in /etc/shadow
4. Verification uses same algorithm

### sudo Authorization

sudo authorization works by:

1. **Reading /etc/sudoers** configuration
2. **Checking user credentials**
3. **Validating permissions** for command
4. **Running as target user**
5. **Logging command** to /var/log/secure

### Group Membership

Group membership works by:

1. **Primary group** set at account creation
2. **Supplementary groups** added via /etc/group
3. **User lookup** by UID in /etc/passwd
4. **Group lookup** by GID in /etc/group
5. **Permission checks** against group ownership

---

## Commands and Administration Tasks

### Creating User Accounts

#### Basic User Creation

```bash
# Create user with default settings
sudo useradd username

# Create user with home directory
sudo useradd -m username

# Create user with specific home directory
sudo useradd -m -d /home/custom/user username

# Create user with specific shell
sudo useradd -m -s /bin/bash username

# Create user with specific UID
sudo useradd -m -u 1001 username

# Create user with comment
sudo useradd -m -c "John Doe" username

# Create user with primary group
sudo useradd -m -g users username

# Create user with supplementary groups
sudo useradd -m -G wheel,adm username

# Create user with expiration date
sudo useradd -m -e 2024-12-31 username

# Create user with specific GECOS field
sudo useradd -m -c "John Doe,john@example.com,123-456-7890" username
```

#### Advanced User Creation

```bash
# Create user with specific attributes
sudo useradd \
    -m \          # Create home directory
    -d /home/john \ # Home directory
    -s /bin/bash \ # Shell
    -g users \     # Primary group
    -G wheel,adm \ # Supplementary groups
    -c "John Doe" \ # Comment
    -e 2024-12-31 \ # Expiration date
    -G staff \     # More supplementary groups
    -u 1001 \      # Specific UID
    -G nogroup \   # Default group (no group)
    john

# Create user without home directory
sudo useradd -m -d /var/www -s /usr/sbin/nologin www-data

# Create user with login disabled
sudo useradd -m -s /usr/sbin/nologin -d /home/backup backup

# Create user with force bad password
sudo useradd -m -d /home/test -s /bin/bash -G wheel test
sudo passwd -l test
```

#### Creating System Users

```bash
# Create system user (UID < 1000)
sudo useradd -m -u 501 -s /bin/bash -d /home/systemsystem systemuser

# Create user with no login access
sudo useradd -m -u 601 -s /usr/sbin/nologin -d /var/www wwwrun

# Create user for service account
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/myservice myservice

# Create user with specific group
sudo useradd -r -g nogroup -s /usr/sbin/nologin -d /var/log syslog
```

### Modifying User Accounts

#### Basic Modifications

```bash
# Change user shell
sudo usermod -s /bin/zsh username

# Change home directory
sudo usermod -d /home/newdir username

# Change primary group
sudo usermod -g newgroup username

# Add user to supplementary groups
sudo usermod -aG wheel,adm username

# Remove user from supplementary groups
sudo usermod -G wheel username

# Change home directory ownership
sudo chown newuser:newgroup /home/username

# Change user comment
sudo usermod -c "New Name, new@example.com" username

# Change user UID
sudo usermod -u 1001 username

# Change user GID
sudo usermod -g 1001 username

# Combine multiple changes
sudo usermod \
    -s /bin/zsh \
    -d /home/newdir \
    -g newgroup \
    -aG wheel,adm \
    username
```

#### Advanced Modifications

```bash
# Force password change at next login
sudo usermod -e 2024-12-31 username

# Lock account
sudo usermod -L username

# Unlock account
sudo usermod -U username

# Change account expiration
sudo usermod -e 2024-12-31 username

# Disable account
sudo usermod -L username

# Create home directory if it doesn't exist
sudo usermod -m username

# Change login shell to nologin
sudo usermod -s /usr/sbin/nologin username

# View user information
id username
getent passwd username
grep username /etc/passwd
```

### Deleting User Accounts

#### Basic Deletion

```bash
# Delete user and home directory
sudo userdel -r username

# Delete user without home directory
sudo userdel username

# Delete user without prompting
sudo userdel -f username

# Delete user and mail spool
sudo userdel -r username

# Delete user and groups
sudo userdel -r username
sudo groupdel username
```

#### Advanced Deletion

```bash
# Delete user with force option
sudo userdel -f username

# Delete user with ignore nonexistent home
sudo userdel -r username

# Delete user and all associated files
sudo rm -rf /home/username
sudo rm -rf /var/spool/mail/username

# Remove user from all groups
sudo usermod -G "" username
sudo userdel username

# Delete user and backup files
sudo rm -rf /home/username/*~
sudo rm -rf /home/username/.bash_history
sudo userdel -r username
```

### Managing Passwords

#### Password Changes

```bash
# Change own password
passwd

# Change user password
sudo passwd username

# Force password change
sudo passwd -e username

# Lock password
sudo passwd -l username

# Unlock password
sudo passwd -u username

# Set password expiration
sudo passwd -e 2024-12-31 username

# Set password never expires
sudo passwd -i username

# View password status
passwd username

# Change root password
sudo passwd

# Force password change at next login
sudo passwd -f username

# Set password to empty (insecure)
sudo passwd -d username
```

#### Password Aging

```bash
# View password aging
chage -l username

# Set maximum password age (days)
sudo chage -M 90 username

# Set minimum password age (days)
sudo chage -m 7 username

# Set password warning days
sudo chage -W 7 username

# Set password inactive days after expiration
sudo chage -I 30 username

# Set password expiration date
sudo chage -E 2024-12-31 username

# Set account to never expire
sudo chage -E -1 username

# Set account to expire immediately
sudo chage -E 2024-01-01 username

# Reset password aging
sudo chage -l username
```

### Managing Groups

#### Creating Groups

```bash
# Create group with default GID
sudo groupadd developers

# Create group with specific GID
sudo groupadd -g 1001 developers

# Create group with duplicate GID allowed
sudo groupadd -f -g 1001 developers

# Create group with specific GECOS field
sudo groupadd -c "Development Team" developers

# Create group with specific shell
sudo groupadd -s /bin/bash developers

# Create group with specific home directory
sudo groupadd -d /home/developers developers

# Create group with specific UID
sudo groupadd -u 1001 developers

# Create group with specific GID range
sudo groupadd -g 1000-1099 developers

# Create group with specific GECOS field
sudo groupadd -c "Development Team" developers
```

#### Modifying Groups

```bash
# Change group name
sudo groupmod -n newname oldname

# Change group GID
sudo groupmod -g 1001 developers

# Change group GECOS field
sudo groupmod -c "New Development Team" developers

# Combine multiple changes
sudo groupmod \
    -n newname \
    -g 1001 \
    -c "New Development Team" \
    developers
```

#### Deleting Groups

```bash
# Delete group
sudo groupdel developers

# Delete group with force option
sudo groupdel -f developers

# Delete group and all members
sudo groupdel developers

# Delete group and all files
sudo rm -rf /home/developers
sudo groupdel developers
```

### Managing Group Memberships

#### Adding Members

```bash
# Add user to group
sudo usermod -aG developers username

# Add user to multiple groups
sudo usermod -aG developers,wheel username

# Add user to primary group
sudo usermod -g developers username

# Add user to supplementary groups
sudo usermod -aG wheel,adm username

# Add user to group without changing primary group
sudo usermod -aG developers username
```

#### Removing Members

```bash
# Remove user from group
sudo usermod -G "" username

# Remove user from specific group
sudo usermod -G wheel username

# Remove user from supplementary groups
sudo usermod -G wheel,adm username

# Remove user from all groups except primary
sudo usermod -G "" username

# Remove user from all groups
sudo usermod -G "" username
```

### Managing User Shells

#### Setting Shells

```bash
# Set default shell
sudo usermod -s /bin/bash username

# Set login shell to nologin
sudo usermod -s /usr/sbin/nologin username

# Set login shell to false
sudo usermod -s /usr/sbin/false username

# Set login shell to sh
sudo usermod -s /bin/sh username

# Set login shell to csh
sudo usermod -s /bin/csh username

# Set login shell to ksh
sudo usermod -s /bin/ksh username

# Set login shell to zsh
sudo usermod -s /bin/zsh username

# Set login shell to fish
sudo usermod -s /usr/bin/fish username

# View available shells
cat /etc/shells
```

#### Shell Configuration

```bash
# Set bash as default shell
sudo usermod -s /bin/bash username

# Set login shell
sudo usermod -s /bin/bash username

# Set interactive shell
sudo usermod -s /bin/bash username

# Set non-interactive shell
sudo usermod -s /usr/sbin/nologin username

# Set shell for service account
sudo useradd -s /usr/sbin/nologin -m myservice

# View shell configuration
getent passwd username
grep username /etc/passwd
```

### Privileged Access Configuration

#### Sudo Configuration

```bash
# Edit sudoers file
sudo visudo

# Add user to sudo group
sudo usermod -aG sudo username

# Allow specific user to run all commands
username ALL=(ALL) ALL

# Allow specific user to run specific commands
username ALL=(ALL) /usr/bin/systemctl

# Allow specific user to run commands in specific directories
username ALL=(ALL) /usr/local/bin/admin_commands

# Allow specific user to run commands as specific users
username ALL=(root) NOPASSWD: /usr/bin/apt

# Allow specific user to run commands without password
username ALL=(ALL) NOPASSWD: ALL

# Allow specific user to run commands with password
username ALL=(ALL) ALL

# Log sudo commands
Defaults logfile="/var/log/sudo.log"

# Set sudo timeout
Defaults timestamp_timeout=30

# Set sudo logging
Defaults log_input, log_output, log_all

# Set sudoers file
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

#### Sudoers Examples

```bash
# Allow specific user to run all commands
username ALL=(ALL) ALL

# Allow specific user to run specific commands
username ALL=(ALL) /usr/bin/systemctl restart nginx

# Allow specific user to run commands without password
username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx

# Allow specific user to run commands as specific users
username ALL=(root) NOPASSWD: /usr/bin/apt-get update

# Allow specific user to run commands in specific directories
username ALL=(ALL) /usr/local/bin/admin_commands

# Allow specific user to run commands with password
username ALL=(ALL) ALL

# Allow specific user to run commands without password
username ALL=(ALL) NOPASSWD: ALL

# Allow specific user to run commands with password
username ALL=(ALL) ALL

# Allow specific user to run commands without password
username ALL=(ALL) NOPASSWD: ALL

# Allow specific user to run commands with password
username ALL=(ALL) ALL
```

---

## Configuration Examples

### /etc/sudoers Configuration

File: `/etc/sudoers.d/admins`

```bash
# Sudoers configuration for administrators

# Allow admin user to run all commands
admin ALL=(ALL) ALL

# Allow admin user to run specific commands without password
admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart httpd

# Allow admin user to run commands as specific users
admin ALL=(root) NOPASSWD: /usr/bin/apt-get update
admin ALL=(root) NOPASSWD: /usr/bin/dnf update

# Allow admin user to run commands in specific directories
admin ALL=(ALL) /usr/local/bin/admin_commands

# Allow admin user to run commands with password
admin ALL=(ALL) ALL

# Log sudo commands
Defaults logfile="/var/log/sudo.log"

# Set sudo timeout
Defaults timestamp_timeout=30

# Set sudo logging
Defaults log_input, log_output, log_all

# Set sudoers file
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

### /etc/pam.d/common-password Configuration

File: `/etc/pam.d/common-password`

```bash
# PAM configuration for password management

# Password quality check
password requisite pam_pwquality.so try_first_pass local_users_only \
    retry=3 minlen=8 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1

# Password aging
password requisite pam_pwhistory.so use_authtok remember=5

# Password expiration
password required pam_expiration.so use_authtok

# Password history
password required pam_unix.so try_first_pass obscure remember=5

# Password change
password optional pam_permit.so
```

### User Account Configuration

File: `/etc/login.defs`

```bash
# Login definitions for user accounts

# User account settings
PASS_MAX_DAYS   90
PASS_MIN_DAYS   7
PASS_MIN_LEN    8
PASS_WARN_AGE   7

# UID/GID ranges
UID_MIN         1000
UID_MAX         60000
UID_INVALID     60001
GID_MIN         1000
GID_MAX         60000
GID_INVALID     60001

# Home directory settings
HOME_DIR        /home
USERGROUPS_ENAB yes
```

### Group Configuration

File: `/etc/group.d/developers`

```bash
# Group configuration for developers

# Developers group
developers:x:1001:john,jane,bob

# Admin group
admin:x:1002:john,jane

# Monitoring group
monitoring:x:1003:jane,bob
```

---

## Real-World Examples

### Scenario 1: Setting Up a Development Team

A company needs to set up user accounts for a development team.

```bash
# Create primary group for developers
sudo groupadd -g 1001 developers

# Create individual user accounts
sudo useradd -m -g developers -s /bin/bash -c "John Doe" john
sudo useradd -m -g developers -s /bin/bash -c "Jane Smith" jane
sudo useradd -m -g developers -s /bin/bash -c "Bob Johnson" bob

# Set initial passwords
echo "john:password123" | sudo chpasswd
echo "jane:password123" | sudo chpasswd
echo "bob:password123" | sudo chpasswd

# Add users to sudo group for administrative access
sudo usermod -aG sudo john
sudo usermod -aG sudo jane

# Set password aging policies
sudo chage -M 90 -m 7 -W 7 -I 30 -E -1 john
sudo chage -M 90 -m 7 -W 7 -I 30 -E -1 jane
sudo chage -M 90 -m 7 -W 7 -I 30 -E -1 bob

# Configure sudo for developers
sudo visudo
# Add to /etc/sudoers.d/developers:
developers ALL=(ALL) ALL

# Verify user accounts
id john
id jane
id bob

# Verify group membership
getent group developers
```

### Scenario 2: Setting Up Service Accounts

A server needs service accounts for various applications.

```bash
# Create service account for web server
sudo useradd -r -s /usr/sbin/nologin -d /var/www -m www-data

# Create service account for database
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/mysql -m mysql

# Create service account for backup
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/backup -m backup

# Create service account for monitoring
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/monitoring -m monitoring

# Create group for service accounts
sudo groupadd -g 1001 services

# Add service accounts to services group
sudo usermod -aG services www-data
sudo usermod -aG services mysql
sudo usermod -aG services backup
sudo usermod -aG services monitoring

# Set service account permissions
sudo chown -R www-data:www-data /var/www
sudo chown -R mysql:mysql /var/lib/mysql
sudo chown -R backup:backup /var/lib/backup
sudo chown -R monitoring:monitoring /var/lib/monitoring

# Verify service accounts
id www-data
id mysql
id backup
id monitoring
```

### Scenario 3: Implementing Password Policies

A company needs to enforce strong password policies.

```bash
# Set password minimum length
sudo passwd -l

# Set password minimum length to 12 characters
sudo sed -i 's/PASS_MIN_LEN.*/PASS_MIN_LEN    12/' /etc/login.defs

# Set password complexity requirements
sudo sed -i 's/pam_pwquality.so.*/password requisite pam_pwquality.so try_first_pass local_users_only \
    retry=3 minlen=12 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1/' /etc/pam.d/common-password

# Set password aging policies
sudo chage -M 90 -m 7 -W 7 -I 30 -E -1 root

# Set password aging for all users
for user in $(cut -d: -f1 /etc/passwd); do
    sudo chage -M 90 -m 7 -W 7 -I 30 -E -1 "$user" 2>/dev/null
done

# Verify password policy
passwd root
chage -l root

# Test password policy
echo "testuser:testpassword" | sudo chpasswd
# Should fail due to password length and complexity
```

### Scenario 4: Troubleshooting User Account Issues

A user cannot log in or access their home directory.

```bash
# Check user account
grep username /etc/passwd

# Check home directory
ls -la /home/username

# Check permissions
ls -ln /home/username

# Check shell
getent passwd username

# Check if account is locked
passwd -S username

# Check password aging
chage -l username

# Check group membership
id username

# Check home directory ownership
chown -R username:username /home/username

# Check permissions
chmod 700 /home/username
chmod 600 /home/username/*

# Check if account is locked
sudo usermod -U username

# Check PAM configuration
cat /etc/pam.d/common-auth
cat /etc/pam.d/common-password

# Check system logs
journalctl -u systemd-logind -f
journalctl -u sshd -f
```

---

## Troubleshooting

### User Account Issues

**Problem**: User cannot log in

```bash
# Check user account
grep username /etc/passwd

# Check password status
passwd -S username

# Check if account is locked
sudo usermod -L username
sudo usermod -U username

# Check password aging
chage -l username

# Check home directory
ls -la /home/username

# Check permissions
ls -ln /home/username

# Check shell
getent passwd username

# Check group membership
id username

# Check PAM configuration
cat /etc/pam.d/common-auth
cat /etc/pam.d/common-password

# Check system logs
journalctl -u systemd-logind -f
```

**Problem**: User cannot access home directory

```bash
# Check home directory permissions
ls -ln /home/username

# Fix permissions
chmod 700 /home/username
chown username:username /home/username

# Check if home directory exists
ls -la /home/username

# Recreate home directory
sudo usermod -d /home/username -m username

# Check file permissions
ls -la /home/username/*

# Fix file ownership
chown -R username:username /home/username

# Check for hidden files
ls -la /home/username/.*
```

**Problem**: User cannot change password

```bash
# Check if user is in shadow group
id username

# Check shadow file permissions
ls -la /etc/shadow

# Check PAM configuration
cat /etc/pam.d/common-password

# Check if password is locked
passwd -S username

# Unlock password
sudo passwd -u username

# Check if password has expired
chage -l username

# Reset password
sudo passwd username

# Check system logs
journalctl -u pam -f
```

### Group Issues

**Problem**: User cannot access group-owned files

```bash
# Check group membership
id username

# Check file group ownership
ls -ln /path/to/file

# Add user to group
sudo usermod -aG groupname username

# Check group permissions
ls -ln /path/to/file

# Fix group ownership
chown :groupname /path/to/file

# Fix group permissions
chmod g+rwx /path/to/file

# Check SGID bit
ls -ln /path/to/directory

# Set SGID bit
chmod g+s /path/to/directory

# Check group file permissions
ls -ln /path/to/file

# Fix group permissions
chmod 775 /path/to/directory
chmod 664 /path/to/file
```

### Sudo Issues

**Problem**: User cannot use sudo

```bash
# Check if user is in sudo group
id username

# Check sudoers configuration
sudo visudo

# Check sudoers syntax
visudo -cf /etc/sudoers

# Check sudo logs
grep username /var/log/secure

# Check sudo configuration
cat /etc/sudoers.d/*

# Check if user is in sudoers
grep username /etc/sudoers

# Check sudo permissions
sudo -l

# Check sudo logging
sudo logname

# Check sudo timeout
sudo -k

# Check sudoers file
cat /etc/sudoers

# Check sudoers.d directory
ls -la /etc/sudoers.d/
```

**Problem**: Sudo command fails

```bash
# Check sudoers configuration
sudo visudo

# Check sudoers syntax
visudo -cf /etc/sudoers

# Check sudo logs
grep username /var/log/secure

# Check sudo permissions
sudo -l

# Check sudo configuration
cat /etc/sudoers.d/*

# Check if user is in sudoers
grep username /etc/sudoers

# Check sudo timeout
sudo -k

# Check sudoers file
cat /etc/sudoers

# Check sudoers.d directory
ls -la /etc/sudoers.d/
```

### Password Issues

**Problem**: Password does not change

```bash
# Check password status
passwd -S username

# Check password aging
chage -l username

# Check if password is locked
sudo usermod -L username
sudo usermod -U username

# Check if password has expired
chage -l username

# Reset password
sudo passwd username

# Check PAM configuration
cat /etc/pam.d/common-password

# Check system logs
journalctl -u pam -f
```

**Problem**: Password policy not enforced

```bash
# Check password minimum length
cat /etc/login.defs | grep PASS_MIN_LEN

# Check PAM configuration
cat /etc/pam.d/common-password

# Check password quality
passwd -S username

# Check password aging
chage -l username

# Check system logs
journalctl -u pam -f
```

---

## Verification Procedures

### User Account Verification

```bash
# Verify user account exists
grep username /etc/passwd

# Verify user ID
id username

# Verify home directory
ls -la /home/username

# Verify shell
getent passwd username

# Verify group membership
id username

# Verify password status
passwd -S username

# Verify password aging
chage -l username

# Verify user can log in
su - username
```

### Group Verification

```bash
# Verify group exists
getent group groupname

# Verify group GID
grep groupname /etc/group

# Verify group members
getent group groupname

# Verify user is in group
id username

# Verify group permissions
ls -ln /path/to/file

# Verify group ownership
chown :groupname /path/to/file
```

### Sudo Verification

```bash
# Verify sudo configuration
sudo -l

# Verify user can use sudo
sudo whoami

# Verify sudo logs
grep username /var/log/secure

# Verify sudoers file
cat /etc/sudoers

# Verify sudoers.d directory
ls -la /etc/sudoers.d/
```

### Password Verification

```bash
# Verify password status
passwd -S username

# Verify password aging
chage -l username

# Verify password can be changed
passwd username

# Verify password policy
passwd root

# Verify PAM configuration
cat /etc/pam.d/common-password
```

---

## RHCSA Exam Notes

### Exam Relevance

Managing users and groups is heavily tested on the RHCSA. Expect 2-3 questions covering user creation, password management, group management, and sudo configuration.

### Key Exam Points

1. **User Creation**: Know `useradd` options for home directory, shell, groups, and UID.

2. **User Modification**: Know `usermod` options for shell, home directory, groups, and UID.

3. **User Deletion**: Know `userdel` options for removing home directory and mail spool.

4. **Password Management**: Know `passwd` and `chage` commands for password aging and locking.

5. **Group Management**: Know `groupadd`, `groupmod`, `groupdel` commands.

6. **Group Membership**: Know `usermod` for adding/removing users from groups.

7. **Sudo Configuration**: Know how to configure sudoers and manage sudo access.

8. **Security**: Know how to lock/unlock accounts and set password policies.

### Common Exam Traps

- Forgetting `-m` flag for creating home directory
- Not using `-a` flag when adding to supplementary groups
- Forgetting `sudo` for privileged operations
- Confusing primary and supplementary groups
- Not using `visudo` for editing sudoers
- Forgetting to enable sudo service

### Time Management Tips

- Use `useradd -m -s /bin/bash -c "Name" username` to create users quickly
- Use `usermod -aG group1,group2 username` to add users to multiple groups
- Use `passwd -l username` to lock accounts quickly
- Use `usermod -L username` to lock accounts
- Use `usermod -U username` to unlock accounts
- Use `id username` to check user and group membership

### Exam Environment Notes

- You will need to create user accounts manually
- Sudoers may need to be configured
- Password policies may need to be enforced
- Group memberships may need to be managed
- User shells may need to be changed

---

## Chapter Summary

This chapter covered user and group management including creating, modifying, and deleting user accounts, managing passwords and password aging, managing groups and group memberships, configuring privileged access with sudo, and troubleshooting common issues. You learned how to use `useradd`, `usermod`, `userdel`, `passwd`, `chage`, `groupadd`, `groupmod`, `groupdel`, and `usermod` for group membership management.

These skills are essential for system administration tasks including user account management, security configuration, and access control. Understanding user and group management helps administrators maintain secure and organized systems.

---

## Quick Reference

### User Commands

```bash
useradd -m username              # Create user with home directory
useradd -m -s /bin/bash username # Create user with shell
useradd -m -g group username     # Create user with primary group
useradd -m -G group1,group2 username # Create user with supplementary groups
useradd -m -e 2024-12-31 username  # Create user with expiration date
usermod -s /bin/zsh username      # Change user shell
usermod -d /home/newdir username  # Change home directory
usermod -aG group1,group2 username # Add user to groups
usermod -G group1 username        # Remove user from groups
usermod -L username               # Lock user account
usermod -U username               # Unlock user account
userdel -r username               # Delete user and home directory
```

### Password Commands

```bash
passwd username                   # Change user password
passwd -l username                # Lock user password
passwd -u username                # Unlock user password
passwd -e 2024-12-31 username     # Set password expiration
passwd -i username                # Reset password aging
chage -l username                 # List password aging
chage -M 90 username              # Set maximum password age
chage -m 7 username               # Set minimum password age
chage -W 7 username               # Set warning days
chage -I 30 username              # Set inactive days
chage -E 2024-12-31 username      # Set expiration date
```

### Group Commands

```bash
groupadd groupname                # Create group
groupadd -g 1001 groupname        # Create group with GID
groupmod -n newname oldname       # Change group name
groupmod -g 1001 groupname        # Change group GID
groupdel groupname                # Delete group
```

### Sudo Commands

```bash
visudo                            # Edit sudoers file
sudo -l                           # List user sudo permissions
sudo username                     # Run command as user
sudo -i                           # Login as root
sudo -s                          # Login as root
```

### User Information Commands

```bash
id username                       # Show user and group IDs
getent passwd username            # Show user entry
getent group groupname            # Show group entry
passwd -S username                # Show password status
chage -l username                 # Show password aging
grep username /etc/passwd         # Show user in passwd
```

---

## Review Questions

1. What is the difference between primary and supplementary groups?

2. How do you create a user account with a home directory and specific shell?

3. What command would you use to lock a user's account?

4. How do you add a user to multiple supplementary groups?

5. What is the purpose of the `-m` option in useradd?

6. How do you change a user's primary group?

7. What command would you use to view password aging information?

8. How do you set a password to expire in 30 days?

9. What is the purpose of the `visudo` command?

10. How do you remove a user from all supplementary groups?

11. What is the difference between `usermod -L` and `passwd -l`?

12. How do you create a system user that cannot log in?

13. What command would you use to change a user's home directory?

14. How do you delete a user account and their home directory?

15. What is the purpose of the `-a` flag in usermod?

---

## Answers

1. A primary group is a user's default group (first group in /etc/passwd), used for file ownership. Supplementary groups are additional groups a user belongs to, allowing access to resources owned by those groups. A user has only one primary group but can belong to multiple supplementary groups.

2. Use `sudo useradd -m -s /bin/bash username`. The `-m` option creates the home directory, and `-s` specifies the login shell.

3. Use `sudo passwd -l username` or `sudo usermod -L username` to lock a user's account. This prevents the user from logging in until the account is unlocked.

4. Use `sudo usermod -aG group1,group2 username`. The `-a` flag appends the user to the groups without removing them from existing supplementary groups.

5. The `-m` option in useradd creates the user's home directory if it doesn't exist. Without this option, the home directory is not created.

6. Use `sudo usermod -g newgroup username` to change a user's primary group. This changes the GID in the user's /etc/passwd entry.

7. Use `chage -l username` to list password aging information. This shows password age, minimum/max days, warning days, inactive days, and expiration date.

8. Use `sudo chage -E 2024-01-30 username` to set the password expiration date. The password will expire on January 30, 2024.

9. `visudo` is the safe way to edit the sudoers file. It checks for syntax errors before saving and prevents direct editing of the file.

10. Use `sudo usermod -G "" username` to remove a user from all supplementary groups. This sets the supplementary group list to empty.

11. `usermod -L` locks the account by adding an "L" to the /etc/shadow entry. `passwd -l` does the same thing. Both methods lock the account until unlocked with `usermod -U` or `passwd -u`.

12. Use `sudo useradd -r -s /usr/sbin/nologin -d /var/www -m www-data`. The `-r` flag creates a system account (UID < 1000), and `-s /usr/sbin/nologin` prevents login.

13. Use `sudo usermod -d /home/newdir username` to change a user's home directory. Use `sudo usermod -m username` to move the home directory if it exists.

14. Use `sudo userdel -r username` to delete a user account and their home directory. The `-r` flag removes the home directory and mail spool.

15. The `-a` flag in usermod appends the user to the specified groups without removing them from existing supplementary groups. Without `-a`, the user is removed from all supplementary groups and added only to the specified groups.
