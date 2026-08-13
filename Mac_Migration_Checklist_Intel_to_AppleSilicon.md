# Mac Migration Checklist — MacBook Pro 2019 (Intel) → M4/M5 Pro (Apple Silicon)

**Constraint:** No iCloud. OneDrive is the only cloud target.
**Note:** M4 Pro and M5 Pro are both `arm64`. Everything below applies identically to either.

---

## 0. Strategy Decision — Read This First

### 0.1 Do NOT use Migration Assistant for a dev machine

Migration Assistant works Intel → Apple Silicon, but it will copy your **Intel Homebrew tree from `/usr/local`** onto a machine where Homebrew belongs in `/opt/homebrew`. You end up with two prefixes, x86 binaries silently running under Rosetta, and `pyenv`/`nvm`/`rbenv` shims pointing at compiled x86 artifacts. Debugging that costs more than a clean setup.

**Recommendation:** Clean install + selective restore (this document). Budget ~4–6 hours of hands-on time on the new machine.

### 0.2 Two backup targets, not one

| Target | Purpose | Why |
|---|---|---|
| External SSD (Time Machine + a raw copy) | Full safety net, preserves permissions/xattrs/symlinks | OneDrive does not |
| OneDrive | Off-site copy of documents + one encrypted archive | Survives SSD failure/theft |

Do not rely on OneDrive alone. See §7 for why.

### 0.3 If this is a company-managed Mac (Jamf / Intune / Kandji)

- [ ] Confirm with IT before wiping — they may need to release the old device serial from MDM
- [ ] Do **not** back up corporate CA certs, VPN profiles, or SCEP identities — MDM re-pushes these
- [ ] Confirm whether OneDrive is the corporate tenant (`OneDrive - <Company>`) — **never upload SSH private keys or cloud credentials there**, even encrypted, if policy forbids it
- [ ] Get the FileVault recovery key or escrow confirmation before you touch anything

---

## 1. Pre-Flight on the Old Mac (T-7 days)

- [ ] Create working folder: `mkdir -p ~/mac-migration/{inventory,configs,exports}`
- [ ] Update old Mac to latest macOS it supports (makes keychain/format compat cleaner)
- [ ] Check free space on old Mac — you need room for the archive
- [ ] Buy/borrow an external SSD (1TB min) if you don't have one
- [ ] Note your current macOS version: `sw_vers > ~/mac-migration/inventory/os-version.txt`

### 1.1 Inventory dump (run this whole block)

```bash
cd ~/mac-migration/inventory

# Installed apps
ls -1 /Applications > applications.txt
ls -1 ~/Applications > applications-user.txt 2>/dev/null
system_profiler SPApplicationsDataType > applications-full.txt

# Homebrew — this is your rebuild manifest
brew bundle dump --file=./Brewfile --describe --force
brew list --formula > brew-formulae.txt
brew list --cask > brew-casks.txt
brew tap > brew-taps.txt

# Language toolchains
pyenv versions          > pyenv-versions.txt 2>/dev/null
pipx list               > pipx-list.txt 2>/dev/null
pip list                > pip-global.txt 2>/dev/null
npm ls -g --depth=0     > npm-global.txt 2>/dev/null
node -v; npm -v         > node-versions.txt 2>/dev/null
ls ~/.nvm/versions/node > nvm-versions.txt 2>/dev/null
sdk current             > sdkman-current.txt 2>/dev/null
/usr/libexec/java_home -V > java-homes.txt 2>&1
go version              > go-version.txt 2>/dev/null
dotnet --list-sdks      > dotnet-sdks.txt 2>/dev/null

# Editors
code --list-extensions  > vscode-extensions.txt 2>/dev/null

# Scheduled jobs
crontab -l              > crontab.txt 2>/dev/null
ls -la ~/Library/LaunchAgents > launchagents.txt 2>/dev/null

# Network / printers
networksetup -listallnetworkservices > network-services.txt
lpstat -p               > printers.txt 2>/dev/null

# Container state (see §2.6)
docker images           > docker-images.txt 2>/dev/null
docker volume ls        > docker-volumes.txt 2>/dev/null
docker context ls       > docker-contexts.txt 2>/dev/null
```

### 1.2 Licences and deauthorisation (do this BEFORE wiping)

Many licences are seat-locked. Deactivate on the old machine or you'll burn a seat.

