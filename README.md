# Ansible MacOS Playbook

This is the playbook I use after a clean install of MacOS to set everything up.

## Roles/Tasks

- Installs Homebrew packages and app casks (Role `homebrew`)
- Modifies system settings (ssh key, zsh prompt) (Role `system_settings`)
- etc etc

## Installation

1. Install [Homebrew](https://brew.sh).
2. Install Python (`brew install python`)
3. Install Ansible (`brew install ansible`)
4. Install visual-studio-code-extentions (`ansible-galaxy install gantsign.visual-studio-code-extensions`)
5. Edit `config.yml` 
6. Run `ansible-playbook main.yml`

## Secrets

1. SSH keys: save the public and private ssh key in `~/.ssh`,

## Acknowledgements

This repo is a fork form [ansible-macos-playbook](https://github.com/jeromegamez/ansible-macos-playbook) by [Jérôme Gamez](https://github.com/jeromegamez). Thank you!


