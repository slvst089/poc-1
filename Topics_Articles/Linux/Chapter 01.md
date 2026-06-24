## _Chapter 01: Linux Fundamentals: File System Navigation and Permissions_
###### Published Time: 2026-03-23

Every Linux system follows a predictable directory layout defined by the Filesystem Hierarchy Standard (FHS). Whether you are SSHing into a production server at 2 AM or writing a Dockerfile, knowing where things live and how to manipulate them is non-negotiable. This guide is a thorough walkthrough of the directory tree, the commands you will use daily, the permission model that underpins Linux security, text processing utilities, filesystem types, and real-world troubleshooting scenarios drawn from production experience.

### [The Filesystem Hierarchy Standard (FHS) in Detail](https://devopsil.com/articles/linux-fundamentals-filesystem#the-filesystem-hierarchy-standard-fhs-in-detail)

Linux organizes everything under a single root `/`. There are no drive letters. Every mounted device, every virtual filesystem, and every user directory hangs off this single tree. Understanding each top-level directory saves you hours of guesswork on unfamiliar systems.

##### [/ (Root)](https://devopsil.com/articles/linux-fundamentals-filesystem#-root)

The root directory is the starting point of the entire filesystem tree. Only the root user should have write access here. On most modern distributions, the contents of `/` are tightly controlled and you should never create arbitrary files directly in it.

##### [/etc (System Configuration)](https://devopsil.com/articles/linux-fundamentals-filesystem#etc-system-configuration)

This is where system-wide configuration files live. Almost every service you install drops a config file here.

```
/etc/nginx/nginx.conf          # Nginx web server config
/etc/ssh/sshd_config           # SSH daemon configuration
/etc/fstab                     # Filesystem mount table (read at boot)
/etc/hosts                     # Static hostname resolution
/etc/resolv.conf               # DNS resolver configuration
/etc/crontab                   # System-wide cron jobs
/etc/passwd                    # User account information
/etc/shadow                    # Encrypted passwords (restricted)
/etc/group                     # Group definitions
/etc/systemd/system/           # Custom systemd unit files
```

When something is broken on a Linux server, `/etc` is usually the first place you look. Treat it as the single source of truth for system configuration.

##### [/var (Variable Data)](https://devopsil.com/articles/linux-fundamentals-filesystem#var-variable-data)

Data that changes frequently during normal operation: logs, caches, mail spools, and database storage.

```
/var/log/syslog                # General system log (Debian/Ubuntu)
/var/log/messages              # General system log (RHEL/CentOS)
/var/log/nginx/access.log      # Nginx access logs
/var/log/auth.log              # Authentication events
/var/lib/docker/               # Docker images, containers, volumes
/var/lib/mysql/                # MySQL data directory
/var/cache/apt/                # APT package cache
/var/spool/cron/               # Per-user crontabs
```

Disk space problems on production servers almost always trace back to `/var/log` growing unchecked. Set up log rotation early.

##### [/home (User Home Directories)](https://devopsil.com/articles/linux-fundamentals-filesystem#home-user-home-directories)

Each non-root user gets a directory under `/home`. This is where personal files, SSH keys, shell configs, and user-specific application data live.

```
/home/aareez/.bashrc           # Bash configuration
/home/aareez/.ssh/             # SSH keys and config
/home/aareez/.config/          # XDG-compliant app configs
```

On servers, service accounts sometimes have their home directory set to `/var/lib/servicename` or `/nonexistent` instead.

##### [/tmp (Temporary Files)](https://devopsil.com/articles/linux-fundamentals-filesystem#tmp-temporary-files)

World-writable scratch space. Most distributions clear `/tmp` on reboot. Some use `tmpfs` which stores it entirely in RAM.

```
ls -ld /tmp
# drwxrwxrwt 15 root root 4096 ...
```

The sticky bit (the `t` at the end) prevents users from deleting each other's temporary files. Never store anything important here.

##### [/usr (User Programs and Libraries)](https://devopsil.com/articles/linux-fundamentals-filesystem#usr-user-programs-and-libraries)

The largest directory on most systems. Contains the bulk of installed software.

```
/usr/bin/                      # Most user commands (python3, git, curl)
/usr/sbin/                     # System administration commands
/usr/lib/                      # Shared libraries
/usr/share/                    # Architecture-independent data (man pages, docs)
/usr/local/                    # Locally compiled software (takes priority over /usr)
/usr/local/bin/                # Custom scripts you want in PATH
```

On modern distributions (Fedora, Ubuntu 20.04+), `/bin` and `/sbin` are symlinks to `/usr/bin` and `/usr/sbin`. This is the "usr merge."

##### [/opt (Optional/Third-Party Software)](https://devopsil.com/articles/linux-fundamentals-filesystem#opt-optionalthird-party-software)

