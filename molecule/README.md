<!--
SPDX-FileCopyrightText: 2018-2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

Currently there is one testing scenario available.

### `default`

Tests a standard pump-it-up-tracker installation, served under a path prefix and on a non-default port. It uses a small score log written the way a person would write one: unquoted dates, a zero-padded score in quotes, and a real result screen.

What it checks:

- The container becomes `healthy`. The image's healthcheck only finds the non-default port through the environment file the role renders.
- `/api/data.json` reports exactly the plays, songs, personal bests, plates and dates from the inventory, and the version of the application matches `piu_tracker_version`.
- Jacket art comes from `piu_tracker_custom_art_path`, and songs without art get a placeholder. Fetching from the internet is off, so the scenario does not depend on the art sources being reachable.
- Song pages link under the path prefix, and the slashless prefix redirects.
- The container runs with a read-only root filesystem, no capabilities and the configured user. The data file is mounted read-only.
- Running the role with a mistyped result screen (`great: 031`, which YAML reads as octal) fails at the application's own validation with a precise message. The installed data file and the running service are left untouched.
- The service does not restart over 45 seconds of watching.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default
```

### Testing an unpublished image

By default the scenario pulls the image `piu_tracker_version` points at. To test an image that has not been published yet, build it from a checkout of [pump-it-up-tracker](https://github.com/spatterIight/pump-it-up-tracker) and hand it over as an archive. The role then skips pulling:

```bash
docker build --build-arg VERSION=1.1.1 -t pump-it-up-tracker:dev ../pump-it-up-tracker
docker save pump-it-up-tracker:dev -o /tmp/pump-it-up-tracker.tar
PIU_TRACKER_MOLECULE_IMAGE_ARCHIVE=/tmp/pump-it-up-tracker.tar molecule test --scenario-name default
```

Pass the value of `piu_tracker_version` as `VERSION`, since the scenario checks the version the application reports.
