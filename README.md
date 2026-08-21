# Ansible Playbook

Provisions a macOS workstation: Homebrew packages from the `Brewfile`, plus
shell, SSH, Finder and Dock settings from `macos_config.yml`.

## Setup the host machine

1. Run xcode-select --install
2. SSH keys: save the public and private ssh key in `~/.ssh` (if needed add the key at ssh-agent)
3. Install [Homebrew](https://brew.sh). `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)" `
4. Update the .zprofile. Run the command in "Next Steps". Add the `eval "$(/opt/homebrew/bin/brew shellenv)")`
5. Install Python (`brew install python3`)
6. Install Ansible (`brew install ansible`)
7. Sign in to the App Store, otherwise the `mas` entries in the `Brewfile` cannot install
8. run git clone

## Run it

1. Edit `Brewfile` for packages and applications, `macos_config.yml` for settings
2. Trust the third-party taps once per machine, otherwise `brew bundle` refuses to run:
   `brew trust anomalyco/tap esolitos/ipa messense/macos-cross-toolchains mongodb/brew sdkman/tap`
3. Run `ansible-playbook macos.yml`

`--ask-become-pass` is only needed if you set `computername`/`hostname`, which
are null by default.

## Adding a new package

The `Brewfile` is the source of truth — nothing is declared in Ansible. Adding a
package means **installing it and editing the `Brewfile` by hand**; there is no
`brew bundle add` command. (`brew bundle dump --force` regenerates the whole
file and is the wrong tool here — see the warnings in the next section.)

Run every command from the repo root. The example adds `ripgrep`; replace the
value of `PKG` and paste the rest as-is.

### 1. Find the exact name and check what it is

```bash
PKG=ripgrep

brew search "$PKG"     # candidates, if you are unsure of the spelling
brew info "$PKG"       # description, version, dependencies, homepage
```

`brew info` failing means the name is wrong or the formula lives in a tap you
have not added yet — see step 5.

### 2. Install it

```bash
brew install "$PKG"                 # CLI formula
# brew install --cask "$PKG"        # GUI app instead
```

### 3. Add the line to the Brewfile

Each entry is two lines: a comment holding the package's one-line description,
then the entry itself. `brew bundle dump` writes exactly that shape, so keeping
it means a future `dump` diff stays clean.

Print the two lines to paste:

```bash
printf '# %s\nbrew "%s"\n' "$(brew desc "$PKG" | sed 's/^[^:]*: //')" "$PKG"
```

Then open the file and paste them into the `brew` block, in alphabetical order:

```bash
${EDITOR:-open -e} Brewfile
```

Block order in the file is `tap`, `brew`, `cask`, `mas`, `vscode`, then the
hand-kept `cargo`/`npm` notes — the same order `dump` uses. Tap-qualified
formulae (`brew "owner/repo/name"`) sit in their own group after the plain ones.
Position is cosmetic: `brew bundle` reads the file in any order, but a
misplaced line turns every future `dump` into a noisy diff.

**Or skip the editor.** This inserts the entry in the right place on its own,
then shows you the result:

```bash
awk -v n="$PKG" -v d="$(brew desc "$PKG" | sed 's/^[^:]*: //')" '
  !ins && /^#/             { c = c $0 "\n"; next }
  !ins && /^(brew|cask) "/ { split($0, a, "\"")
                             if ($1 == "cask" || a[2] ~ /\// || (a[2] "") > (n "")) {
                               printf "# %s\nbrew \"%s\"\n", d, n; ins = 1 }
                             printf "%s", c; c = ""; print; next }
                           { printf "%s", c; c = ""; print }
' Brewfile > Brewfile.new && mv Brewfile.new Brewfile

git diff Brewfile
```

For a cask, change `brew "%s"` to `cask "%s"` and strip the parenthesised app
name that `brew desc --cask` prepends:
`brew desc --cask "$PKG" | sed 's/^[^:]*: //; s/^([^)]*) //'`.

`mas` and `vscode` entries have no such helper — `mas list` and
`code --list-extensions` give you the ids to paste.

### 4. Verify and commit

```bash
brew list --versions "$PKG"          # installed?
command -v "$PKG"                    # on PATH? (binary name may differ: ripgrep → rg)
git diff Brewfile                    # exactly one entry added, nothing else moved
git add Brewfile && git commit -m "Add $PKG to the Brewfile"
```

Nothing else needs to run on this machine — you already installed it in step 2.
On a *different* machine the entry gets picked up by `ansible-playbook macos.yml`
(or `brew bundle install --file=Brewfile --no-upgrade` for packages only).

### 5. Only if the package comes from a third-party tap

```bash
brew tap owner/repo                  # add the tap
brew trust owner/repo                # required, once per machine
```

Add a matching `tap "owner/repo"` line at the top of the `Brewfile`, and write
the formula as `brew "owner/repo/name"`. Without the `brew trust`, `brew bundle`
refuses to run at all — no `Brewfile` flag can grant it, which is why the taps
are listed in "Run it" above.

Removing a package is *not* the reverse: deleting the line uninstalls nothing.
See the next section.

## Keeping the Brewfile honest

The `Brewfile` and the machine drift in both directions. Three commands, three
different questions:

```bash
brew bundle check --file=Brewfile      # is anything in the Brewfile missing here?
brew bundle cleanup --file=Brewfile    # dry run: what is installed but NOT listed?
brew bundle dump --force               # regenerate from this machine, then diff
```

Four things that bite:

- **`install` never uninstalls.** Deleting a line is a no-op. Real removal is
  `brew bundle cleanup --force` — destructive, and deliberately not wired into
  the playbook. Prefer an explicit `brew uninstall <name>`.
- **`check` conflates "missing" with "outdated".** It reports `needs to be
  installed or updated` for packages that are present but stale, so a non-zero
  exit is not proof anything is absent.
- **`dump --force` overwrites the file.** Always `git diff` before committing.
  It re-adds the ten casks this machine cannot uninstall (see `AGENTS.md`), and
  it drops the hand-written comments for untrusted taps.
- **`dump` silently skips untrusted taps.** `sshpass`, `sdkman-cli`,
  `mongodb-database-tools` and the arm64 cross toolchain are listed by hand and
  disappear on every regeneration. Re-add them after each `dump`.

The `cargo` and `npm` lines at the bottom are likewise not managed by
`brew bundle` — they are notes, kept by hand.

## Keeping Homebrew up to date

**Is `brew update && brew upgrade` enough?** Almost — it covers formulae and
most casks, but it silently skips self-updating casks and does not touch the
App Store at all. The full routine:

```bash
brew update                # 1. refresh formula/cask definitions (no packages change)
brew outdated              # 2. see what would change, before it changes
brew upgrade               # 3. upgrade formulae + casks
brew upgrade --cask --greedy   # 4. casks that auto-update, pinned by brew otherwise
mas upgrade                # 5. Mac App Store apps
brew cleanup               # 6. delete old versions and stale downloads
brew doctor                # 7. only when something misbehaves
```

Why steps 4 and 5 exist:

- `brew upgrade` skips casks marked `auto_updates true` or `version :latest`,
  on the assumption the app updates itself. `--greedy` forces them. Compare
  `brew outdated --cask` with `brew outdated --cask --greedy` to see the gap —
  on this machine it is roughly a dozen extra apps (Chrome, Docker Desktop,
  IntelliJ, …). Run it occasionally rather than never, but expect it to churn
  apps that were already keeping themselves current.
- `mas` apps are a separate world; `brew upgrade` never touches them.

Two notes specific to this repo:

- **The playbook is not an updater.** `macos/homebrew` runs `brew bundle
  install --no-upgrade`, so `ansible-playbook macos.yml` installs what is
  missing and upgrades nothing. Setting `homebrew_upgrade_all: true` in
  `roles/macos/homebrew/defaults/main.yml` adds a `brew upgrade` before the
  bundle; it is `false` on purpose — a bootstrap should not double as a package
  updater.
- **`brew update` runs on its own** before most commands (auto-update). Set
  `HOMEBREW_NO_AUTO_UPDATE=1` to suppress it when you want a fast, predictable
  install.

## Acknowledgements

This repo is a fork form [ansible-macos-playbook](https://github.com/jeromegamez/ansible-macos-playbook) by [Jérôme Gamez](https://github.com/jeromegamez). Thank you!