Self-contained third-party packages that do not follow the standard `/usr` layout install here.

```
/opt/datadog-agent/
/opt/google/chrome/
/opt/containerd/
```

A common deployment pattern is to put application releases under `/opt/myapp/releases/` and symlink `/opt/myapp/current` to the active version.

##### [/proc (Process and Kernel Information)](https://devopsil.com/articles/linux-fundamentals-filesystem#proc-process-and-kernel-information)

A virtual filesystem that does not exist on disk. The kernel populates it with live system information.

```
cat /proc/cpuinfo              # CPU details
cat /proc/meminfo              # Memory statistics
cat /proc/loadavg              # Current load averages
cat /proc/1/status             # Status of PID 1 (init/systemd)
cat /proc/sys/net/ipv4/ip_forward   # Is IP forwarding enabled?
ls /proc/self/fd/              # File descriptors of the current process
```

Every running process has a numbered directory under `/proc`. This is how tools like `ps`, `top`, and `htop` get their data.

##### [/sys (Kernel Subsystem Data)](https://devopsil.com/articles/linux-fundamentals-filesystem#sys-kernel-subsystem-data)

Another virtual filesystem, organized around the kernel's device model. You interact with `/sys` when tuning hardware parameters or querying device state.

```
cat /sys/class/net/eth0/speed           # Network link speed
cat /sys/block/sda/queue/scheduler      # Disk I/O scheduler
cat /sys/class/thermal/thermal_zone0/temp  # CPU temperature
```

##### [/dev (Device Files)](https://devopsil.com/articles/linux-fundamentals-filesystem#dev-device-files)

Special files representing hardware and virtual devices.

```
/dev/sda                       # First SCSI/SATA disk
/dev/sda1                      # First partition on that disk
/dev/nvme0n1                   # First NVMe drive
/dev/null                      # Discards all input (the black hole)
/dev/zero                      # Infinite stream of zero bytes
/dev/urandom                   # Cryptographically secure random bytes
/dev/tty                       # Current terminal
```

The distinction between block devices (disks) and character devices (terminals, random) matters when you are writing udev rules or debugging storage.

| Path | Purpose |
| --- | --- |
| `/` | Root of the entire filesystem |
| `/etc` | System-wide configuration files |
| `/var` | Variable data: logs, caches, databases |
| `/home` | Per-user home directories |
| `/tmp` | Temporary files, cleared on reboot |
| `/usr` | Installed programs, libraries, documentation |
| `/opt` | Third-party or self-contained software |
| `/proc` | Virtual filesystem: kernel and process info |
| `/sys` | Virtual filesystem: kernel subsystem data |
| `/dev` | Device files (block and character devices) |
| `/boot` | Kernel images, initrd, GRUB config |
| `/mnt`, `/media` | Mount points for temporary and removable filesystems |

<br><br>
### [Essential Navigation Commands](https://devopsil.com/articles/linux-fundamentals-filesystem#essential-navigation-commands)

##### [pwd (Print Working Directory)](https://devopsil.com/articles/linux-fundamentals-filesystem#pwd-print-working-directory)

```
pwd
# /home/aareez/projects
```

Simple but critical in scripts where you need to confirm the working directory before performing destructive operations.

##### [cd (Change Directory)](https://devopsil.com/articles/linux-fundamentals-filesystem#cd-change-directory)

```
cd /var/log             # Absolute path
cd ..                   # Move up one directory
cd ../..                # Move up two directories
cd -                    # Jump back to the previous directory
cd ~                    # Return to home directory
cd                      # Also returns to home (shorthand)
```

A useful pattern for running a command in a different directory and returning automatically:

```
(cd /opt/myapp && ./deploy.sh)
# The subshell means your working directory does not change
```

##### [ls (List Directory Contents)](https://devopsil.com/articles/linux-fundamentals-filesystem#ls-list-directory-contents)

```
ls                      # Basic listing
ls -l                   # Long format with permissions, owner, size, date
ls -la                  # Long format, including hidden (dot) files
ls -lh                  # Human-readable sizes (K, M, G)
ls -lt                  # Sort by modification time, newest first
ls -ltr                 # Sort by modification time, oldest first
ls -lS                  # Sort by size, largest first
ls -R /etc/nginx        # Recursive listing
ls -d */                # List only directories
ls -i                   # Show inode numbers
```

The output of `ls -l` follows this format:

```
-rw-r--r-- 1 root root  4096 Mar 23 09:15 config.yaml
drwxr-xr-x 3 aareez aareez 4096 Mar 22 14:30 deployments/
```

The first character indicates file type: `-` regular file, `d` directory, `l` symlink, `b` block device, `c` character device, `p` named pipe, `s` socket.

