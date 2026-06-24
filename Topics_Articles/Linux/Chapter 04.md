## _Chapter 04: Linux Package Management: apt, dnf, and zypper Compared_
###### Published Time: 2026-03-23


Package management is the backbone of every Linux system. It governs how software is installed, updated, removed, and audited. Whether you are provisioning a fresh cloud instance, hardening a production fleet, or debugging a dependency conflict at 2 a.m., fluency in your distribution's package manager is non-negotiable. This guide provides a deep, practical walkthrough of the three major ecosystems -- apt (Debian/Ubuntu), dnf (RHEL/Fedora/Rocky/Alma), and zypper (SUSE/openSUSE) -- along with universal package formats, building from source, repository management, security automation, and real-world troubleshooting scenarios.
<br><br>
### [Package Management Concepts](https://devopsil.com/articles/linux-package-management#package-management-concepts)

Before touching a single command, you need to understand the architecture that makes package management work.

- **Packages** are compressed archives that bundle compiled binaries, libraries, configuration files, documentation, and metadata scripts (pre-install, post-install, pre-remove, post-remove). The two dominant formats are DEB (used by Debian, Ubuntu, and derivatives) and RPM (used by Red Hat, Fedora, SUSE, and derivatives). Each package declares what it provides, what it depends on, what it conflicts with, and what it replaces.
- **Dependencies** are the libraries and tools a package requires to function. When you install nginx, for example, the package manager reads the dependency tree, resolves every transitive requirement, and installs them all in the correct order. This is the single biggest advantage of a package manager over manual installation.
- **Repositories** are remote (or local) servers hosting collections of packages and their metadata indexes. Your system is configured to query one or more repos, and each repo is signed with a GPG key so your package manager can verify that packages have not been tampered with.
- **Metadata cache** is a local snapshot of the repository index. Commands like `apt update`, `dnf makecache`, and `zypper refresh` download fresh metadata so your system knows what versions are available. Stale metadata is one of the most common causes of confusing installation failures.
- **GPG signing** ensures integrity and authenticity. Every reputable repository signs its packages and metadata with a GPG key. Your package manager refuses to install packages from unsigned or untrusted sources unless you explicitly override the check, which you should almost never do in production.
<br><br>
## [APT in Depth (Debian/Ubuntu)](https://devopsil.com/articles/linux-package-management#apt-in-depth-debianubuntu)

APT (Advanced Package Tool) is the default package manager for Debian, Ubuntu, Linux Mint, and all Debian-family distributions. Under the hood, APT uses dpkg for the actual package installation, while APT handles dependency resolution and repository management.

##### [Core Commands](https://devopsil.com/articles/linux-package-management#core-commands)

```
# Refresh the local package index from configured repositories
# Always run this before installing or upgrading
sudo apt update

# Upgrade all installed packages to their latest available versions
sudo apt upgrade                    # Safe: never removes packages
sudo apt full-upgrade               # May remove packages to resolve conflicts

# Install one or more packages
sudo apt install nginx curl vim

# Install a specific version
sudo apt install nginx=1.24.0-1ubuntu1

# Install without pulling in recommended (but optional) packages
# Useful for minimal server images and containers
sudo apt install --no-install-recommends nginx

# Simulate an install to see what would happen without making changes
sudo apt install --dry-run nginx

# Reinstall a package (useful when binaries get corrupted)
sudo apt reinstall nginx

# Remove a package but keep its configuration files
sudo apt remove nginx

# Remove a package and its configuration files
sudo apt purge nginx

# Remove packages that were installed as dependencies but are no longer needed
sudo apt autoremove

# Clean up the local package cache to free disk space
sudo apt clean                      # Remove all cached .deb files
sudo apt autoclean                  # Remove only obsolete cached .deb files
```

##### [Searching and Querying](https://devopsil.com/articles/linux-package-management#searching-and-querying)

```
# Search for packages by keyword
apt search "web server"

# List installed packages
apt list --installed

# List packages that have updates available
apt list --upgradable

# Show detailed information about a package
apt show nginx

# Show installed and candidate versions, plus repository sources
apt policy nginx

# Show which versions are available across all configured repos
apt-cache madison nginx

# Show a package's dependencies
apt-cache depends nginx

# Show reverse dependencies (what depends on this package)
apt-cache rdepends nginx

# Show detailed package metadata
apt-cache showpkg nginx
```

