<!--
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2025, 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up pump-it-up-tracker

This is an [Ansible](https://www.ansible.com/) role which installs [pump-it-up-tracker](https://github.com/spatterIight/pump-it-up-tracker) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

pump-it-up-tracker is a read-only web application for personal tracking of [Pump It Up](https://www.piugame.com/) scores, which you log as Ansible variables.

See the project's [documentation](https://github.com/spatterIight/pump-it-up-tracker#readme) to learn what pump-it-up-tracker does and why it might be useful to you.

## Adjusting the playbook configuration

To enable pump-it-up-tracker with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# piu-tracker                                                          #
#                                                                      #
########################################################################

piu_tracker_enabled: true

########################################################################
#                                                                      #
# /piu-tracker                                                         #
#                                                                      #
########################################################################
```

### Set the hostname

To enable pump-it-up-tracker you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
piu_tracker_hostname: "example.com"
```

### Add your scores

Each result screen is one entry in `piu_tracker_scores`. To add them, add the following configuration to your `vars.yml` file:

```yaml
piu_tracker_scores:
  - song: Big Daddy
    chart: S11                 # the level ball: S11, D21, SP12, DP20, CoOp2
    date: 2026-09-28           # or "2026-09-28 18:45" to order plays within a day
    score: 938204
    plate: TG                  # PG UG EG SG MG TG FG RG, or the full name
    judgments: {perfect: 506, great: 31, good: 11, bad: 7, miss: 6}
    max_combo: 294
    kcal: 31.137
    note: "finally"
```

`song`, `chart` and `date` are required, together with either `score` or both `judgments` and `max_combo` (the score is then worked out from them). Add `broken: true` for a stage break. See [`defaults/main.yml`](../defaults/main.yml) for the details of every key.

A failed play whose result screen shows `-` instead of a score is logged with `broken: true` and no score:

```yaml
piu_tracker_scores:
  - {song: DUEL, chart: S13, date: 2026-09-08, broken: true}
```

Such an entry may only have `song`, `chart`, `date`, `kcal` and `note`. It shows as a failed attempt, and never counts as a clear or a personal best.

Plays are from Pump It Up Phoenix unless they say otherwise. A play from Prime 2 or XX names its version with `version` (`prime2` or `xx`; `phoenix` is the default):

```yaml
piu_tracker_scores:
  - song: Le Grand Bleu
    chart: S7
    version: prime2
    date: 2025-09-09
    score: 1038500             # required for Prime 2 and XX, and can be over 1,000,000
    grade: S                   # optional, shown as logged; there are no plates
```

Scores and grades are kept as each version's result screen showed them, and a chart of one version is a different chart from the same level of another. The front page's headline stats cover the newest version you have played. To follow a chart into a newer version, possibly at another level, link it with `continues` on one of its plays, e.g. `continues: {version: phoenix, chart: S7}` on a play of that chart's S8 in a later version. See the project's [README](https://github.com/spatterIight/pump-it-up-tracker#game-versions) for what each version checks and how personal bests carry across a link.

>[!IMPORTANT]
> Write numbers without the leading zeros the cabinet shows: YAML reads `031` as the octal number 25. When `judgments` and `max_combo` are both given, the score is checked against them, which catches such mistakes.

Before the data file is replaced, the role checks it with the application's own validator. A mistake fails the run with a message naming the entry, and leaves the installed data and the running service as they were.

A long score log can live in a file of its own, next to `vars.yml`. That file holds only the list, which is then loaded as below:

```yaml
piu_tracker_scores: "{{ lookup('ansible.builtin.file', inventory_dir + '/host_vars/mash.example.com/piu-scores.yml') | from_yaml }}"
```

### Set the player name (optional)

The player name is shown on the front page. To set it, add the following configuration to your `vars.yml` file:

```yaml
piu_tracker_player_name: "YOUR NAME"
```

### Add song details (optional)

You can add an artist, a BPM or your own jacket art to songs, keyed by title. To do so, add the following configuration to your `vars.yml` file:

```yaml
piu_tracker_songs:
  Conflict:
    artist: Siromaru + Cranky
    bpm: 160
  Big Daddy:
    image: https://www.piugame.com/data/song_img/<id>.png
```

### Configure jacket art (optional)

Jacket art is found automatically on [PIU Scores](https://piuscores.arroweclip.se) and the [PIU Fandom wiki](https://pumpitup.fandom.com), and cached in the data path. Songs without any show a generated placeholder.

To use your own images instead, set `piu_tracker_custom_art_path` to a directory on the server holding them, named after the song (e.g. `big-daddy.png`) or referred to by a song's `image`. To stop downloading art altogether, add the following configuration to your `vars.yml` file:

```yaml
piu_tracker_art_fetch_enabled: false
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `piu_tracker_environment_variables_additional_variables` variable
- The pump-it-up-tracker [README](https://github.com/spatterIight/pump-it-up-tracker#configuration) for all the environment variables it supports.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, pump-it-up-tracker becomes available at the specified hostname like `https://example.com`.

To log new scores, add them to `piu_tracker_scores` and run the installation command again. The service is restarted when the scores change.

>[!NOTE]
> The `piu_tracker_path_prefix` variable can be adjusted to host under a subpath (e.g. `piu_tracker_path_prefix: /piu`).

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu piu-tracker` (or how you/your playbook named the service, e.g. `mash-piu-tracker`).
