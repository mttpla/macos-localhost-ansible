# Ansible Playbook

## Setup the host machine

### Ansible installation on MacOs localhost

1. Run xcode-select --install
2. SSH keys: save the public and private ssh key in `~/.ssh` (if needed add the key at ssh-agent)
3. Install [Homebrew](https://brew.sh). `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)" `
4. Update the .zprofile. Run the command in "Next Steps". Add the `eval "$(/opt/homebrew/bin/brew shellenv)")`
5. Install Python (`brew install python3`)
6. Install Ansible (`brew install ansible`) 
7. [Optional] Install visual-studio-code-extentions (`ansible-galaxy install gantsign.visual-studio-code-extensions`)
8. run git clone

## Playbook: MacOs localhost

1. Edit `Brewfile` for packages and applications, `macos_config.yml` for settings
2. Trust the third-party taps once per machine, otherwise `brew bundle` refuses to run:
   `brew trust anomalyco/tap esolitos/ipa messense/macos-cross-toolchains mongodb/brew sdkman/tap`
3. Run `ansible-playbook macos.yml --ask-become-pass`

`ansible_secrets.yml` is only used by the Debian playbook.

To see what is installed but not tracked in the `Brewfile`, run `brew bundle cleanup`.

## Playbook: Debian on AWS ECS

Run the instance in AWS and get the new public ip. Update the inventory, file `hosts` with the public IP.

Run the ansible playbook:

1. Edit `debian_config.yml`
2. Edit the file `ansible_secrets.yml` following the `ansible_secrets_example.yml` file
3. Run `ansible-playbook debian.yml`

### Connect to the EC2

Connect using ssh:
`ssh -i ./mttRemoteDesktop.pem root@<EC2-address>.compute.amazonaws.com`

### Note

#### how to get the dynamic aws ec2 ip

`aws ec2 describe-instances --instance-ids $instance_id --query 'Reservations[*].Instances[*].PublicIpAddress' --output text`


## Acknowledgements

This repo is a fork form [ansible-macos-playbook](https://github.com/jeromegamez/ansible-macos-playbook) by [Jérôme Gamez](https://github.com/jeromegamez). Thank you!