##### [dpkg: The Low-Level Tool](https://devopsil.com/articles/linux-package-management#dpkg-the-low-level-tool)

APT delegates to dpkg for the actual installation and removal of `.deb` files. You interact with dpkg directly when working with downloaded package files or querying the local database:

```
# Install a .deb file directly
sudo dpkg -i package.deb

# If dpkg fails due to missing dependencies, fix them with:
sudo apt install -f

# List all installed packages
dpkg -l

# Filter for a specific package
dpkg -l | grep nginx

# List every file installed by a package
dpkg -L nginx

# Find which package owns a specific file
dpkg -S /usr/sbin/nginx

# Show the contents of a .deb file without installing it
dpkg --contents package.deb

# Show the control information of a .deb file
dpkg --info package.deb

# Reconfigure a package (re-run its configuration prompts)
sudo dpkg-reconfigure tzdata
```

##### [Repository Management and Sources](https://devopsil.com/articles/linux-package-management#repository-management-and-sources)

Package sources are defined in `/etc/apt/sources.list` and individual files under `/etc/apt/sources.list.d/`. Modern Ubuntu (24.04+) uses the DEB822 format with `.sources` files, while older systems use the classic one-line format.

Classic one-line format:

```
# /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu jammy-security main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
```

The components (main, restricted, universe, multiverse) control which categories of software are available. `main` is officially supported, `universe` is community-maintained, `restricted` contains proprietary drivers, and `multiverse` contains software with legal restrictions.

##### [Adding PPAs and Third-Party Repos](https://devopsil.com/articles/linux-package-management#adding-ppas-and-third-party-repos)

```
# Add a PPA (Personal Package Archive, Ubuntu-specific)
sudo add-apt-repository ppa:ondrej/php
sudo apt update

# Add a third-party repository with a GPG key (modern best practice)
# Step 1: Download and store the GPG key
curl -fsSL https://packages.example.com/gpg.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg

# Step 2: Add the repository definition
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/example.gpg] https://packages.example.com/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/example.list

# Step 3: Update and install
sudo apt update
sudo apt install example-package

# Remove a PPA
sudo add-apt-repository --remove ppa:ondrej/php
```

The `signed-by` approach is the modern replacement for the deprecated `apt-key` command. Always use `/etc/apt/keyrings/` to store third-party GPG keys.

##### [APT Pinning](https://devopsil.com/articles/linux-package-management#apt-pinning)

APT pinning lets you control which repository or version takes priority. Create files in `/etc/apt/preferences.d/`:

```
# /etc/apt/preferences.d/pin-nginx
Package: nginx
Pin: version 1.24.0*
Pin-Priority: 1001
```

Pin priority values control behavior:

*   **1001 and above**: Install this version even if it means downgrading
*   **990**: Default priority for the target release
*   **500**: Default for non-target repos
*   **100**: Priority for already-installed packages
*   **-1**: Never install this package

You can also pin by repository origin:

```
# /etc/apt/preferences.d/prefer-security
Package: *
Pin: release a=jammy-security
Pin-Priority: 900
```

And hold packages to prevent any upgrades:

```
sudo apt-mark hold nginx
sudo apt-mark showhold
sudo apt-mark unhold nginx
```
<br><br>
### [DNF in Depth (RHEL/Fedora/Rocky/AlmaLinux)](https://devopsil.com/articles/linux-package-management#dnf-in-depth-rhelfedorarockyalmalinux)

DNF (Dandified YUM) replaced YUM starting with RHEL 8 and Fedora 22. On most modern systems, `yum` is a symlink to `dnf`, so legacy scripts continue to work. DNF offers faster dependency resolution, better memory usage, and a plugin architecture.

##### [Core Commands](https://devopsil.com/articles/linux-package-management#core-commands-1)