##### [tree (Directory Tree Visualization)](https://devopsil.com/articles/linux-fundamentals-filesystem#tree-directory-tree-visualization)

```
tree /opt/myapp
# /opt/myapp
# ├── current -> releases/v2.3.1
# ├── releases
# │   ├── v2.3.0
# │   └── v2.3.1
# └── shared
#     ├── config
#     └── logs

tree -L 2 /etc          # Limit depth to 2 levels
tree -d /var             # Directories only
tree -I 'node_modules|.git'  # Ignore patterns
tree --du -h /var/log    # Show directory sizes
```

Install it with `sudo apt install tree` or `sudo yum install tree`. It is invaluable for understanding unfamiliar project layouts.

<br><br>
### [File Operations](https://devopsil.com/articles/linux-fundamentals-filesystem#file-operations)

##### [Creating Files and Directories](https://devopsil.com/articles/linux-fundamentals-filesystem#creating-files-and-directories)

```
touch newfile.txt                        # Create empty file or update timestamp
touch -t 202603231200 file.txt           # Set a specific timestamp
mkdir logs                               # Create a single directory
mkdir -p /opt/myapp/config/env           # Create nested directories in one shot
mkdir -m 750 /opt/secure                 # Create with explicit permissions
```

##### [Copying Files](https://devopsil.com/articles/linux-fundamentals-filesystem#copying-files)

```
cp source.txt dest.txt                   # Copy a file
cp -r src/ dest/                         # Copy directory recursively
cp -a src/ dest/                         # Archive mode: preserves permissions, timestamps, symlinks, ownership
cp -i file.txt /tmp/                     # Prompt before overwriting
cp -u *.conf /backup/                    # Copy only if source is newer than destination
cp --preserve=all important.dat /backup/ # Preserve all attributes
```

The difference between `-r` and `-a` matters in production. Use `-a` when you need an exact replica (backups, migrations). Use `-r` for casual copies.

##### [Moving and Renaming](https://devopsil.com/articles/linux-fundamentals-filesystem#moving-and-renaming)

```
mv old_name.conf new_name.conf           # Rename a file
mv file.txt /tmp/                        # Move a file
mv -i *.log /archive/                    # Prompt before overwriting
mv -n source.txt dest.txt               # Never overwrite
```

`mv` across filesystems is actually a copy-then-delete, which can be slow for large files.

##### [Removing Files and Directories](https://devopsil.com/articles/linux-fundamentals-filesystem#removing-files-and-directories)

```
rm file.txt                              # Remove a file
rm -r /tmp/build/                        # Remove directory recursively
rm -rf /tmp/build/                       # Force recursive removal (no prompts)
rm -ri /opt/old-release/                 # Interactive mode: confirm each file
rmdir empty_directory                    # Remove only if empty
```

A senior-level habit: use `rm -ri` on production systems for interactive confirmation. Better yet, use `trash-cli` to move files to a recoverable trash instead of deleting permanently.

##### [Finding Files with find](https://devopsil.com/articles/linux-fundamentals-filesystem#finding-files-with-find)

`find` searches the filesystem in real time. It is slower than indexed search but always returns current results.

```
# Basic name search
find /var/log -name "*.log"

# Case-insensitive search
find /etc -iname "*.conf"

# Find by modification time
find /var/log -name "*.log" -mtime -7        # Modified within last 7 days
find /var/log -name "*.log" -mtime +30       # Modified more than 30 days ago

# Find by size
find / -type f -size +100M                   # Files larger than 100MB
find /home -type f -size +1G 2>/dev/null     # Files larger than 1GB

# Find by type
find /opt -type d -name "config"             # Directories named config
find /usr -type l                            # All symbolic links

# Find by permissions
find / -type f -perm -o+w -not -path "/proc/*" 2>/dev/null   # World-writable files

# Execute commands on results
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;      # Compress old logs
find /tmp -type f -atime +7 -delete                           # Delete files not accessed in 7 days

# Using -exec with confirmation
find /opt -name "*.bak" -exec rm -i {} \;

# Efficient execution with + instead of \;
find /var/log -name "*.log" -exec ls -lh {} +                 # Passes multiple files at once
```

##### [Finding Files with locate](https://devopsil.com/articles/linux-fundamentals-filesystem#finding-files-with-locate)

`locate` uses a pre-built database, so it returns results instantly but may be stale.

```
sudo updatedb                        # Refresh the database
locate nginx.conf                    # Instant search
locate -i readme                     # Case-insensitive
locate -c "*.service"                # Count matches instead of listing
```

<br><br>
### [File Permissions in Depth](https://devopsil.com/articles/linux-fundamentals-filesystem#file-permissions-in-depth)

