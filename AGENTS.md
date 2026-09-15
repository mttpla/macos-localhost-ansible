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

So five casks stay installed here even though the `Brewfile` deliberately omits them: `antigravity` `crystalfetch` `dbeaver-community` `firefox@developer-edition` `webex`. `brew bundle cleanup` will always list them — that is expected, not drift to fix. A blind `brew bundle dump --force` will put all five straight back into the `Brewfile`, so always diff the result before committing it. Removing them for real needs an IT request.

### Casks deliberately untracked

Eleven casks had their receipt under `/opt/homebrew/Caskroom` deleted on purpose. Homebrew no longer knows about them, so they no longer appear in `brew outdated` or `brew bundle cleanup`. **Do not re-add them to the `Brewfile`** — `brew bundle install` would reinstall into `/Applications` and recreate the exact problem. The apps themselves are untouched and still work.

The diagnostic that decides whether a cask belongs here: compare the Caskroom receipt version against `CFBundleShortVersionString` of the real app.

```sh
brew list --cask --versions <cask>
defaults read "/Applications/<App>.app/Contents/Info.plist" CFBundleShortVersionString
```

If the app is *newer* than the receipt, the app self-updates and Homebrew can never catch up — upgrading it means purging `/Applications/<App>.app`, which endpoint policy blocks. The receipt is then permanently stale and every `brew upgrade` retries the same failure.

| cask | untracked | reason |
|---|---|---|
| `libreoffice` | 2026-09-09 | upgrade loop: stale app at target path, purge needs interactive sudo |
| `gimp` | 2026-09-15 | upgrade loop: receipt pinned at 3.2.4 while `/Applications` was already 3.2.6 |
| `clipy` | 2026-09-15 | self-updating: receipt 1.2.1, app 1.3.0 |
| `cyberduck` | 2026-09-15 | self-updating: receipt 8.7.2, app 9.5.4 |
| `firefox` | 2026-09-15 | self-updating: receipt 153.0.1, app 155.0.1 |
| `miro` | 2026-09-15 | self-updating: receipt 0.7.32, app 0.11.168 |
| `spotify` | 2026-09-15 | self-updating |
| `sublime-text` | 2026-09-15 | self-updating: receipt 4143, app Build 4200 |
| `zed` | 2026-09-15 | self-updating: receipt 1.1.6, app 1.19.2 |
| `utm` | 2026-09-15 | no `UTM.app` anywhere on disk; the receipt only held a 1.1G orphaned stage copy |
| `telegram` | 2026-09-15 | tracked the wrong app entirely — see below |

`gimp` `clipy` `miro` `sublime-text` `zed` `telegram` were in the `Brewfile` and were removed from it at the same time; the rest never were.

**What stays tracked, and why it must:** casks Homebrew can genuinely still upgrade, because they never touch `/Applications`.

- `claude-code` `codex` `copilot-cli` — artifact `binary`, installed into `/opt/homebrew/bin`.
- `temurin` `temurin@8` `temurin@11` `temurin@17` `temurin@21` — artifact `pkg` with an `uninstall` stanza, removed via `pkgutil`.
- `crystalfetch` `kindle` — receipt and app still in step.

Untracking any of these would trade a working update channel for nothing.

One is still undecided and was left tracked on purpose:

- `teamviewer` — receipt 15.42.4 but the app is 15.81.6, so it self-updates like the untracked group. It is a `pkg` cask with an `uninstall` stanza, though, so Homebrew may be able to upgrade it where a plain app bundle cannot. Try a single `brew upgrade --cask teamviewer` before deciding.

### `telegram` tracked a different app than the one installed

Worth reading before trusting any version comparison here, because the version numbers alone actively mislead.

The `telegram` cask is **Telegram for macOS** (macos.telegram.org, bundle id `ru.keepcoder.Telegram`), installed May 2023 with receipt 9.6.3 and now at 12.10 upstream. But `/Applications/Telegram.app` is **Telegram Desktop** (tdesktop, bundle id `com.tdesktop.Telegram`), version 7.1.3, signed by Telegram FZ-LLC in August 2026. Two separate projects that happen to install to the same path under the same name — so `brew outdated` reporting `9.6.3 != 12.10` said nothing at all about the app actually on disk.

Telegram Desktop self-updates: `~/Library/Application Support/Telegram Desktop/tupdates/temp/` already held Telegram 7.2.5, staged and waiting for a restart. The Caskroom entry was 8K — a bare symlink to `/Applications/Telegram.app`, no staged copy.

The reason untracking mattered more than usual: `brew upgrade --cask telegram` would have installed Telegram for macOS *over* `/Applications/Telegram.app`, replacing Telegram Desktop with a different client. The two keep their data in different locations, so the session and config would have looked wiped.

Check `CFBundleIdentifier`, not just the version, whenever a cask's version gap looks implausible:

```sh
defaults read "/Applications/<App>.app/Contents/Info.plist" CFBundleIdentifier
```

**`macos/system_settings` edits `~/.zshrc` through `blockinfile` markers**, one marker per concern (`# BEGIN Prompt setting`, `# BEGIN NVM setting`, …). Adding a new shell export means adding a new `*_lines` variable plus a new `blockinfile` task with its own marker — reusing an existing marker silently overwrites that block. Setting a `*_lines` variable to `null` skips the task but leaves any previously written block in place; removing it needs an explicit `state: absent` task (see the "remove legacy ssh-add blocks" task for the pattern).

**`repos` role has a dead default.** `roles/repos/defaults/main.yml` declares `repos: []`, but the task loops over `repositories`. The real definition lives in `macos_config.yml`, where entries merge a YAML anchor (`<<: *personal_data`) supplying the shared ssh key, git remote, and destination folder.

**Disabled by design:** the `dockutil`-based Dock tasks (`roles/macos/system_settings/tasks/dock.yml`) and the vendored `gantsign.visual-studio-code-extensions` role in `macos.yml` are both commented out. Dock and Finder settings go through `community.general.osx_defaults`, which reads before writing — so `changed` means changed. `killall Finder` and `killall Dock` are handlers and fire only on a real change. Do not replace these with `shell: defaults write`: that always reports changed and makes a no-op run indistinguishable from a real one.