- [ ] JetBrains (IntelliJ / PyCharm / DataGrip) — `Help → Register → Deactivate`; also enable **Settings Sync** to a JetBrains account
- [ ] Microsoft 365 / Office — sign out of all Office apps
- [ ] Adobe CC — `Help → Sign Out`
- [ ] Setapp / Parallels / VMware Fusion — deactivate seat
- [ ] Sublime Text, Alfred Powerpack, Bartender, iStat Menus, TablePlus, Dash, Little Snitch, Paw/RapidAPI — copy licence keys into `~/mac-migration/inventory/licences.md`
- [ ] Docker Desktop — note whether you're on a paid Business/Team subscription
- [ ] Any hardware-dongle or MAC-locked corporate tooling
- [ ] Screenshot or export licence emails to `~/mac-migration/inventory/`

### 1.3 2FA / authenticator — the one people forget

- [ ] If you use a **desktop** authenticator (Authy desktop, Step Two, 1Password OTP, Raivo) — confirm you can restore it, or move seeds to your phone **first**
- [ ] Confirm you still have recovery codes for: GitLab, GitHub, AWS, Google, brokerage accounts (Zerodha/IBKR/Groww/HDFC), Microsoft
- [ ] Store recovery codes somewhere that is not the machine you're about to erase

---

## 2. Backup — Developer Environment

> Paths below are relative to `$HOME` unless prefixed with `/`.

### 2.1 SSH & GPG — highest priority

| What | Path |
|---|---|
| SSH keys, config, known_hosts | `~/.ssh/` |
| SSH allowed signers (git ssh signing) | `~/.config/git/allowed_signers` |
| GPG keyring | `~/.gnupg/` |

```bash
# Export GPG keys explicitly — safer than copying the keyring directory
gpg --list-secret-keys --keyid-format=long
gpg --export-secret-keys --armor > ~/mac-migration/exports/gpg-secret.asc
gpg --export --armor              > ~/mac-migration/exports/gpg-public.asc
gpg --export-ownertrust           > ~/mac-migration/exports/gpg-ownertrust.txt
```

- [ ] `~/.ssh/config` — check for `IdentityFile`, `ProxyJump`, corporate bastion entries
- [ ] Note which SSH keys are registered on: GitLab, GitHub, jump hosts, OpenShift nodes
- [ ] **Decide now:** reuse existing keys, or generate new ed25519 keys on the new Mac and rotate. Rotating is cleaner but means updating every remote — plan for it.

### 2.2 Cloud, cluster & registry credentials