```
# Check for available updates without installing
sudo dnf check-update

# Upgrade all installed packages
sudo dnf upgrade

# Upgrade only security patches
sudo dnf upgrade --security

# Install a package
sudo dnf install nginx

# Install a specific version
sudo dnf install nginx-1.24.0-1.el9

# Install a local RPM file and resolve dependencies from repos
sudo dnf localinstall package.rpm

# Remove a package
sudo dnf remove nginx

# Remove unused dependencies
sudo dnf autoremove

# Reinstall a corrupted package
sudo dnf reinstall nginx

# Downgrade to the previous version
sudo dnf downgrade nginx

# Clean metadata and cached packages
sudo dnf clean all
sudo dnf clean metadata              # Only metadata
sudo dnf clean packages              # Only cached RPMs
sudo dnf makecache                   # Rebuild the cache
```

##### [Searching and Querying](https://devopsil.com/articles/linux-package-management#searching-and-querying-1)

```
# Search by keyword
dnf search "web server"

# List installed packages
dnf list installed

# List available updates
dnf list updates

# Show package details
dnf info nginx

# Find which package provides a file
dnf provides /usr/sbin/nginx

# Find which package provides a command
dnf provides "*/bin/dig"

# List package groups
dnf group list

# Install a package group
sudo dnf group install "Development Tools"

# Show group details
dnf group info "Development Tools"
```

##### [Module Streams (RHEL 8+)](https://devopsil.com/articles/linux-package-management#module-streams-rhel-8)

Module streams let you choose between different major versions of software. For example, you can pick Node.js 18 or Node.js 20 from the same repository:

```
# List available module streams
dnf module list

# List streams for a specific module
dnf module list nodejs

# Enable a specific stream
sudo dnf module enable nodejs:20

# Install the default profile of a module
sudo dnf module install nodejs:20

# Install a specific profile
sudo dnf module install nodejs:20/development

# Reset a module (before switching streams)
sudo dnf module reset nodejs
```

##### [Repository Management](https://devopsil.com/articles/linux-package-management#repository-management)

Repositories are defined as `.repo` files in `/etc/yum.repos.d/`:

```
# /etc/yum.repos.d/example.repo
[example]
name=Example Repository
baseurl=https://packages.example.com/rpm/$releasever/$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.example.com/gpg.key
sslverify=1
```

```
# List configured repos
dnf repolist
dnf repolist all                    # Including disabled repos

# Add a new repo from a URL
sudo dnf config-manager --add-repo https://packages.example.com/rpm/example.repo

# Enable or disable a repo
sudo dnf config-manager --set-enabled example
sudo dnf config-manager --set-disabled example

# Enable EPEL (Extra Packages for Enterprise Linux)
sudo dnf install epel-release

# Install from a specific repo only
sudo dnf install --repo=epel some-package
```

##### [DNF History and Rollbacks](https://devopsil.com/articles/linux-package-management#dnf-history-and-rollbacks)

One of DNF's most powerful features is full transaction history with rollback support:

```
# View transaction history
dnf history

# Show details of a specific transaction
dnf history info 15

# Undo a specific transaction (reverses only that transaction)
sudo dnf history undo 15

# Roll back to the state after a specific transaction
# This undoes all transactions after the specified one
sudo dnf history rollback 10

# Repeat a transaction (useful for replicating on another server)
sudo dnf history redo 15
```

##### [Version Locking with versionlock](https://devopsil.com/articles/linux-package-management#version-locking-with-versionlock)

```
# Install the versionlock plugin
sudo dnf install python3-dnf-plugin-versionlock

# Lock a package at its current version
sudo dnf versionlock add nginx

# List locked packages
sudo dnf versionlock list

# Remove a lock
sudo dnf versionlock delete nginx

# Clear all locks
sudo dnf versionlock clear
```

<br><br>
### [Zypper for SUSE (openSUSE/SLES)](https://devopsil.com/articles/linux-package-management#zypper-for-suse-opensusesles)

Zypper is the command-line interface for libzypp, the package management library used by SUSE Linux Enterprise and openSUSE. It supports RPM packages and shares many concepts with DNF but has its own syntax and some unique features.

##### [Core Commands](https://devopsil.com/articles/linux-package-management#core-commands-2)