Linux permissions are expressed as three triplets: owner, group, and others. Each triplet has read (r=4), write (w=2), and execute (x=1) bits.

##### [Reading Permissions](https://devopsil.com/articles/linux-fundamentals-filesystem#reading-permissions)

```
-rwxr-xr-- 1 deploy webteam 8192 Mar 23 10:00 deploy.sh
```

Breaking this down:

| Component | Value | Meaning |
| --- | --- | --- |
| Owner (`deploy`) | rwx = 4+2+1 = 7 | Full control |
| Group (`webteam`) | r-x = 4+0+1 = 5 | Read and execute |
| Others | r-- = 4+0+0 = 4 | Read only |
| **Octal** | **754** |  |

What each permission means varies by file type:

| Permission | On a File | On a Directory |
| --- | --- | --- |
| Read (r) | View file contents | List directory contents |
| Write (w) | Modify file contents | Create/delete files in directory |
| Execute (x) | Run as a program | Enter the directory (cd into it) |

A directory without execute permission is useless even if read is set. You can list its contents but you cannot access any file inside it.

##### [Modifying Permissions with chmod](https://devopsil.com/articles/linux-fundamentals-filesystem#modifying-permissions-with-chmod)

```
# Symbolic notation
chmod u+x script.sh              # Add execute for owner
chmod g-w,o-r config.yaml        # Remove write from group, read from others
chmod a+r public.html            # Add read for all (a = all)
chmod u=rwx,g=rx,o= private.sh  # Set exact permissions

# Octal notation (most common in DevOps)
chmod 755 deploy.sh              # rwxr-xr-x: standard for executable scripts
chmod 644 config.yaml            # rw-r--r--: standard for config files
chmod 600 id_rsa                 # rw-------: private keys must be this
chmod 700 .ssh/                  # rwx------: SSH directory
chmod 666 /dev/null              # rw-rw-rw-: everyone can read/write

# Recursive
chmod -R 755 /var/www/html/      # Apply to all files and directories
```

Common permission patterns:

| Octal | Symbolic | Typical Use |
| --- | --- | --- |
| 755 | rwxr-xr-x | Executable scripts, directories |
| 644 | rw-r--r-- | Configuration files, static content |
| 600 | rw------- | Private keys, secrets |
| 700 | rwx------ | .ssh directory, private scripts |
| 775 | rwxrwxr-x | Shared project directories |
| 400 | r-------- | Read-only sensitive files |

##### [Changing Ownership with chown and chgrp](https://devopsil.com/articles/linux-fundamentals-filesystem#changing-ownership-with-chown-and-chgrp)

```
chown deploy:webteam deploy.sh           # Change owner and group
chown deploy deploy.sh                   # Change owner only
chown :webteam deploy.sh                 # Change group only
chown -R www-data:www-data /var/www/     # Recursive ownership change
chgrp docker /var/run/docker.sock        # Change group only (alternative)
```

##### [Default Permissions: umask](https://devopsil.com/articles/linux-fundamentals-filesystem#default-permissions-umask)

When a file is created, the kernel applies the umask to determine default permissions.

```
umask              # Show current umask (typically 0022)
umask 0027         # Set umask: files get 640, directories get 750
```

The formula: directory permissions = 777 minus umask, file permissions = 666 minus umask. With a umask of 0022, new files get 644 and new directories get 755.

```
# See it in action
umask 0022
touch testfile
mkdir testdir
ls -l testfile    # -rw-r--r-- (644)
ls -ld testdir    # drwxr-xr-x (755)
```

<br><br>
### [Special Permissions](https://devopsil.com/articles/linux-fundamentals-filesystem#special-permissions)

##### [Setuid (4xxx)](https://devopsil.com/articles/linux-fundamentals-filesystem#setuid-4xxx)

When set on an executable, it runs with the file owner's privileges, not the caller's. The classic example is `/usr/bin/passwd`, which needs root access to write `/etc/shadow`.

```
chmod u+s /usr/bin/myutil        # Set setuid
chmod 4755 /usr/bin/myutil       # Same in octal
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root 68208 ...
```

The `s` in the owner execute position indicates setuid. If the underlying execute bit is not set, it appears as `S` (capital), meaning the setuid is set but the file is not actually executable, which is usually a misconfiguration.

**Security note**: setuid binaries owned by root are prime targets for privilege escalation. Audit them regularly:

```
find / -type f -perm -4000 2>/dev/null
```

##### [Setgid (2xxx)](https://devopsil.com/articles/linux-fundamentals-filesystem#setgid-2xxx)

On a file, it runs with the group's privileges. On a directory, it forces new files to inherit the directory's group rather than the creating user's primary group. This is extremely useful for shared project directories.

