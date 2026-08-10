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

## Keeping the Brewfile honest

```bash
brew bundle check      # something missing or outdated?
brew bundle cleanup    # dry run: installed but NOT in the Brewfile
brew bundle dump --force   # regenerate from this machine, then review the diff
```

`brew bundle install` never uninstalls, and `dump` silently skips untrusted
taps. See `AGENTS.md` for the details worth knowing before trusting either.

## Acknowledgements

This repo is a fork form [ansible-macos-playbook](https://github.com/jeromegamez/ansible-macos-playbook) by [Jérôme Gamez](https://github.com/jeromegamez). Thank you!