```
# Refresh repository metadata
sudo zypper refresh
sudo zypper ref                      # Short form

# Install a package
sudo zypper install nginx
sudo zypper in nginx                 # Short form

# Install a specific version
sudo zypper install nginx=1.24.0

# Update all installed packages
sudo zypper update
sudo zypper up                       # Short form

# Perform a full distribution upgrade (for version upgrades)
sudo zypper dist-upgrade
sudo zypper dup

# Remove a package
sudo zypper remove nginx
sudo zypper rm nginx

# Remove a package and its unneeded dependencies
sudo zypper remove --clean-deps nginx

# Clean package caches
sudo zypper clean
sudo zypper clean --all              # Remove all cached data
```

##### [Searching and Querying](https://devopsil.com/articles/linux-package-management#searching-and-querying-2)

```
# Search by name
zypper search nginx
zypper se nginx                      # Short form

# Search in package descriptions too
zypper search -d "web server"

# List installed packages
zypper search -i

# Show package details
zypper info nginx

# Find what provides a specific file
zypper search --provides /usr/sbin/nginx

# Show dependencies
zypper info --requires nginx

# List available patches
zypper list-patches

# Install patterns (curated groups of packages)
zypper patterns
sudo zypper install -t pattern web_server
```

##### [Repository Management](https://devopsil.com/articles/linux-package-management#repository-management-1)

```
# List configured repos
zypper repos
zypper lr                            # Short form
zypper lr -d                         # Detailed listing with URIs

# Add a repository
sudo zypper addrepo https://packages.example.com/rpm/repo example-repo
sudo zypper ar https://packages.example.com/rpm/repo example-repo  # Short form

# Add a repo and immediately refresh it
sudo zypper addrepo -f https://packages.example.com/rpm/repo example-repo

# Remove a repo
sudo zypper removerepo example-repo
sudo zypper rr example-repo

# Enable or disable a repo
sudo zypper modifyrepo --enable example-repo
sudo zypper modifyrepo --disable example-repo

# Modify repo priority (lower number means higher priority)
sudo zypper modifyrepo --priority 90 example-repo

# Refresh a specific repo only
sudo zypper refresh example-repo
```

##### [Version Locking](https://devopsil.com/articles/linux-package-management#version-locking)

```
# Lock a package to prevent upgrades or removal
sudo zypper addlock nginx
sudo zypper al nginx                 # Short form

# List all locks
sudo zypper locks
sudo zypper ll

# Remove a lock
sudo zypper removelock nginx
sudo zypper rl nginx
```

<br><br>
### [Cross-Reference Table: apt vs dnf vs zypper](https://devopsil.com/articles/linux-package-management#cross-reference-table-apt-vs-dnf-vs-zypper)

This table is your cheat sheet when switching between distributions:

| Task | apt (Debian/Ubuntu) | dnf (RHEL/Fedora) | zypper (SUSE) |
| --- | --- | --- | --- |
| Refresh metadata | `apt update` | `dnf makecache` | `zypper refresh` |
| Upgrade all | `apt upgrade` | `dnf upgrade` | `zypper update` |
| Full upgrade | `apt full-upgrade` | `dnf distro-sync` | `zypper dup` |
| Install | `apt install pkg` | `dnf install pkg` | `zypper install pkg` |
| Install specific version | `apt install pkg=ver` | `dnf install pkg-ver` | `zypper install pkg=ver` |
| Remove | `apt remove pkg` | `dnf remove pkg` | `zypper remove pkg` |
| Purge (with configs) | `apt purge pkg` | `dnf remove pkg` | `zypper remove pkg` |
| Auto-remove deps | `apt autoremove` | `dnf autoremove` | `zypper rm --clean-deps` |
| Search | `apt search term` | `dnf search term` | `zypper search term` |
| Show info | `apt show pkg` | `dnf info pkg` | `zypper info pkg` |
| List installed | `apt list --installed` | `dnf list installed` | `zypper search -i` |
| File owner | `dpkg -S /path` | `rpm -qf /path` | `rpm -qf /path` |
| Clean cache | `apt clean` | `dnf clean all` | `zypper clean` |
| Lock version | `apt-mark hold pkg` | `dnf versionlock add pkg` | `zypper addlock pkg` |
| Transaction history | N/A (use /var/log/apt/) | `dnf history` | `zypper log` |
| Rollback | N/A | `dnf history undo N` | `snapper rollback` |

<br><br>
### [RPM vs DEB Package Formats](https://devopsil.com/articles/linux-package-management#rpm-vs-deb-package-formats)