```
chmod g+s /shared/project/       # New files inherit the group
chmod 2775 /shared/project/

# Demonstration
mkdir /shared/project
chown :developers /shared/project
chmod 2775 /shared/project
# Now when any user in 'developers' creates a file here,
# it automatically belongs to the 'developers' group
```

##### [Sticky Bit (1xxx)](https://devopsil.com/articles/linux-fundamentals-filesystem#sticky-bit-1xxx)

On a directory, only the file owner (or root) can delete files within it, even if others have write permission. `/tmp` uses this to prevent users from deleting each other's temporary files.

```
chmod +t /shared/uploads/        # Set sticky bit
chmod 1777 /tmp                  # Classic /tmp permissions
ls -ld /tmp
# drwxrwxrwt 15 root root 4096 ...
```

The `t` in the others execute position confirms the sticky bit.

| Special Bit | Octal Prefix | On File | On Directory |
| --- | --- | --- | --- |
| Setuid | 4 | Runs as file owner | No effect |
| Setgid | 2 | Runs as file group | New files inherit directory group |
| Sticky | 1 | No effect | Only owner/root can delete files |

<br><br>
### [Hard Links vs Soft Links](https://devopsil.com/articles/linux-fundamentals-filesystem#hard-links-vs-soft-links)

##### [Symbolic (Soft) Links](https://devopsil.com/articles/linux-fundamentals-filesystem#symbolic-soft-links)

A symlink is a pointer to a pathname. It can cross filesystems and link to directories. If the target is deleted, the symlink becomes a dangling (broken) link.

```
# Create a symlink
ln -s /etc/nginx/sites-available/mysite.conf /etc/nginx/sites-enabled/mysite.conf

# Inspect it
ls -l /etc/nginx/sites-enabled/
# lrwxrwxrwx 1 root root 42 ... mysite.conf -> /etc/nginx/sites-available/mysite.conf

# Common DevOps pattern: version switching with atomic symlink replacement
ln -sfn /opt/myapp/releases/v2.3.1 /opt/myapp/current
# -s = symbolic, -f = force (overwrite), -n = don't dereference existing symlink

# Check if a symlink is broken
file /opt/myapp/current
# /opt/myapp/current: symbolic link to /opt/myapp/releases/v2.3.1

# Find all broken symlinks in a directory
find /etc -xtype l 2>/dev/null
```

##### [Hard Links](https://devopsil.com/articles/linux-fundamentals-filesystem#hard-links)

A hard link is another directory entry pointing to the same inode (the actual data on disk). It cannot cross filesystems or link to directories. Deleting one hard link does not affect the others because the data persists as long as at least one link exists.

```
# Create a hard link
ln config.yaml config.yaml.bak

# Both share the same inode number
ls -li config.yaml config.yaml.bak
# 1234567 -rw-r--r-- 2 root root 512 ... config.yaml
# 1234567 -rw-r--r-- 2 root root 512 ... config.yaml.bak
```

Same inode number, link count of 2. Changes to one are reflected in the other because they are the same data on disk.

| Feature | Hard Link | Symbolic Link |
| --- | --- | --- |
| Cross filesystems | No | Yes |
| Link to directories | No | Yes |
| Survives target deletion | Yes (data persists) | No (becomes dangling) |
| Has own inode | No (shares inode) | Yes (own inode) |
| Performance | Slightly faster (no extra lookup) | Extra lookup to resolve path |

Practical use case for hard links: backup systems like `rsnapshot` use hard links to create space-efficient snapshots. Files that have not changed share the same blocks on disk.

<br><br>
### [Pipes and Redirection](https://devopsil.com/articles/linux-fundamentals-filesystem#pipes-and-redirection)

##### [Standard Streams](https://devopsil.com/articles/linux-fundamentals-filesystem#standard-streams)

Every process has three standard streams: stdin (file descriptor 0), stdout (file descriptor 1), stderr (file descriptor 2).

```
# Redirect stdout to a file (overwrite)
ls /etc > filelist.txt

# Redirect stdout to a file (append)
echo "deploy completed" >> /var/log/deploy.log

# Redirect stderr to a file
find / -name "*.conf" 2> errors.txt

# Redirect both stdout and stderr to the same file
command > output.txt 2>&1
command &> output.txt              # Bash shorthand for the same thing

# Redirect stdout and stderr to different files
command 1> stdout.txt 2> stderr.txt

# Discard all output
command > /dev/null 2>&1

# Read stdin from a file
sort < unsorted.txt > sorted.txt
```

##### [The Pipe Operator](https://devopsil.com/articles/linux-fundamentals-filesystem#the-pipe-operator)