| What | Path |
|---|---|
| AWS | `~/.aws/` (`credentials`, `config`) |
| Azure | `~/.azure/` |
| GCP | `~/.config/gcloud/` |
| Kubernetes / OpenShift | `~/.kube/config` (skip `~/.kube/cache`, `~/.kube/http-cache`) |
| Extra kubeconfigs | check `$KUBECONFIG` for a colon-separated list |
| Docker registry auth | `~/.docker/config.json` |
| Helm | `~/Library/Preferences/helm/`, `~/Library/Application Support/helm/` |
| Krew plugins | `~/.krew/` (reinstall, don't copy — arch-specific) |
| Netrc | `~/.netrc` |

- [ ] `~/.kube/config` — OpenShift tokens expire; you'll re-`oc login` anyway, but keep cluster/context names
- [ ] Note your `oc` / `kubectl` / `helm` / `argocd` versions so you match them on the new Mac

### 2.3 Git

| What | Path |
|---|---|
| Global config | `~/.gitconfig` |
| XDG config | `~/.config/git/config`, `~/.config/git/ignore` |
| Global gitignore | `~/.gitignore_global` |
| Credential store | `~/.git-credentials` |
| GitLab CLI | `~/.config/glab-cli/config.yml` |
| GitHub CLI | `~/.config/gh/hosts.yml` |
| python-gitlab | `~/.python-gitlab.cfg` |

### 2.4 Shell & dotfiles

| What | Path |
|---|---|
| Zsh | `~/.zshrc`, `~/.zprofile`, `~/.zshenv`, `~/.zlogin`, `~/.zsh_history` |
| Bash | `~/.bashrc`, `~/.bash_profile`, `~/.profile`, `~/.bash_history` |
| Oh-My-Zsh custom | `~/.oh-my-zsh/custom/` (reinstall OMZ itself, keep custom) |
| Powerlevel10k | `~/.p10k.zsh` |
| Starship | `~/.config/starship.toml` |
| Aliases/functions | wherever you sourced them from — grep your `.zshrc` |
| Readline | `~/.inputrc` |
| Vim / Neovim | `~/.vimrc`, `~/.config/nvim/` |
| tmux | `~/.tmux.conf`, `~/.tmux/` |
| Direnv | `~/.config/direnv/` and per-project `.envrc` |

```bash
# Find anything your shell sources from outside $HOME
grep -nE '^\s*(source|\.)\s' ~/.zshrc ~/.zprofile ~/.zshenv 2>/dev/null
```

- [ ] Grep your dotfiles for hardcoded `/usr/local` — every one of these needs to become `/opt/homebrew` on the new Mac

```bash
grep -rn "/usr/local" ~/.zshrc ~/.zprofile ~/.zshenv ~/.bash_profile 2>/dev/null
```

### 2.5 Language toolchains

**Rule: back up the *config*, reinstall the *runtime*.** Compiled x86 artifacts are dead weight on ARM.

| Stack | Back up (config) | Reinstall (do NOT copy) |
|---|---|---|
| Python | `~/.config/pip/pip.conf`, `~/.pypirc`, `~/.condarc`, `~/.streamlit/` | `~/.pyenv/`, `~/miniconda3/`, all `venv/` `.venv/` folders |
| Poetry | `~/Library/Application Support/pypoetry/config.toml`, `auth.toml` | `~/Library/Caches/pypoetry/` |
| pipx | `~/.local/pipx/` — reinstall from `pipx-list.txt` | binaries in `~/.local/bin` |
| Node | `~/.npmrc`, `~/.yarnrc.yml`, `~/.config/configstore/` | `~/.nvm/versions/`, every `node_modules/` |
| Java/Maven | `~/.m2/settings.xml`, `~/.m2/settings-security.xml` | `~/.m2/repository/` (re-downloads) |
| Gradle | `~/.gradle/gradle.properties`, `~/.gradle/init.d/` | `~/.gradle/caches/`, `~/.gradle/wrapper/` |
| SDKMAN | `~/.sdkman/etc/config` | `~/.sdkman/candidates/` |
| .NET | `~/.nuget/NuGet/NuGet.Config` | `~/.nuget/packages/`, `~/.dotnet/` |
| Go | `~/.config/go/env` | `~/go/pkg/`, `~/go/bin/` |
| Ruby | `~/.gemrc`, `~/.bundle/config` | `~/.rbenv/versions/` |
| Rust | `~/.cargo/config.toml`, `~/.cargo/credentials.toml` | `~/.cargo/registry/`, `~/.rustup/` |

> **Your Nexus/Artifactory configs live here.** `~/.m2/settings.xml`, `~/.npmrc`, `~/.config/pip/pip.conf`, and `~/.nuget/NuGet/NuGet.Config` all likely carry internal mirror URLs and credentials. These are annoying to recreate — grab all four.

### 2.6 Containers — the one that bites

| Tool | Path |
|---|---|
| Docker Desktop settings | `~/Library/Group Containers/group.com.docker/settings.json` |
| Docker config/auth | `~/.docker/config.json`, `~/.docker/daemon.json` |
| Buildx state | `~/.docker/buildx/` |
| Colima | `~/.colima/` |
| Rancher Desktop | `~/Library/Application Support/rancher-desktop/` |
| Podman | `~/.config/containers/`, `~/.local/share/containers/` |

- [ ] **Docker volumes are NOT in your home dir.** They live inside the Docker VM disk image. If you have local Postgres/MySQL/Redis data in named volumes, export them:

```bash
# Per volume — repeat for each
docker run --rm -v <volume-name>:/data -v ~/mac-migration/exports:/backup \
  alpine tar czf /backup/<volume-name>.tar.gz -C /data .
```

- [ ] Dump any local databases properly:

```bash
pg_dumpall -U postgres > ~/mac-migration/exports/postgres-all.sql
mysqldump --all-databases -u root -p > ~/mac-migration/exports/mysql-all.sql
```

- [ ] List any **locally built** images that aren't in a registry — those are gone unless you `docker save` them (and they'll be `amd64`, so likely worth rebuilding anyway)
- [ ] Note Docker Desktop resource settings (CPU/RAM/disk) so you can match on the new Mac

### 2.7 IaC & automation