Understanding the differences between RPM and DEB helps when you need to inspect packages, build your own, or migrate between distributions.

**DEB packages** are `ar` archives containing three components: `debian-binary` (format version), `control.tar.gz` (metadata, dependencies, scripts), and `data.tar.gz` (actual files). You inspect them with `dpkg-deb` and build them with `dpkg-buildpackage` or simpler tools like `fpm`.

**RPM packages** are cpio archives with a header containing metadata. They use `.spec` files to define the build process and are created with `rpmbuild`. RPM supports more granular triggers and scriptlet types than DEB.

| Feature | DEB | RPM |
| --- | --- | --- |
| File extension | `.deb` | `.rpm` |
| Low-level tool | `dpkg` | `rpm` |
| High-level tool | `apt` | `dnf` / `zypper` |
| Build tool | `dpkg-buildpackage` | `rpmbuild` |
| Build definition | `debian/` directory | `.spec` file |
| Architecture naming | `amd64`, `arm64` | `x86_64`, `aarch64` |
| Source packages | `.dsc` + `.orig.tar.gz` | `.src.rpm` |
| Used by | Debian, Ubuntu, Mint | RHEL, Fedora, SUSE, Rocky |

Converting between formats is possible with `alien`, but it is not recommended for production use due to differences in filesystem layout conventions and init system integration.

<br><br>
## [Building Packages from Source](https://devopsil.com/articles/linux-package-management#building-packages-from-source)

Sometimes the version you need is not available in any repository, or you need custom compile flags. Building from source is the fallback.

##### [The Classic Approach](https://devopsil.com/articles/linux-package-management#the-classic-approach)

```
# Install build dependencies
sudo apt install build-essential     # Debian/Ubuntu
sudo dnf group install "Development Tools"  # RHEL/Fedora

# Download and extract the source tarball
wget https://example.com/software-2.0.tar.gz
tar xzf software-2.0.tar.gz
cd software-2.0

# Configure: detects your system, checks for dependencies, sets install prefix
./configure --prefix=/usr/local --enable-ssl --with-pcre

# Compile the source code
make -j$(nproc)

# Install into the system
sudo make install
```

The problem with `make install` is that it scatters files across the filesystem with no tracking. Your package manager does not know about them, so upgrades, removals, and dependency checks all break.

##### [checkinstall: A Better Way](https://devopsil.com/articles/linux-package-management#checkinstall-a-better-way)

`checkinstall` wraps `make install` and creates a DEB or RPM package from the result, then installs that package. This means your package manager can track and remove the software cleanly:

```
sudo apt install checkinstall        # Debian/Ubuntu

# Instead of "sudo make install", run:
sudo checkinstall --pkgname=my-software --pkgversion=2.0 --default

# This creates and installs a .deb package
# You can remove it cleanly later:
sudo apt remove my-software
```

##### [Using fpm for Cross-Format Packaging](https://devopsil.com/articles/linux-package-management#using-fpm-for-cross-format-packaging)

`fpm` (Effing Package Management) is a tool that creates DEB, RPM, and other package formats from directories, tarballs, or other sources:

```
# Install fpm
sudo apt install ruby ruby-dev
sudo gem install fpm

# Create a DEB from a directory
fpm -s dir -t deb --name my-app --version 1.0 \
  --prefix /opt/my-app /path/to/build/output/

# Create an RPM from the same source
fpm -s dir -t rpm --name my-app --version 1.0 \
  --prefix /opt/my-app /path/to/build/output/
```

<br><br>
### [Snap, Flatpak, and AppImage](https://devopsil.com/articles/linux-package-management#snap-flatpak-and-appimage)

These universal packaging formats solve the "works on my distro" problem by bundling dependencies into self-contained packages.

**Snap** (developed by Canonical) uses a centralized store and automatic background updates. Snaps are confined by default using AppArmor and seccomp. They are common on Ubuntu Server for tools like LXD and certbot:

```
# List installed snaps
snap list

# Install a snap
sudo snap install code --classic    # --classic disables confinement

# Update all snaps
sudo snap refresh

# Revert to the previous version
sudo snap revert code

# Remove a snap
sudo snap remove code

# List available channels (versions/tracks)
snap info code
```

