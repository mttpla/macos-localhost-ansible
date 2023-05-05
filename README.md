# Ansible Playbook

## Setup the host machine

### Ansible installation on MacOs localhost

1. Run xcode-select --install
2. SSH keys: save the public and private ssh key in `~/.ssh`
3. Install [Homebrew](https://brew.sh). `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)" `
5. Install Python (`brew install python`)
6. Install Ansible (`brew install ansible`)
7. [Optional] Install visual-studio-code-extentions (`ansible-galaxy install gantsign.visual-studio-code-extensions`)

## Playbook: MacOs localhost

1. Edit `macos_config.yml`
2. Edit the file `ansible_secrets.yml` following the `ansible_secrets_example.yml` file
3. Run `ansible-playbook macos.yml`

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
