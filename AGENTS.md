# AGENTS.md

This file provides guidance to coding agents when working with code in this repository.

## What this is

Ansible playbooks for provisioning two unrelated targets from one repo: the author's macOS workstation (run locally) and a Debian box on AWS EC2 (run over SSH). There is no build, no test suite, and no lint config — the playbooks *are* the deliverable, and the only real "run" is applying them to a live machine.

## Commands

```bash
# macOS workstation (localhost)
ansible-playbook macos.yml --ask-become-pass

# Debian on EC2
ansible-playbook debian.yml

# Validate without touching the machine
ansible-playbook macos.yml --syntax-check
ansible-playbook macos.yml --list-tasks
ansible-playbook macos.yml --check --diff        # dry run; some `shell:` tasks still execute

# Run a subset — no tags are defined anywhere, so use these instead
ansible-playbook macos.yml --start-at-task "Ensure packages"
ansible-playbook macos.yml --step
```

`ansible.cfg` already points at both inventories (`hosts, aws_hosts`) and `roles_path = roles`, so run every command from the repo root.

## Architecture

**Two independent entry points.** `macos.yml` targets `localhost` (roles: `macos/homebrew`, `macos/system_settings`, `repos`). `debian.yml` targets the `mttRemoteDesktop` host (roles: `debian/system`, `debian/xrdp`). They share nothing but the repo.

**The two playbooks load variables differently — this trips people up.**

- `macos.yml` does `include_vars` on `macos_config.yml` with no `name:`, so its keys land in the global namespace and roles reference them bare: `homebrew_apps`, `dock`, `ssh_keys`, `repositories`.
- `debian.yml` loads `debian_config.yml` under `name: config` and `ansible_secrets.yml` under `name: secret`, so its roles reference them namespaced: `config.hostname`, `config.packages`, `secret.sudoer_users`.

When adding a variable, match the convention of the playbook you're touching.

**Secrets.** `ansible_secrets.yml` is gitignored; copy `ansible_secrets_example.yml`. Only `debian.yml` loads it — the macOS playbook runs fine without it, despite what the README implies.

**Inventory.** `hosts` pins localhost to `/opt/homebrew/bin/python3` (Apple Silicon path). `aws_hosts` hardcodes the EC2 public DNS name, which changes whenever the instance restarts — update it before running `debian.yml`.

## Role-specific gotchas

**`macos/homebrew` is install-only.** Every task defaults to `state: present`. Deleting or commenting out an entry in `homebrew_apps` / `homebrew_packages` does **not** uninstall anything — it only stops managing it. To actually remove something, use `- { name: webex, state: absent }`.

Commented-out cask entries in `macos_config.yml` are deliberate: they are kept so they can be re-enabled by uncommenting. Keep the `# - name` spacing consistent when adding one. Mac App Store entries (`homebrew_mas_apps`) need the `mas` formula (already in `homebrew_packages`) and the `community.general` collection.

**`macos/system_settings` edits `~/.zshrc` through `blockinfile` markers**, one marker per concern (`# BEGIN Prompt setting`, `# BEGIN NVM setting`, …). Adding a new shell export means adding a new `*_lines` variable plus a new `blockinfile` task with its own marker — reusing an existing marker silently overwrites that block. Setting a `*_lines` variable to `null` skips the task but leaves any previously written block in place; removing it needs an explicit `state: absent` task (see the "remove legacy ssh-add blocks" task for the pattern).

**`repos` role has a dead default.** `roles/repos/defaults/main.yml` declares `repos: []`, but the task loops over `repositories`. The real definition lives in `macos_config.yml`, where entries merge a YAML anchor (`<<: *personal_data`) supplying the shared ssh key, git remote, and destination folder.

**Disabled by design:** the `dockutil`-based Dock tasks (`roles/macos/system_settings/tasks/dock.yml`) and the vendored `gantsign.visual-studio-code-extensions` role in `macos.yml` are both commented out. Dock position/size instead goes through `defaults write` and the `m` CLI (`m-cli` formula).
