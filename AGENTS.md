# AGENTS.md

This file provides guidance to coding agents when working with code in this repository.

## What this is

An Ansible playbook that provisions the author's macOS workstation, run locally against `localhost`. There is no build, no test suite, and no lint config — the playbook *is* the deliverable, and the only real "run" is applying it to a live machine.

A second playbook provisioned a Debian box on AWS EC2. It was unused for years and was deleted; `git log -- debian.yml` has it if it is ever wanted back.

## Commands

```bash
ansible-playbook macos.yml

# Validate without touching the machine
ansible-playbook macos.yml --syntax-check
ansible-playbook macos.yml --list-tasks
ansible-playbook macos.yml --check --diff        # dry run; some `shell:` tasks still execute

# Run a subset — no tags are defined anywhere, so use these instead
ansible-playbook macos.yml --start-at-task "Set the hot corners"
ansible-playbook macos.yml --step
```

`ansible.cfg` already points at the `hosts` inventory and `roles_path = roles`, so run every command from the repo root.

## Architecture

**One entry point.** `macos.yml` targets `localhost` and runs three roles: `macos/homebrew`, `macos/system_settings`, `repos`.

**Variables are global.** `macos.yml` does `include_vars` on `macos_config.yml` with no `name:`, so its keys land in the global namespace and roles reference them bare: `dock`, `ssh_keys`, `repositories`. No secrets are involved — nothing in the playbook reads `ansible_secrets.yml`.

**Inventory.** `hosts` pins localhost to `/opt/homebrew/bin/python3` (Apple Silicon path).

**Renaming the machine is off.** `computername` and `hostname` are `null`, and the `scutil` tasks are guarded by `when: != None`, so they skip. They were set to `mttPC-2023`, a personal Mac from 2023; on the corporate machine this repo now runs on, renaming would break the IT inventory. Set them only on a machine you own.

## Role-specific gotchas

**Package lists live in the `Brewfile`, not in Ansible.** `macos_config.yml` holds only settings. The `macos/homebrew` role shells out to `brew bundle install --file=Brewfile --no-upgrade`; taps, formulae, casks, Mac App Store apps, VS Code extensions and a few `cargo`/`npm` globals are all declared there.

Working with it:

```bash
brew bundle check --file=Brewfile      # something missing or outdated?
brew bundle cleanup --file=Brewfile    # dry run: installed but NOT in the Brewfile
brew bundle dump --force               # regenerate from the current machine
```

Three things that bite:

- **`brew bundle install` is still install-only.** Dropping a line never uninstalls. Removal goes through `brew bundle cleanup --force`, which is destructive and deliberately not wired into the playbook.
- **`check` conflates "missing" with "outdated"** — it reports `needs to be installed or updated` for packages that are present but stale, so a non-zero exit is not proof anything is absent.
- **`dump` silently skips untrusted third-party taps.** Four packages (`sshpass`, `sdkman-cli`, `mongodb-database-tools`, the arm64 cross toolchain) were missing from the generated file and are now listed by hand. Re-check them after every `dump`, and note that `brew bundle install` refuses to run at all until those taps are trusted with `brew trust <tap>` — a per-machine setting that no Brewfile flag can grant.

`homebrew_upgrade_all` (default `false`) gates a `brew upgrade` before the bundle runs. A bootstrap should not double as a package updater.

**This machine cannot uninstall apps from `/Applications`.** It is corporate-managed: the account is in `staff`, not `admin`, `/Applications` is `root:admin drwxrwxr-x`, and endpoint policy blocks `sudo rm -R -f /Applications/<app>` outright — with or without `brew uninstall --cask --force`. Casks whose removal only needs `pkgutil` (`zoom`, `microsoft-onenote`, `dotnet-sdk`, `openvpn-connect`, `powershell`, `postman`) do uninstall; plain app bundles do not.

So ten casks stay installed here even though the `Brewfile` deliberately omits them: `antigravity` `crystalfetch` `cyberduck` `dbeaver-community` `firefox` `firefox@developer-edition` `libreoffice` `spotify` `utm` `webex`. `brew bundle cleanup` will always list them — that is expected, not drift to fix. A blind `brew bundle dump --force` will put all ten straight back into the `Brewfile`, so always diff the result before committing it. Do not "tidy up" by deleting their receipts under `/opt/homebrew/Caskroom`: the apps would remain on disk and simply stop receiving updates. Removing them for real needs an IT request.

**`macos/system_settings` edits `~/.zshrc` through `blockinfile` markers**, one marker per concern (`# BEGIN Prompt setting`, `# BEGIN NVM setting`, …). Adding a new shell export means adding a new `*_lines` variable plus a new `blockinfile` task with its own marker — reusing an existing marker silently overwrites that block. Setting a `*_lines` variable to `null` skips the task but leaves any previously written block in place; removing it needs an explicit `state: absent` task (see the "remove legacy ssh-add blocks" task for the pattern).

**`repos` role has a dead default.** `roles/repos/defaults/main.yml` declares `repos: []`, but the task loops over `repositories`. The real definition lives in `macos_config.yml`, where entries merge a YAML anchor (`<<: *personal_data`) supplying the shared ssh key, git remote, and destination folder.

**Disabled by design:** the `dockutil`-based Dock tasks (`roles/macos/system_settings/tasks/dock.yml`) and the vendored `gantsign.visual-studio-code-extensions` role in `macos.yml` are both commented out. Dock and Finder settings go through `community.general.osx_defaults`, which reads before writing — so `changed` means changed. `killall Finder` and `killall Dock` are handlers and fire only on a real change. Do not replace these with `shell: defaults write`: that always reports changed and makes a no-op run indistinguishable from a real one.
