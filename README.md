<!--
SPDX-FileCopyrightText: 2025, 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# pump-it-up-tracker Ansible role

This is an [Ansible](https://www.ansible.com/) role which installs [pump-it-up-tracker](https://github.com/spatterIight/pump-it-up-tracker) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

pump-it-up-tracker is a read-only web application for personal tracking of Pump It Up scores. You log each result screen as an entry in `piu_tracker_scores`, and this role turns the entries into the data file the application reads:

```yaml
piu_tracker_scores:
  - song: Big Daddy
    chart: S11
    date: 2026-09-28
    score: 938204
    plate: TG
    judgments: {perfect: 506, great: 31, good: 11, bad: 7, miss: 6}
    max_combo: 294
```

Each run checks the data with the application's own validator before replacing the live file. A typo (such as a score that does not match its judgments) fails the run with a precise message, instead of reaching the running service.

This role *implicitly* depends on:

- [`com.devture.ansible.role.playbook_help`](https://github.com/devture/com.devture.ansible.role.playbook_help)
- [`com.devture.ansible.role.systemd_docker_base`](https://github.com/devture/com.devture.ansible.role.systemd_docker_base)

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options. Refer to [this page](docs/configuring-piu-tracker.md) for details about setting up the service with this role.

💡 For an Ansible playbook which integrates this role and makes it easier to use, see the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

## Development

### pre-commit

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```

### Molecule

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

Refer to [this page](./molecule/README.md) for details about how to utilize it.
