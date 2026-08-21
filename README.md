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

The `Brewfile` is the source of truth — nothing is declared in Ansible. Steps,
in order:

1. **Find the exact name.** `brew search <term>`, then `brew info <name>` to
   confirm it is what you want and to read its one-line description.
2. **Add one line to the `Brewfile`**, in the right block and in alphabetical
   order inside that block. `brew bundle dump` writes the blocks in this order —
   `tap`, `brew`, `cask`, `mas`, `vscode` — so matching it keeps future diffs
   small:

   ```ruby
   # Search tool like grep and The Silver Searcher
   brew "ripgrep"                       # CLI formula
   cask "obsidian"                      # GUI app
   mas  "Keynote", id: 361285480        # Mac App Store app
   vscode "ms-python.python"            # VS Code extension
   ```

   The comment above each line is the formula's `desc` — `brew bundle dump`
   generates it, so write it by hand to keep a later `dump` diff clean.
3. **From a third-party tap?** Add the `tap "owner/repo"` line too, and run
   `brew trust owner/repo` once on the machine. Without the trust, `brew bundle`
   refuses to run at all — no Brewfile flag can grant it.
4. **Apply it:** `ansible-playbook macos.yml`, or `brew bundle install
   --file=Brewfile --no-upgrade` for just the packages.
5. **Verify:** `brew list --versions <name>` and `which <name>`.
6. Commit the `Brewfile` change.

Removing a package is *not* the reverse: deleting the line uninstalls nothing.
See "Keeping the Brewfile honest" below.

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