| What | Path |
|---|---|
| Terraform | `~/.terraformrc`, `~/.terraform.d/` (credentials, plugin cache) |
| Terraform state | **check for local `terraform.tfstate` files not in remote backend** |
| Ansible | `~/.ansible.cfg`, `~/.ansible/`, vault password files |
| Vagrant | `~/.vagrant.d/` (Intel boxes won't work on ARM) |
| Packer | `~/.packer.d/` |
| Vault | `~/.vault-token` |

### 2.8 Editors & IDEs

**VS Code**

| What | Path |
|---|---|
| Settings | `~/Library/Application Support/Code/User/settings.json` |
| Keybindings | `~/Library/Application Support/Code/User/keybindings.json` |
| Snippets | `~/Library/Application Support/Code/User/snippets/` |
| Workspace storage | `~/Library/Application Support/Code/User/workspaceStorage/` |
| Extensions | `~/.vscode/extensions/` — **reinstall from list, don't copy** (native modules) |

Restore extensions on the new Mac:
```bash
xargs -n1 code --install-extension < vscode-extensions.txt
```

**JetBrains** — easiest path is `Settings → Settings Sync → Enable`. Manual paths:
- `~/Library/Application Support/JetBrains/<Product><Version>/`
- `~/Library/Preferences/<Product><Version>/`

**Obsidian** — two parts, both needed:
- Vault folder itself (wherever you keep it — likely under `~/Documents/` or inside OneDrive)
- App config: `~/Library/Application Support/obsidian/`
- Per-vault config lives inside the vault at `<vault>/.obsidian/` — **make sure your archive doesn't skip dotfolders**

**Others**
- Sublime Text: `~/Library/Application Support/Sublime Text/Packages/User/`
- Zed: `~/.config/zed/`
- Cursor: `~/Library/Application Support/Cursor/User/`

### 2.9 Terminal

| What | Path |
|---|---|
| iTerm2 prefs | `~/Library/Preferences/com.googlecode.iterm2.plist` |
| iTerm2 dynamic profiles | `~/Library/Application Support/iTerm2/DynamicProfiles/` |
| Terminal.app profiles | `~/Library/Preferences/com.apple.Terminal.plist` (or export `.terminal` files) |
| Warp | `~/.warp/` |
| Fonts (Nerd Fonts etc.) | `~/Library/Fonts/` |

```bash
# Force iTerm2 to flush prefs to disk before copying
defaults read com.googlecode.iterm2 > /dev/null
killall cfprefsd
```

---

## 3. Backup — Personal Data

| What | Path |
|---|---|
| Documents | `~/Documents/` |
| Desktop | `~/Desktop/` |
| Downloads | `~/Downloads/` (triage first — usually 80% deletable) |
| Pictures | `~/Pictures/` |
| Photos library | `~/Pictures/Photos Library.photoslibrary` (large; treat as one bundle) |
| Movies | `~/Movies/` |
| Music library | `~/Music/Music/Media/` |
| Public | `~/Public/` |
| Code repos | wherever you keep them — `~/projects`, `~/work`, `~/dev` |
| Existing OneDrive folder | `~/Library/CloudStorage/OneDrive-<Tenant>/` (macOS 12.3+) or `~/OneDrive/` (older) |

- [ ] Check for data outside `$HOME`: `ls -la /Volumes/`, `/opt/`, `/srv/`, `/usr/local/var/`
- [ ] Check `/usr/local/var/` for Homebrew-installed Postgres/MySQL data dirs (Intel Homebrew put them here)

---

## 4. Backup — App & System Settings

| What | Path |
|---|---|
| App preferences (selective) | `~/Library/Preferences/` |
| App support data (selective) | `~/Library/Application Support/` |
| Sandboxed app data | `~/Library/Containers/<bundle-id>/Data/` |
| Fonts | `~/Library/Fonts/` |
| Login items / agents | `~/Library/LaunchAgents/` |
| Services / Automator | `~/Library/Services/` |
| Shortcuts (Automator quick actions) | `~/Library/Services/` |
| Screen Saver / wallpaper | `~/Library/Application Support/com.apple.wallpaper/` |
| Dictionaries / text replacements | `~/Library/Dictionaries/`, `~/Library/KeyboardServices/` |

**Specific apps worth checking:**
- Alfred: `~/Library/Application Support/Alfred/` (+ Alfred's own sync folder setting)
- Raycast: export via `Raycast Settings → Advanced → Export`
- Rectangle: `~/Library/Preferences/com.knollsoft.Rectangle.plist`
- Karabiner-Elements: `~/.config/karabiner/`
- Hammerspoon: `~/.hammerspoon/`
- CleanShot / Shottr: preference plists
- Tunnelblick VPN: `~/Library/Application Support/Tunnelblick/`
- Little Snitch rules: export via app

**Browsers**

| Browser | Path |
|---|---|
| Chrome profile | `~/Library/Application Support/Google/Chrome/Default/` |
| Chrome bookmarks only | `~/Library/Application Support/Google/Chrome/Default/Bookmarks` |
| Edge | `~/Library/Application Support/Microsoft Edge/Default/` |
| Firefox | `~/Library/Application Support/Firefox/Profiles/` |
| Brave | `~/Library/Application Support/BraveSoftware/Brave-Browser/Default/` |
| Safari (no iCloud → local only!) | `~/Library/Safari/Bookmarks.plist`, `~/Library/Containers/com.apple.Safari/` |

> Since you're not using iCloud, **Safari bookmarks/history will not sync**. Export bookmarks to HTML manually: `File → Export → Bookmarks`.

**Apple apps holding local-only data (no iCloud = no sync)**

| App | Path |
|---|---|
| Notes | `~/Library/Group Containers/group.com.apple.notes/` |
| Mail | `~/Library/Mail/`, `~/Library/Containers/com.apple.mail/` |
| Messages | `~/Library/Messages/` |
| Stickies | `~/Library/Containers/com.apple.Stickies/` |
| Contacts | `~/Library/Application Support/AddressBook/` |
| Calendars | `~/Library/Calendars/` |
| Reminders | `~/Library/Group Containers/group.com.apple.reminders/` |
| Preview signatures / annotations | `~/Library/Containers/com.apple.Preview/` |

- [ ] Export Notes to PDF/text if you have anything important there — the container format is fragile across versions
- [ ] Export Contacts: `Contacts → File → Export → Contacts Archive`
- [ ] Export Calendars: `Calendar → File → Export → Calendar Archive`

**System-level files**

| What | Path |
|---|---|
| Hosts file | `/etc/hosts` |
| SSH client system config | `/etc/ssh/ssh_config` |
| Sudoers customisations | `/etc/sudoers.d/` |
| System-wide profile | `/etc/paths`, `/etc/paths.d/` |
| Custom CA certs | System keychain (see §5) |

```bash
sudo cp /etc/hosts /etc/paths ~/mac-migration/configs/
sudo cp -R /etc/paths.d ~/mac-migration/configs/ 2>/dev/null
```

---

## 5. Keychain & Passwords — Critical Without iCloud

**No iCloud Keychain means nothing carries over automatically.** Handle this deliberately.

| What | Path |
|---|---|
| Login keychain | `~/Library/Keychains/login.keychain-db` |
| Local items keychain | `~/Library/Keychains/<UUID>/` |
| System keychain (certs, Wi-Fi) | `/Library/Keychains/System.keychain` |

**Preferred approach — move to a password manager now:**
- [ ] Install 1Password / Bitwarden / KeePassXC if you aren't already on one
- [ ] Export Chrome/Safari saved passwords to CSV, import into the manager, **then delete the CSV securely** (`rm -P file.csv`)
- [ ] Manually record Wi-Fi passwords (corporate 802.1X won't transfer)

**Fallback — copy the keychain file:**
```bash
cp ~/Library/Keychains/login.keychain-db ~/mac-migration/exports/
```
On the new Mac: `Keychain Access → File → Add Keychain…` and unlock with your **old Mac's login password**. Works, but the file is a plaintext-adjacent secret store — encrypt it before it goes near OneDrive.

**Certificates you'll need to re-import:**
- [ ] Corporate root/intermediate CA certs (for Artifactory/Nexus/GitLab over internal TLS)
- [ ] Client certs for VPN or internal services
- [ ] Export from Keychain Access → select cert → `File → Export Items` → `.p12`

---

## 6. Uncommitted Code Sweep — Do This Last, Right Before Wiping

This is the most common real loss during a Mac migration.

```bash
# Find every repo with dirty state, stashes, or unpushed commits
find ~ -type d -name .git -maxdepth 6 -prune 2>/dev/null | while read -r g; do
  r="${g%/.git}"
  dirty=$(git -C "$r" status --porcelain 2>/dev/null | wc -l | tr -d ' ')
  stash=$(git -C "$r" stash list 2>/dev/null | wc -l | tr -d ' ')
  ahead=$(git -C "$r" log --branches --not --remotes --oneline 2>/dev/null | wc -l | tr -d ' ')
  if [ "$dirty" != "0" ] || [ "$stash" != "0" ] || [ "$ahead" != "0" ]; then
    printf '%-60s dirty:%-5s stash:%-5s unpushed:%s\n' "$r" "$dirty" "$stash" "$ahead"
  fi
done | tee ~/mac-migration/inventory/dirty-repos.txt
```

- [ ] Push every branch that matters, or bundle the repo: `git bundle create repo.bundle --all`
- [ ] **Git stashes do not push.** Either apply+commit them or bundle
- [ ] **`.env` files are gitignored** — they exist only on this disk. Find them:

```bash
find ~ -name "*.env" -o -name ".env" -o -name ".env.*" 2>/dev/null | grep -v node_modules
```

- [ ] Same for `*.pem`, `*.key`, `*.p12`, `kubeconfig-*`, `secrets.yaml` — grab anything not in version control
- [ ] Scratch scripts and one-off SQL in `~/Desktop`, `~/tmp`, `/tmp`
- [ ] Your local SQLite portfolio database and any Streamlit app data files

---

## 7. Packaging for OneDrive — Gotchas That Will Bite You

OneDrive is a document sync tool, not a filesystem backup. It **does not preserve**:
- POSIX permissions (your `~/.ssh` 600 bits → gone)
- Symlinks (silently converted or skipped)
- Extended attributes and macOS resource forks
- Executable bits on scripts
- Hard links, sparse files, `.app` bundle integrity

**Additional OneDrive restrictions:**
- Invalid filename characters: `\ / : * ? " < > |`
- Reserved names: `CON`, `PRN`, `AUX`, `NUL`, `COM0-9`, `LPT0-9`
- No leading/trailing spaces, no names ending in `.`
- Full path length limit ~400 characters → **`node_modules` will break this**
- Individual file limit 250 GB
- Files On-Demand: files may be online-only placeholders. If you `tar` a folder inside OneDrive it will either stall on hydration or archive empty stubs.

### 7.1 The correct approach: archive first, then upload

```bash
cd ~
mkdir -p ~/mac-migration/exports

# 1. Dotfiles + dev configs, permissions preserved (-p)
tar -czpf ~/mac-migration/exports/dotfiles-$(date +%F).tar.gz \
  --exclude='.m2/repository' \
  --exclude='.gradle/caches' \
  --exclude='.docker/buildx' \
  .ssh .gnupg .aws .azure .kube/config .docker/config.json .config \
  .zshrc .zprofile .zshenv .zsh_history .bash_profile .inputrc \
  .gitconfig .gitignore_global .git-credentials .netrc \
  .m2/settings.xml .m2/settings-security.xml .npmrc .yarnrc.yml .pypirc \
  .nuget .gemrc .cargo/config.toml .terraformrc .terraform.d \
  .ansible.cfg .python-gitlab.cfg .streamlit .p10k.zsh .vimrc .tmux.conf \
  2>/dev/null

# 2. Verify what actually made it in
tar -tzf ~/mac-migration/exports/dotfiles-*.tar.gz | head -50
```

### 7.2 Encrypt before it touches OneDrive

This archive contains SSH private keys and cloud credentials. Encrypt it.

```bash
# Option A — GPG symmetric
gpg -c --cipher-algo AES256 ~/mac-migration/exports/dotfiles-$(date +%F).tar.gz
rm ~/mac-migration/exports/dotfiles-$(date +%F).tar.gz

# Option B — encrypted disk image for the whole migration folder
hdiutil create -encryption AES-256 -stdinpass -srcfolder ~/mac-migration \
  -format UDZO ~/Desktop/mac-migration-$(date +%F).dmg
```

- [ ] Store the passphrase in your password manager **and** somewhere off-machine
- [ ] Confirm with IT whether corporate credentials may live in the corporate OneDrive tenant at all

### 7.3 What NOT to sync to OneDrive

- `node_modules/`, `.venv/`, `venv/`, `__pycache__/`, `.tox/`
- `.git/` folders (push to GitLab instead — that *is* your backup)
- `~/.m2/repository/`, `~/.gradle/caches/`, `~/.nuget/packages/`, `~/.cargo/registry/`
- `~/Library/Caches/`
- `.app` bundles / `/Applications`
- Docker VM disk images
- Anything in an active build directory

### 7.4 Verify the upload actually completed

- [ ] OneDrive menu bar icon shows **"Up to date"**, not "Syncing" or "Processing changes"
- [ ] Check for sync errors: OneDrive menu → any red badge → resolve every one
- [ ] **Log into OneDrive in a browser and confirm the archive is visible and downloadable there** — not just present locally
- [ ] Note the file size in the browser matches local
- [ ] Right-click critical folders → **"Always keep on this device"** so you're not backing up placeholders

### 7.5 Also do a Time Machine backup

```bash
# After connecting external SSD and configuring in System Settings
tmutil startbackup --block
```
Time Machine preserves everything OneDrive can't. Keep the SSD until the new Mac has been running clean for a month.

---

## 8. Intel → Apple Silicon: What Breaks

| Issue | What to do |
|---|---|
| **Homebrew prefix change** | Intel `/usr/local` → ARM `/opt/homebrew`. Update every PATH reference in your dotfiles |
| **Rosetta 2 required** for x86 apps | `softwareupdate --install-rosetta --agree-to-license` |
| **VirtualBox** | No usable ARM support. Migrate to UTM, Parallels, or Lima/Colima |
| **Intel VMs / Vagrant boxes** | Won't boot. Find `arm64` box equivalents or use remote runners |
| **Docker `amd64` images** | Run under QEMU emulation — functional but slow. For your GitLab CI work: build multi-arch with `docker buildx build --platform linux/amd64,linux/arm64`, or push builds to a Linux runner |
| **Docker: `exec format error`** | Missing `--platform linux/amd64` on an amd64-only image |
| **Python wheels** | Some older packages have no `arm64` wheel and need compilation. Install Xcode CLT first |
| **Node native modules** | `node-gyp`, `sharp`, `bcrypt`, `canvas` → delete `node_modules` and reinstall, never copy |
| **Java** | Install an `arm64` JDK (Temurin/Zulu/Corretto). An x86 JDK will run under Rosetta and be noticeably slower |
| **Homebrew formulae without ARM bottles** | Rare now, but a few build from source (slow). If truly Intel-only: `arch -x86_64 /usr/local/bin/brew install <pkg>` under a second Rosetta Homebrew |
| **Old 32-bit apps** | Already dead since Catalina, but double-check anything ancient in `/Applications` |
| **Kernel extensions (kexts)** | Legacy VPN clients, Little Snitch <5, older backup tools. Check vendor support before assuming |
| **Wireshark / packet capture tools** | Need updated ARM builds |

> **For your GitLab CI work specifically:** if you build images locally that ship to OpenShift on `amd64`, plan for `buildx` + a remote builder or CI-side builds. Local QEMU emulation is fine for smoke tests, painful for anything with a compile step.

---

## 9. New Mac Setup — Order Matters

Do these in sequence. Skipping ahead causes rework.

1. [ ] **Setup Assistant** — create your local account with the **same short username** as the old Mac (avoids path breakage in configs). Check with `whoami` on the old Mac first.
2. [ ] **Skip iCloud entirely.** If you sign in to an Apple Account for App Store only, immediately turn off: iCloud Drive, Desktop & Documents Folders, Photos, Keychain, Mail, Contacts, Calendars, Find My (keep Find My on if you want, but nothing else).
3. [ ] **System Settings** — turn on FileVault. Save the recovery key to your password manager.
4. [ ] **Rosetta 2**: `softwareupdate --install-rosetta --agree-to-license`
5. [ ] **Xcode Command Line Tools**: `xcode-select --install`
6. [ ] **Homebrew** (installs to `/opt/homebrew` automatically on ARM):
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
   eval "$(/opt/homebrew/bin/brew shellenv)"
   ```
7. [ ] **Install OneDrive**, sign in, sync down your migration folder
8. [ ] **Restore dotfiles**:
   ```bash
   gpg -d dotfiles-YYYY-MM-DD.tar.gz.gpg | tar -xzpf - -C ~
   ```
9. [ ] **Fix permissions immediately** (tar `-p` helps, but verify):
   ```bash
   chmod 700 ~/.ssh ~/.gnupg
   chmod 600 ~/.ssh/id_* ~/.ssh/config
   chmod 644 ~/.ssh/*.pub ~/.ssh/known_hosts
   chmod 600 ~/.aws/credentials ~/.netrc ~/.python-gitlab.cfg 2>/dev/null
   chmod 600 ~/.m2/settings.xml ~/.m2/settings-security.xml 2>/dev/null
   ```
10. [ ] **Update PATH references** — replace `/usr/local` with `/opt/homebrew` in `~/.zshrc` / `~/.zprofile`
11. [ ] **Restore Homebrew packages**: `brew bundle install --file=./Brewfile`
    - Review the Brewfile first — drop Intel-era cruft you no longer need. This is a good chance to clean house.
12. [ ] **Language runtimes** — pyenv/nvm/sdkman installs, then rebuild versions from your inventory files
13. [ ] **GPG import**:
    ```bash
    gpg --import gpg-secret.asc
    gpg --import-ownertrust gpg-ownertrust.txt
    ```
14. [ ] **Keychain / password manager** restore
15. [ ] **Certificates** — import corporate CA certs into System keychain, set to "Always Trust"
16. [ ] **VS Code** — restore settings + `xargs -n1 code --install-extension < vscode-extensions.txt`
17. [ ] **JetBrains** — Settings Sync, or restore config folders
18. [ ] **Docker Desktop / Colima** — install, set resources, restore `~/.docker/config.json`, `docker login` to Artifactory
19. [ ] **VPN client** + profiles
20. [ ] **Restore Docker volumes** from the tarballs in §2.6
21. [ ] **Clone your repos fresh** from GitLab — don't copy working trees
22. [ ] **Touch ID for sudo**:
    ```bash
    sudo cp /etc/pam.d/sudo_local.template /etc/pam.d/sudo_local
    sudo sed -i '' 's/^#auth/auth/' /etc/pam.d/sudo_local
    ```

---

## 10. Verification — Prove It Works Before Wiping the Old Mac

Run each of these on the new Mac. Don't erase the 2019 until all pass.

```bash
# Architecture sanity
uname -m                          # → arm64
which brew                        # → /opt/homebrew/bin/brew
brew config | grep -i rosetta     # → Rosetta 2: false

# SSH / Git
ssh -T git@gitlab.<your-domain>
git clone <a-real-repo> /tmp/test-clone && rm -rf /tmp/test-clone
git commit --allow-empty -m "signing test" -S   # if you sign commits

# GPG
gpg --list-secret-keys

# Cloud / cluster
aws sts get-caller-identity
oc login <cluster>  &&  oc get projects
kubectl config get-contexts && kubectl get nodes
helm version

# Registries
docker login <artifactory-host>
docker pull <a-known-internal-image>
docker run --rm --platform linux/amd64 alpine uname -m   # → x86_64 via Rosetta

# Build tooling with internal mirrors
mvn -s ~/.m2/settings.xml help:effective-settings | head
pip config list
npm config get registry

# Python
python3 -c "import platform; print(platform.machine())"   # → arm64
```

**Manual checks:**
- [ ] VPN connects and internal DNS resolves
- [ ] Artifactory + Nexus web UIs reachable and authenticated
- [ ] GitLab CI pipeline triggerable from your local `glab` / `python-gitlab` scripts
- [ ] Obsidian vault opens with all plugins and your `Haresh_Complete_Plan.md` intact
- [ ] Streamlit portfolio tracker runs and reads the SQLite DB
- [ ] Printers, external display, dock/hub, Bluetooth peripherals
- [ ] Backup: configure Time Machine on the new Mac immediately

---

## 11. Decommission the Old Mac

**Only after §10 fully passes, and ideally leave 2–4 weeks of overlap.**

- [ ] Final Time Machine backup, then disconnect and label the SSD
- [ ] Sign out of every app (Office, Adobe, JetBrains, Slack, browsers)
- [ ] Sign out of Apple Account: `System Settings → [Your Name] → Sign Out`
- [ ] Turn off Find My Mac
- [ ] Unpair Bluetooth devices
- [ ] Sign out and **unlink** OneDrive: `OneDrive → Settings → Account → Unlink this Mac`
- [ ] **Erase All Content and Settings**: `System Settings → General → Transfer or Reset → Erase All Content and Settings`
  - (2019 MBP has a T2 chip, so this works — no need for a full reinstall)
- [ ] If company-owned: hand back per IT process, confirm MDM record removal
- [ ] If selling: reinstall macOS after erase and leave at Setup Assistant

---

## Quick Priority Ranking

If you only have time for a subset, do these in order:

1. `~/.ssh/` + GPG keys export
2. Uncommitted/unpushed code + `.env` files (§6)
3. `~/.m2/settings.xml`, `~/.npmrc`, `~/.config/pip/pip.conf`, `~/.nuget/NuGet/NuGet.Config` — your Nexus/Artifactory configs
4. `~/.aws/`, `~/.kube/config`, `~/.docker/config.json`
5. `~/.zshrc` + `~/.gitconfig`
6. `brew bundle dump` Brewfile
7. Obsidian vault (including `.obsidian/`)
8. Local databases + Docker volumes
9. Password manager migration + 2FA recovery codes
10. `~/Documents`, `~/Desktop`, `~/Pictures`