**Flatpak** (developed by Red Hat) focuses on desktop applications with a decentralized repository model. It uses bubblewrap for sandboxing and is default on Fedora Workstation:

```
# Add the Flathub repository
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Install an application
flatpak install flathub org.gimp.GIMP

# Run an application
flatpak run org.gimp.GIMP

# Update all flatpaks
flatpak update

# Remove an application and its unused runtimes
flatpak uninstall org.gimp.GIMP
flatpak uninstall --unused
```

**AppImage** takes a different approach with no installation at all. An AppImage is a single executable file that contains the application and all its dependencies:

```
# Download an AppImage
wget https://example.com/app-1.0-x86_64.AppImage

# Make it executable and run it
chmod +x app-1.0-x86_64.AppImage
./app-1.0-x86_64.AppImage

# Extract the contents (useful for inspection or integration)
./app-1.0-x86_64.AppImage --appimage-extract
```

**When to use which?** Use native packages (apt/dnf/zypper) for servers and system software. Use Snap when it is the officially supported delivery method (e.g., certbot on Ubuntu). Use Flatpak for desktop applications when the native repo version is outdated. Use AppImage when you need a portable, no-install option for a single user.

<br><br>
### [Repository Management](https://devopsil.com/articles/linux-package-management#repository-management-2)

##### [Creating a Local APT Repository](https://devopsil.com/articles/linux-package-management#creating-a-local-apt-repository)

A local repo is essential for air-gapped environments or when you need to distribute custom packages across your infrastructure:

```
# Install the tools
sudo apt install dpkg-dev

# Create the repository structure
sudo mkdir -p /opt/local-repo/pool
sudo cp *.deb /opt/local-repo/pool/

# Generate the package index
cd /opt/local-repo
dpkg-scanpackages pool /dev/null | gzip -9c > Packages.gz
dpkg-scanpackages pool /dev/null > Packages

# Add the local repo to your sources
echo "deb [trusted=yes] file:///opt/local-repo ./" | \
  sudo tee /etc/apt/sources.list.d/local.list
sudo apt update
```

##### [Creating a Local DNF Repository](https://devopsil.com/articles/linux-package-management#creating-a-local-dnf-repository)

```
# Install createrepo
sudo dnf install createrepo_c

# Create the repository structure
sudo mkdir -p /opt/local-repo
sudo cp *.rpm /opt/local-repo/

# Generate repository metadata
sudo createrepo_c /opt/local-repo

# Add the local repo
cat <<'REPOEOF' | sudo tee /etc/yum.repos.d/local.repo
[local]
name=Local Repository
baseurl=file:///opt/local-repo
enabled=1
gpgcheck=0
REPOEOF

sudo dnf makecache
```

##### [Caching Proxy with apt-cacher-ng](https://devopsil.com/articles/linux-package-management#caching-proxy-with-apt-cacher-ng)

When you manage dozens or hundreds of Ubuntu servers, each one downloading the same packages from the internet is wasteful. `apt-cacher-ng` acts as a transparent caching proxy:

```
# Install on the cache server
sudo apt install apt-cacher-ng

# The service listens on port 3142 by default
# Access the web UI at http://cache-server:3142

# Configure clients to use the proxy
echo 'Acquire::http::Proxy "http://cache-server:3142";' | \
  sudo tee /etc/apt/apt.conf.d/02proxy

# Or set the proxy for a specific repo only in sources.list:
# deb http://cache-server:3142/archive.ubuntu.com/ubuntu jammy main
```

