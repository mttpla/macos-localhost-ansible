# Ansible Playbook

## Macos

This is the playbook I use after a clean install of MacOS to set everything up.

### Roles/Tasks

- Installs Homebrew packages and app casks (Role `homebrew`)
- Modifies system settings (ssh key, zsh prompt) (Role `system_settings`)
- etc etc

### Installation

1. Install [Homebrew](https://brew.sh).
2. Install Python (`brew install python`)
3. Install Ansible (`brew install ansible`)
4. Install visual-studio-code-extentions (`ansible-galaxy install gantsign.visual-studio-code-extensions`)

### Secrets

1. SSH keys: save the public and private ssh key in `~/.ssh`,
2. vars secrets: edit the file `ansible_secrets.yml` following the `ansible_secrets_example.yml` file

## MacOs locahost

1. Edit `macos_config.yml`
2. Run `ansible-playbook macos.yml`

## Debian on AWS ECS

Run the instance in AWS and get the new public ip. Update the inventory, file `hosts`

Run the ansible playbook:

1. Edit `debian_config.yml`
2. Run `ansible-playbook debian.yml`

Connect using ssh:
`ssh -i ./mttRemoteDesktop.pem root@<EC2-address>.compute.amazonaws.com`

### Acknowledgements

This repo is a fork form [ansible-macos-playbook](https://github.com/jeromegamez/ansible-macos-playbook) by [Jérôme Gamez](https://github.com/jeromegamez). Thank you!