Pipes connect the stdout of one command to the stdin of the next. This is the foundation of the Unix philosophy: small tools that each do one thing well, composed together.

```
# Count running Docker containers
docker ps | wc -l

# Find the 10 largest files in /var
du -ah /var 2>/dev/null | sort -rh | head -10

# Extract unique IPs from an access log, sorted by frequency
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Filter processes and extract PIDs
ps aux | grep '[n]ginx' | awk '{print $2}'

# Chain multiple filters
cat /var/log/auth.log | grep "Failed password" | awk '{print $11}' | sort | uniq -c | sort -rn
```

##### [tee (Write to File and stdout Simultaneously)](https://devopsil.com/articles/linux-fundamentals-filesystem#tee-write-to-file-and-stdout-simultaneously)

```
# Save output to file while also displaying it
df -h | tee disk_report.txt

# Append to file instead of overwriting
free -h | tee -a system_report.txt

# Write to multiple files
echo "config change" | tee /var/log/changes.log /var/log/audit.log

# Use with sudo when redirection alone does not work
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts
```

That last pattern is essential. `sudo echo "text" > /etc/hosts` fails because the redirection is handled by the current (non-root) shell. Piping through `sudo tee` solves this.

##### [Here Documents and Here Strings](https://devopsil.com/articles/linux-fundamentals-filesystem#here-documents-and-here-strings)

```
# Here document: multi-line input
cat <<EOF > /etc/nginx/conf.d/mysite.conf
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
}
EOF

# Here string: single-line input
grep "error" <<< "this is an error message"
```

<br><br>
### [Text Processing Commands](https://devopsil.com/articles/linux-fundamentals-filesystem#text-processing-commands)

These commands form the backbone of log analysis, data extraction, and automation scripting on Linux.

##### [Viewing File Contents](https://devopsil.com/articles/linux-fundamentals-filesystem#viewing-file-contents)

```
cat file.txt                     # Print entire file
cat -n file.txt                  # Print with line numbers
less /var/log/syslog             # Scrollable pager (q to quit, / to search)
head -20 access.log              # First 20 lines
head -c 1024 binary.dat          # First 1024 bytes
tail -20 access.log              # Last 20 lines
tail -f /var/log/syslog          # Follow new output in real time
tail -f /var/log/syslog | grep --line-buffered "error"   # Follow and filter
```

`tail -f` is the single most common command for live debugging on production servers.

##### [grep (Search Text with Patterns)](https://devopsil.com/articles/linux-fundamentals-filesystem#grep-search-text-with-patterns)

```
grep "ERROR" /var/log/app.log                    # Basic search
grep -i "error" /var/log/app.log                 # Case-insensitive
grep -r "database_url" /etc/                     # Recursive search through directory
grep -rn "TODO" /opt/myapp/src/                  # Recursive with line numbers
grep -c "404" access.log                         # Count matching lines
grep -v "GET /health" access.log                 # Invert match (exclude lines)
grep -l "password" /etc/*.conf                   # List only filenames with matches
grep -E "error|warning|critical" app.log         # Extended regex (OR)
grep -P "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" access.log   # Perl regex for IPs
grep -A 3 "Exception" app.log                    # Show 3 lines after match
grep -B 2 "Exception" app.log                    # Show 2 lines before match
grep -C 5 "Exception" app.log                    # Show 5 lines of context (before and after)
```

##### [awk (Column-Based Text Processing)](https://devopsil.com/articles/linux-fundamentals-filesystem#awk-column-based-text-processing)

```
# Print specific columns
awk '{print $1, $7}' access.log                  # IP address and URL path

# Filter by condition
awk '$9 == 500 {print $1, $7}' access.log        # Requests that returned 500

# Sum a column
awk '{sum += $10} END {print sum}' access.log     # Total bytes transferred

# Custom field separator
awk -F: '{print $1, $3}' /etc/passwd             # Username and UID

# Formatted output
awk -F: '{printf "%-20s %s\n", $1, $7}' /etc/passwd   # Aligned columns with shell info
```

##### [sed (Stream Editor)](https://devopsil.com/articles/linux-fundamentals-filesystem#sed-stream-editor)

```
# Replace first occurrence per line
sed 's/old/new/' file.txt

# Replace all occurrences (global)
sed 's/old/new/g' file.txt

# Edit file in-place
sed -i 's/debug=true/debug=false/g' config.ini

# In-place with backup
sed -i.bak 's/port=8080/port=9090/g' config.ini

# Delete lines matching a pattern
sed '/^#/d' config.conf                          # Remove comment lines
sed '/^$/d' file.txt                             # Remove blank lines

# Print specific line range
sed -n '10,20p' file.txt                         # Print lines 10 through 20

# Insert a line before/after a match
sed '/\[server\]/a\    bind_address = 0.0.0.0' config.ini
```

