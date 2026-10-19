# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The parent `salt-formulas/CLAUDE.md` describes the shared toolchain (Kitchen,
pre-commit, CI, the three Debian/Salt platforms) and the `template-formula`
layout. This file only records where `logrotate-formula` departs from it.

## Commands

```bash
bundle install
bin/kitchen list                                # 3 instances: default-<platform>
bin/kitchen test default-debian-13-3008-2       # one instance, end to end
bin/kitchen converge default-debian-11-3006-27  # apply only, keep the container
bin/kitchen verify default-debian-11-3006-27    # rerun the InSpec controls
pre-commit run --all-files
```

There is a single Kitchen suite, `default`. Its pillar is `pillar.example`
itself (`pillars_from_files`), so editing `pillar.example` changes what the
tests converge and must stay in step with `test/integration/default/controls/`.

The suite pulls the **upstream** `saltstack-formulas/cron-formula` over git as a
dependency, and its `state_top` runs `logrotate._mapdata`, `cron`, then
`logrotate`.

## Layout — not the template one

- States are flat files under `logrotate/`, not `package/` / `config/` /
  `service/` subdirectories: `install.sls`, `config.sls`, `service.sls`,
  `jobs.sls`, all included by `init.sls`. Requisites use literal IDs
  (`pkg: logrotate-pkg`, `file: logrotate-config`), not `sls:` built from
  `tplroot`.
- `map.jinja` merges four YAML layers, not five: there is no `osarchmap.yaml`
  (`defaults` <- `osfamilymap` <- `osmap` <- `osfingermap` <- `lookup` <-
  `logrotate` pillar/config).
- No `libtofs.jinja`: templates are under `logrotate/templates/` and referenced
  by a literal `salt://logrotate/templates/...` source.
- No `_mapdata` reference dumps under `test/integration/default/files/`, and no
  control compares them: `logrotate._mapdata` runs but nothing asserts on it.

## How a job becomes a file

`logrotate.jobs` iterates `pillar.get('logrotate:jobs')` (the raw pillar, not
the merged map) and writes one file per key, named after the **last path
segment** of the key (`/tmp/var/log/mysql/error` -> `/etc/logrotate.d/error`).
Two pillar forms work:

- `{path: [...], config: [...]}` — rendered through `templates/job.tmpl`;
  `path` defaults to `[<key>]`.
- `{contents: "..."}` — written verbatim, no template.

`jobs.sls` also has a branch for a bare list (key as path, list as config),
but it is unreachable: `value.get('contents')` runs first and fails on a list,
as does `value.config` in `config.sls`.

A job whose `contents` or `config` contains `hourly` goes to
`hourly_include_dir` instead. `config.sls` runs the same test over the merged
map's `jobs` to decide whether to create the hourly config file, directory and
`/etc/cron.hourly/logrotate` script; the two sides must agree.

`jobs.sls` includes `logrotate`, so applying `logrotate.jobs` alone still
installs the package, rewrites `/etc/logrotate.conf` from
`templates/logrotate.conf.tmpl` and manages the service.

`logrotate.conf.tmpl` renders `default_config` (pillar `logrotate:default_config`
first, map value otherwise): `true` -> `<param>`, `false` -> `no<param>`, a
scalar -> `<param> <value>`; the known periods and `rotate`, `create`,
`dateext`, `compress` come first, then any other key. `login_records_jobs`
appends the `wtmp`/`btmp` blocks; it is `false` on Debian, whose package
already rotates them.

Owner/group/mode lookups differ by state: `config.sls` uses `config.get`,
`jobs.sls` uses `pillar.get`, both on `logrotate:config:{user,group,mode}`.

## Debian specifics

On `os_family: Debian`, `service` is `logrotate.timer` (the package schedules
logrotate with a systemd timer). The InSpec service control still checks
`cron` on Debian, which the `cron` dependency formula keeps running.
