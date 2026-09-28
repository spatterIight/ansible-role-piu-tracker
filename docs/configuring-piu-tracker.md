<!--
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2025, 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up pump-it-up-tracker

This is an [Ansible](https://www.ansible.com/) role which installs [pump-it-up-tracker](https://github.com/spatterIight/pump-it-up-tracker) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

pump-it-up-tracker is a read-only web application for personal tracking of [Pump It Up](https://www.piugame.com/) scores. You log your result screens as Ansible variables. This role turns them into the data file the application reads, and the application shows:

- your songs with jacket art
- per-chart personal bests and progress charts
- every attempt with its judgments
- your sessions, day by day

## Adjusting the playbook configuration

To enable pump-it-up-tracker with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# piu_tracker                                                          #
#                                                                      #
########################################################################

piu_tracker_enabled: true

piu_tracker_hostname: piu.example.com

piu_tracker_player_name: PUMPITUP

piu_tracker_scores: []

########################################################################
#                                                                      #
# /piu_tracker                                                         #
#                                                                      #
########################################################################
```

To host it under a subpath rather than at the root of a hostname, also set `piu_tracker_path_prefix` (e.g. `/piu`). The application serves itself under that prefix, so nothing needs to be rewritten by the reverse-proxy.

### Logging a result screen

Add one entry to `piu_tracker_scores` per result screen:

```yaml
piu_tracker_scores:
  - song: Big Daddy
    chart: S11                 # the level ball: S11, D21, SP12, DP20, CoOp2
    date: 2026-09-28           # or "2026-09-28 18:45" to order plays within a day
    score: 938204
    plate: TG                  # PG UG EG SG MG TG FG RG, or the full name ("Talented Game")
    judgments: {perfect: 506, great: 31, good: 11, bad: 7, miss: 6}
    max_combo: 294
    kcal: 31.137
    note: "finally"            # optional
```

- **Always required:** `song`, `chart` and `date`.
- **Score:** either give `score`, or give both `judgments` and `max_combo` and the score is worked out from them.
- **Stage breaks:** add `broken: true`.
- **Grade:** worked out from the score. Set `grade` only if you want it double-checked.

>[!IMPORTANT]
> The cabinet pads numbers with zeros (`031`), but YAML reads numbers with a leading zero as octal, so `great: 031` would become 25. Write numbers without the leading zeros.

When `judgments` and `max_combo` are both given, the score must match what the game awards for them. This catches most transcription mistakes, including the octal one above.

#### Keeping the score log in its own file

A long score log is easier to maintain in a file of its own, next to `vars.yml`:

```yaml
piu_tracker_scores: "{{ lookup('ansible.builtin.file', inventory_dir + '/host_vars/mash.example.com/piu-scores.yml') | from_yaml }}"
```

where `piu-scores.yml` holds only the list:

```yaml

- song: Big Daddy
  chart: S11
  date: 2026-09-28
  score: 938204
```

#### Song details

`piu_tracker_songs` optionally adds details about songs, keyed by title (matched case-insensitively):

```yaml
piu_tracker_songs:
  Conflict:
    artist: Siromaru + Cranky
    bpm: 160
  Big Daddy:
    image: https://www.piugame.com/data/song_img/<id>.png
```

### How mistakes are caught

Before the data file is replaced, this role checks it with the application's own `validate` command, run in a throwaway container. If anything is wrong, the playbook run fails and lists every problem, for example:

```text
scores[42] (Big Daddy S11): score 938204 does not match the judgments and max combo, which give 944579; check for a typo
```

The data file already in place, and the running service, are left untouched. To skip the check, set `piu_tracker_data_validation_enabled: false`.

### Jacket art

For each song, art is looked up in this order and downloaded once into `/piu-tracker/data/art`:

1. Your own images in `piu_tracker_custom_art_path`, if set. Name each file after the song (`big-daddy.png`), or refer to it with the song's `image`.
2. The song's `image`, when it is a URL.
3. [PIU Scores](https://piuscores.arroweclip.se), a community site with jackets for about 1,000 songs.
4. The [PIU Fandom wiki](https://pumpitup.fandom.com).
5. A generated placeholder, until something is found.

To never download anything, set:

```yaml
piu_tracker_art_fetch_enabled: false
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `piu_tracker_environment_variables_additional_variables` variable.
- The application's [README](https://github.com/spatterIight/pump-it-up-tracker#configuration) for the environment variables it supports.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`.

After logging new scores, only this service needs to be updated:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-piu-tracker,start
```

The service is restarted automatically when the scores changed.

## Usage

After running the command for installation, pump-it-up-tracker becomes available at the specified hostname like `https://piu.example.com`.

The application is read-only and has no login. If it should not be public, put authentication in front of it at the reverse-proxy. For example, with Traefik [basic auth](https://doc.traefik.io/traefik/middlewares/http/basicauth/) (generate `USER:HASH` with `htpasswd -nB USER`):

```yaml
piu_tracker_container_labels_additional_labels_custom:
  - "traefik.http.middlewares.piu-tracker-auth.basicauth.users=USER:HASH"

piu_tracker_container_labels_traefik_additional_middlewares_custom:
  - piu-tracker-auth
```

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu piu-tracker` (or how you/your playbook named the service, e.g. `mash-piu-tracker`).