For DNF-based systems, you can use a simple [Nginx](https://devopsil.com/tools/nginx "Nginx") reverse proxy with caching, or `yum-cron` with a local mirror created by `reposync`:

```
# Mirror a remote repo locally
sudo dnf install dnf-utils
sudo dnf reposync --repoid=baseos --download-metadata -p /opt/mirror/
```

<br><br>
### [Security Updates and Unattended Upgrades](https://devopsil.com/articles/linux-package-management#security-updates-and-unattended-upgrades)

##### [Debian/Ubuntu: unattended-upgrades](https://devopsil.com/articles/linux-package-management#debianubuntu-unattended-upgrades)

```
sudo apt install unattended-upgrades apt-listchanges
sudo dpkg-reconfigure -plow unattended-upgrades
```

Configure in `/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
};
Unattended-Upgrade::Package-Blacklist {
    "linux-image*";
    "linux-headers*";
};
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Mail "admin@example.com";
Unattended-Upgrade::MailReport "on-change";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
```

Test the configuration:

```
sudo unattended-upgrade --dry-run --debug
```

##### [RHEL/Fedora: dnf-automatic](https://devopsil.com/articles/linux-package-management#rhelfedora-dnf-automatic)

```
sudo dnf install dnf-automatic
sudo systemctl enable --now dnf-automatic-install.timer
```

Configure in `/etc/dnf/automatic.conf`:

```
[commands]
upgrade_type = security
apply_updates = yes
random_sleep = 3600

[emitters]
emit_via = email,stdio

[email]
email_from = root@server.example.com
email_to = admin@example.com
email_host = localhost
```

##### [SUSE: Automatic Patches](https://devopsil.com/articles/linux-package-management#suse-automatic-patches)

```
# Configure automatic patching
sudo zypper patch --auto-agree-with-licenses

# For automated operation, use a cron job or systemd timer:
# zypper --non-interactive patch --category security
```

<br><br>
### [GPG Key Management](https://devopsil.com/articles/linux-package-management#gpg-key-management)

All three ecosystems verify packages using GPG signatures. Proper key management is critical for supply-chain security.

```
# APT: modern approach using /etc/apt/keyrings/
curl -fsSL https://example.com/key.gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg
# Reference in sources.list with signed-by=/etc/apt/keyrings/example.gpg

# DNF/Zypper: import RPM GPG keys
sudo rpm --import https://example.com/key.gpg

# Or specify gpgkey= in the .repo file and dnf imports it automatically

# List imported keys on RPM-based systems
rpm -qa gpg-pubkey* --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n'

# Remove a GPG key on RPM-based systems
sudo rpm -e gpg-pubkey-XXXXXXXX-YYYYYYYY
```

<br><br>
## [Troubleshooting Dependency Issues](https://devopsil.com/articles/linux-package-management#troubleshooting-dependency-issues)

Dependency problems are inevitable. Here is a systematic approach for each ecosystem.

##### [APT Dependency Problems](https://devopsil.com/articles/linux-package-management#apt-dependency-problems)

```
# When apt install fails with unmet dependencies:
sudo apt --fix-broken install

# Check for broken packages
sudo dpkg --audit

# Force configuration of pending packages
sudo dpkg --configure -a

# As a last resort, force install ignoring dependencies (dangerous)
sudo dpkg -i --force-depends package.deb

# Simulate an installation to see what would change
apt install -s problematic-package

# Check what depends on a package before removing it
apt-cache rdepends --installed problematic-package
```

##### [DNF Dependency Problems](https://devopsil.com/articles/linux-package-management#dnf-dependency-problems)

```
# Check for duplicate packages
dnf repoquery --duplicates

# Remove duplicate packages
sudo dnf remove --duplicates

# Synchronize installed packages with the latest available versions
sudo dnf distro-sync

# Skip a broken package during upgrade
sudo dnf upgrade --exclude=broken-package

# Clear all caches and retry
sudo dnf clean all
sudo dnf makecache
sudo dnf upgrade
```

##### [Zypper Dependency Problems](https://devopsil.com/articles/linux-package-management#zypper-dependency-problems)

```
# Verify system integrity
sudo zypper verify

# Resolve dependency problems interactively
sudo zypper install --force-resolution problematic-package

# View solver test cases for debugging
sudo zypper install --debug-solver problematic-package
# Output goes to /var/log/zypper.solverTestCase/
```

<br><br>
### [Practical Scenarios](https://devopsil.com/articles/linux-package-management#practical-scenarios)

##### [Setting Up an Offline Repository](https://devopsil.com/articles/linux-package-management#setting-up-an-offline-repository)

In air-gapped environments (government, finance, classified networks), servers cannot reach the internet. You need an offline repository.

```
# On an internet-connected machine, download all packages for a base install:

# For APT:
sudo apt install apt-mirror
# Configure /etc/apt/mirror.list with the repos you need
sudo apt-mirror
# Copy /var/spool/apt-mirror to USB or transfer media
# On the air-gapped network, serve it via Nginx or use file:// URIs

# For DNF:
sudo dnf reposync --repoid=baseos --repoid=appstream \
  --download-metadata -p /media/usb/mirror/
# On the air-gapped server:
sudo dnf config-manager --add-repo file:///media/usb/mirror/baseos
sudo dnf config-manager --add-repo file:///media/usb/mirror/appstream
```

##### [Managing Updates Across a Fleet](https://devopsil.com/articles/linux-package-management#managing-updates-across-a-fleet)

When managing hundreds of servers, you do not run `apt upgrade` manually on each one. Use a staged approach:

```
# Stage 1: Mirror the upstream repo to your internal mirror
# Stage 2: Snapshot the mirror (so all servers get the same versions)
# Stage 3: Test the snapshot on staging servers
# Stage 4: Roll out to production in batches

# With Ansible, a simple playbook for APT:
# - name: Update all packages
#   apt:
#     update_cache: yes
#     upgrade: safe
#   when: inventory_hostname in groups['batch_1']

# For DNF, use dnf history to track what changed:
dnf history info last

# Export the transaction for documentation:
dnf history info last > /var/log/update-$(date +%Y%m%d).log
```

##### [Rolling Back a Broken Update](https://devopsil.com/articles/linux-package-management#rolling-back-a-broken-update)

An update broke your application. Here is how to recover on each platform:

```
# DNF: Use transaction history (the strongest rollback support)
dnf history                          # Find the transaction ID
sudo dnf history undo 42            # Reverse that specific transaction

# APT: No native rollback, but you can downgrade specific packages
apt policy nginx                     # Find the previous version
sudo apt install nginx=1.22.0-1ubuntu1  # Install the older version
sudo apt-mark hold nginx             # Prevent it from upgrading again

# Zypper + Snapper: SUSE integrates with btrfs snapshots
sudo snapper list                    # List filesystem snapshots
sudo snapper rollback 15             # Roll back to snapshot 15
```

##### [Investigating What Changed After an Update](https://devopsil.com/articles/linux-package-management#investigating-what-changed-after-an-update)

```
# APT: Check the log
cat /var/log/apt/history.log | tail -50

# DNF: Use history
dnf history info last

# Zypper: Check the log
cat /var/log/zypp/history | tail -50

# Find recently modified files (useful for debugging post-update issues)
find /etc -mmin -30 -type f 2>/dev/null
```

<br><br>
### [Key Takeaways](https://devopsil.com/articles/linux-package-management#key-takeaways)

Package management is not glamorous, but it is absolutely critical infrastructure. Here are the principles that matter most in production:

- **Always refresh metadata before installing.** Stale indexes cause confusing "package not found" errors and can result in installing outdated versions with known vulnerabilities. Run `apt update`, `dnf makecache`, or `zypper refresh` before every install operation.
- **Pin versions for critical packages in production.** Uncontrolled upgrades of databases, web servers, and runtime environments cause outages. Use `apt-mark hold`, `dnf versionlock`, or `zypper addlock` to freeze versions and upgrade deliberately during maintenance windows.
- **Enable automatic security updates.** The risk of an unpatched CVE is almost always greater than the risk of a security patch breaking something. Configure `unattended-upgrades` or `dnf-automatic` to apply security patches automatically, while holding back feature upgrades for manual review.
- **Know your rollback options before you need them.** DNF history rollback and SUSE's snapper integration can save you during a crisis. On Debian systems, keep old package versions in your cache (`apt clean` sparingly) so you can downgrade.
- **Use caching proxies and local mirrors at scale.** When you have more than a handful of servers, hitting upstream repos from every node wastes bandwidth and introduces a single point of failure. Tools like `apt-cacher-ng`, `reposync`, and `apt-mirror` keep your infrastructure self-sufficient.
- **Avoid building from source unless you have to.** When you must, use `checkinstall` or `fpm` to create proper packages so your package manager can track and remove the software. Untracked files from `make install` are a maintenance nightmare waiting to happen.
- **Always import GPG keys before adding third-party repositories.** Unsigned packages are a supply-chain attack vector. Verify the key fingerprint against the vendor's documentation, and use the modern `signed-by` approach on Debian systems instead of the deprecated `apt-key`.