##### [cut, sort, uniq, wc](https://devopsil.com/articles/linux-fundamentals-filesystem#cut-sort-uniq-wc)

```
# cut: extract columns by delimiter
cut -d: -f1,3 /etc/passwd                        # Username and UID
cut -d',' -f2-4 data.csv                         # Fields 2 through 4 from CSV
cut -c1-10 file.txt                              # First 10 characters of each line

# sort: order lines
sort file.txt                                     # Alphabetical sort
sort -n numbers.txt                               # Numeric sort
sort -r file.txt                                  # Reverse sort
sort -t: -k3 -n /etc/passwd                      # Sort by UID (third field, colon-delimited)
sort -u file.txt                                  # Sort and remove duplicates
sort -h sizes.txt                                 # Human-readable numeric sort (1K, 2M, 3G)

# uniq: remove consecutive duplicates (input must be sorted)
sort access.log | uniq                            # Remove duplicate lines
sort access.log | uniq -c                         # Count occurrences
sort access.log | uniq -d                         # Show only duplicate lines

# wc: count lines, words, characters
wc -l access.log                                 # Count lines
wc -w document.txt                               # Count words
wc -c file.bin                                   # Count bytes
find /var/log -name "*.log" | wc -l              # Count log files
```

##### [Practical One-Liners](https://devopsil.com/articles/linux-fundamentals-filesystem#practical-one-liners)

```
# Top 10 IPs by request count from Nginx access log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Requests per minute from timestamped log
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f1-3 | sort | uniq -c | tail -20

# Disk usage by directory, sorted
du -sh /var/*/ 2>/dev/null | sort -rh

# Find all listening ports with associated processes
ss -tlnp | awk 'NR>1 {print $4, $6}'

# Count HTTP status codes
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

<br><br>
### [Filesystem Types and Mounting](https://devopsil.com/articles/linux-fundamentals-filesystem#filesystem-types-and-mounting)

##### [Common Linux Filesystem Types](https://devopsil.com/articles/linux-fundamentals-filesystem#common-linux-filesystem-types)

| Filesystem | Max File Size | Max Volume Size | Key Features |
| --- | --- | --- | --- |
| ext4 | 16 TB | 1 EB | Default on most distros, journaling, reliable, mature |
| XFS | 8 EB | 8 EB | Default on RHEL, excellent with large files, online grow only |
| Btrfs | 16 EB | 16 EB | Copy-on-write, snapshots, compression, checksums |

**ext4** is the safe default. It has decades of battle testing and handles most workloads well.

**XFS** excels with large files and high-throughput workloads. It cannot be shrunk, only grown. RHEL/CentOS/Rocky default to XFS.

**Btrfs** offers advanced features like transparent compression, snapshots, and built-in RAID. It is the default on openSUSE and Fedora Workstation. Use it when you need snapshot-based backup strategies.

##### [Mounting Filesystems](https://devopsil.com/articles/linux-fundamentals-filesystem#mounting-filesystems)

```
# List all mounted filesystems
mount | column -t
df -hT                           # Show filesystem types and usage

# Mount a device
sudo mount /dev/sdb1 /mnt/data

# Mount with specific type and options
sudo mount -t ext4 -o noatime,nodiratime /dev/sdb1 /mnt/data

# Mount an NFS share
sudo mount -t nfs 192.168.1.100:/shared /mnt/nfs

# Unmount
sudo umount /mnt/data

# Persistent mount via /etc/fstab
# Device            Mountpoint    Type   Options                    Dump  Pass
# /dev/sdb1         /mnt/data     ext4   defaults,noatime           0     2
```

After editing `/etc/fstab`, test without rebooting:

```
sudo mount -a                    # Mount all entries in fstab
```

If that fails, fix fstab before rebooting. A broken fstab entry can prevent a server from booting.

##### [Checking and Repairing Filesystems](https://devopsil.com/articles/linux-fundamentals-filesystem#checking-and-repairing-filesystems)

```
# Check filesystem (must be unmounted or read-only)
sudo fsck /dev/sdb1

# Force check on ext4
sudo e2fsck -f /dev/sdb1

# Check XFS (can run on mounted read-only)
sudo xfs_repair /dev/sdb1

# View filesystem info
sudo tune2fs -l /dev/sdb1       # ext4 details
sudo xfs_info /dev/sdb1         # XFS details
```

<br><br>
### [Practical Troubleshooting Scenarios](https://devopsil.com/articles/linux-fundamentals-filesystem#practical-troubleshooting-scenarios)

##### [Scenario 1: Disk Full on Production Server](https://devopsil.com/articles/linux-fundamentals-filesystem#scenario-1-disk-full-on-production-server)

The monitoring alert fires: root filesystem at 98%.

```
# Step 1: Find what is consuming space
df -h                            # Confirm which mount is full
du -sh /* 2>/dev/null | sort -rh | head -10  # Top-level breakdown

# Step 2: Drill into the largest directory (usually /var)
du -sh /var/*/ 2>/dev/null | sort -rh | head -10

# Step 3: Find specific large files
find /var/log -type f -size +100M -exec ls -lh {} +

# Step 4: Check for deleted files still held open by processes
lsof +L1 2>/dev/null | grep deleted

# Step 5: Compress or remove old logs
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;
find /var/log -name "*.gz" -mtime +90 -delete

# Step 6: If deleted-but-open files are the culprit, restart the process
# or truncate the file handle:
truncate -s 0 /var/log/huge-app.log
```

That `lsof +L1` step is critical. A common gotcha: you delete a 10 GB log file, `df` still shows the disk as full, because the process that had the file open still holds the file descriptor. The space is not freed until the process releases the handle.

##### [Scenario 2: Permission Denied on a Deployment](https://devopsil.com/articles/linux-fundamentals-filesystem#scenario-2-permission-denied-on-a-deployment)

Your CI/CD pipeline fails with "Permission denied" when writing to `/opt/myapp`.

```
# Step 1: Check the current state
ls -la /opt/myapp/
namei -l /opt/myapp/releases/v2.4.0

# namei traces the entire path and shows permissions at each level
# This catches the case where /opt itself is the bottleneck

# Step 2: Fix ownership for the deploy user
sudo chown -R deploy:deploy /opt/myapp
sudo chmod 750 /opt/myapp

# Step 3: Use setgid for shared access
sudo chmod 2770 /opt/myapp/shared/logs
# Now all team members in the 'deploy' group can write logs
```

`namei -l` is an underused tool that saves significant debugging time. It walks every component of a path and shows permissions at each level, instantly revealing which directory in the chain is blocking access.

##### [Scenario 3: Secure a New Server](https://devopsil.com/articles/linux-fundamentals-filesystem#scenario-3-secure-a-new-server)

```
# Audit world-writable files (CIS benchmark check)
find / -xdev -type f -perm -o+w -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null

# Audit setuid/setgid binaries
find / -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null | sort

# Lock down SSH directory
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 600 ~/.ssh/authorized_keys

# Secure deployment directories
sudo mkdir -p /opt/myapp/{releases,shared/config,shared/logs}
sudo chown -R deploy:deploy /opt/myapp
sudo chmod 750 /opt/myapp
sudo chmod 2770 /opt/myapp/shared/logs
```

##### [Scenario 4: Debug a Broken Symlink Chain](https://devopsil.com/articles/linux-fundamentals-filesystem#scenario-4-debug-a-broken-symlink-chain)

Application returns "file not found" but the config file visibly exists.

```
# Check if it is a broken symlink
file /etc/myapp/config.yaml
# If broken: /etc/myapp/config.yaml: broken symbolic link to /opt/myapp/shared/config.yaml

# Trace the symlink chain
readlink -f /etc/myapp/config.yaml    # Show final resolved path
namei -l /etc/myapp/config.yaml       # Show each step with permissions

# Find all broken symlinks on the system
find /etc -xtype l 2>/dev/null

# Fix by recreating the symlink
ln -sfn /opt/myapp/current/config.yaml /etc/myapp/config.yaml
```

##### [Scenario 5: Recover Space from Docker](https://devopsil.com/articles/linux-fundamentals-filesystem#scenario-5-recover-space-from-docker)

Docker is a notorious disk space consumer. When `/var/lib/docker` fills up:

```
# Check Docker disk usage
docker system df

# Remove unused containers, images, networks, and build cache
docker system prune -a --volumes

# Check overlay2 storage driver usage
du -sh /var/lib/docker/overlay2/ 2>/dev/null

# Find containers writing the most data
docker ps -s --format "table {{.Names}}\t{{.Size}}"
```

<br><br>
### [Key Takeaways](https://devopsil.com/articles/linux-fundamentals-filesystem#key-takeaways)

The Linux filesystem is not just a place to store files. It is the operating system's contract with every program, user, and administrator. Knowing the FHS means you never waste time guessing where configuration lives. Understanding permissions means you can lock down a production server without breaking the application. Mastering pipes and redirection means you can chain simple tools into powerful one-liners that solve real problems. Knowing your filesystem types means you can make informed decisions about storage.

Build muscle memory with these commands. Run them on a throwaway VM until `chmod 644`, `find -exec`, and `2>&1` feel like second nature. Everything else in DevOps (containers, CI/CD, infrastructure as code) builds on this foundation.
