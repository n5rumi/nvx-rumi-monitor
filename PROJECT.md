# Rumi Monitor — the project, explained

> Open work lives in `gtm/rumi/TODO.md` and the JIRA `RUMI` project (Suite board, sprint
> `4.0.Mgmt.Beta1`). This file explains the project; it does not track tasks.

## What it is, in one breath

Rumi Monitor is **Grafana 10.1.1 with a Rumi coat of paint and three dashboards already hung on the
wall**. A Rumi container emits a heartbeat every few seconds. The Rumi Agent catches those heartbeats
and writes them into InfluxDB. Rumi Monitor reads InfluxDB and draws pictures. That is the whole job.

Think of it as a car's dashboard sold separately from the car: the engine (the container) produces
the numbers, the wiring loom (the agent + InfluxDB) carries them, and this repo is the instrument
cluster. It cannot make the car faster, but if a needle is wired to the wrong unit, the driver will
make terrible decisions — which, as you will see below, actually happened.

## The shape of the repo

This is a **fork of `grafana/grafana`**, so 99.9% of the tree is upstream Grafana and should be left
alone. Everything Rumi-specific is small and lives in a handful of places:

| Where | What | Why it is here |
|---|---|---|
| `conf/provisioning/dashboards/System/` | `host.json`, `container.json`, `service.json` | The three dashboards. Grafana's file provisioner loads them into a **System** folder at startup. |
| `conf/provisioning/dashboards/dashboards.yaml` | Provisioner config | Points Grafana at that folder (path relative to `/usr/share/grafana`). |
| `conf/provisioning/datasources/rumi-datasource.yaml` | The `rumi` InfluxDB data source | URL `influxdb.rumi.local:8086`, database `rumi`, user `root`, password `rootroot`. |
| `conf/grafana.ini` | Grafana settings | Copied to `/etc/grafana/grafana.ini` in the image. |
| `public/img/Rumi_*`, `public/app/core/components/Branding/`, `public/views/index-template.html`, `pkg/api/index.go` | Branding | Logos, login page, loading text. |
| `Dockerfile` | The image | Upstream's multi-stage build plus a "custom packaging" tail that swaps in our conf. |
| `tarball/` | RHEL 7.9 / Ubuntu tarball scripts | For a customer who could not run Docker. |
| `CHANGES.md`, `BUILD.md` | The fork's own notes | What we changed vs upstream, and how to build. |

**Branches.** `main` and `release-10.1.1` are pristine upstream — grep them for a dashboard and you
will find nothing, which looks alarmingly like "the dashboards were deleted". They were not. Rumi's
work lives on **`rumi-monitor-release-10.1.1`** (plus `-TEST`, and two `rumi-monitor-rhel7.9-*`
variants for the tarball line, which lag behind).

## How it is built and shipped

```
rumi-monitor-release-10.1.1 ──► TeamCity Monitor_Docker (10.1.1-RELEASE)
                                  full source build (yarn + go, amd64)
                                  push neeve/nvx-rumi-monitor:10.1.1.<n>, :10.1.1.latest, :latest
                                        │
                                        ▼
                   bake-rumi-ami skill (nvx-rumi-management Packer template)
                   nvx-rumi-monitor-amzn2023-x86_64-v10.1.1.<n> in Engg-DevOps
                                        │
                                        ▼
             AwsProvisioner.RUMI_MONITOR_SERVER_AMI_LATEST (nvx-rumi-cloud), shipped
             in the next Management milestone; Hetzner/Civo bakes pull :latest
```

- **TeamCity:** `Rumi / Management / Monitor / Docker`. `Monitor_Docker` is the release (manual,
  version `10.1.1.<build counter>`); `Monitor_Docker_1011snapshot` builds on every commit and pushes
  nothing. Neither bakes an AMI.
- **AMI:** the recipe is `nvx-rumi-cloud/src/main/resources/scripts/amzn/monitor/bake.sh` in
  `nvx-rumi-management`: Docker + `docker run --restart=always --network=host` of the pinned tag,
  data in `/home/rumi/grafana-data`. Every new AMI must be shared with the Paywhere and Datafye
  accounts by hand before anything points at it.
- **The dashboards also ship standalone** for customers who run their own Grafana; the docs live in
  `nvx-rumi-docs` under `management/guides/monitor/` and `management/reference/monitor/`.

## Words matter: Service and Container

Rumi 4.0 renamed the old X Platform vocabulary: an **XVM** (a.k.a. "VM" or "server") is now a
**container**, and an **app** is now a **service**. The dashboards were the last place still speaking
the old dialect, so RUMI-496 renamed every user-visible string — titles, row titles, dropdown labels,
column names, tooltips, text panels — and the "VM" dashboard became "Container".

What was **deliberately not renamed**: InfluxDB measurement names (`application`, `xvm.heartbeats.logger`),
tag and variable names (`vm_name`, `app_name`), and field regexes. Those are the wire contract with the
agent; renaming them would silently empty every panel. The rule of thumb: *rename what a human reads,
never what a query reads.*

The Container dashboard kept its uid (`-u-KrYh7z`), because links and bookmarks use the uid, not the
title. (The file was renamed `vm.json` → `container.json`; Grafana's file provisioner treats that as a
delete + create of the same uid, so links survive but stars and dashboard version history do not.)

## War stories (bugs we hit, and what they teach)

### 1. The chart that nearly re-sharded a customer (RUMI-497)

The "Message Processing Times" chart plotted median and max. The panel's unit was `ns` — correct, the
agent writes nanoseconds — but somebody had added a field override on the **max** series setting its
unit to `µs`. So a 30.2 ms worst case rendered as **30.2 s**. A customer read the chart, concluded their
handlers were taking tens of seconds, and proposed re-architecting their deployment. The real max was
82 ms. The table one row down showed the truth the whole time.

*Lesson:* a unit is a claim about the data, and a wrong one is worse than none. When two panels on the
same dashboard disagree, believe neither until you have read the raw number in InfluxDB. And when a
number is off by exactly 1000x, it is a units bug, every time — check both ends of the pipe (RUMI-244 is
the same class at the *emit* end, in the opposite direction).

### 2. The password nobody typed (RUMI-496)

The provisioned data source had `password:` — blank. With InfluxDB 1.x and auth off, that worked.
The agent moved to InfluxDB 2.x, which demands an 8+ character V1 password (`rootroot`), and from then
on every fresh monitor came up with a data source that said `authorization failed` and dashboards full
of "No data". Nothing *crashed*, which is exactly why it went unnoticed: an empty dashboard looks like a
quiet system.

*Lesson:* "No data" is an error state wearing a calm face. A health check (`/api/datasources/uid/<uid>/health`)
is a one-line proof the plumbing works; run it after every bake.

### 3. Grafana's field names are not your field names (RUMI-338)

The Version stat panel showed `-` even though InfluxDB held the jar version. A stat panel with an empty
field selector reduces **numeric fields only**, and the version is a string. The ticket proposed
selecting `/^jar_version$/` — reasonable, and wrong: Grafana names an InfluxQL field
`<measurement>.<field>`, i.e. `application.jar_version`, so the proposed regex matches nothing. It was
caught only because the fix was tested against a live Grafana via `/api/ds/query`, which returns the
frame's real `displayNameFromDS`.

*Lesson:* don't reason about what a framework will call something — ask it. Thirty seconds against the
real API beats an hour of confident guessing.

### 4. Suffix matches: `/$vm_name$/` (RUMI-498)

Forty Service-dashboard queries filtered with `=~ /$vm_name$/`. That is anchored at the end only, so
container `worker1` also matched `myworker1`, and the panels quietly summed both. Every other query in
the dashboard used `/^$vm_name$/`; these forty were simply copied from an older panel. Anchoring them
fixed it. (Still open, noted in the TODO: values are interpolated **unescaped**, so a name containing a
regex metacharacter like `.` can still over-match; tag equality or `${vm_name:regex}` would close it.)

*Lesson:* when one panel in a family differs from its siblings, that difference is either the point or
the bug. Diff the siblings.

### 5. The clone that forgot to change (RUMI-238)

The "Per Message Type Stats" row had two identical "Message Create To Offer Times" tables and no
filtering table, although the agent emits five latency kinds (`CreateToOffer`, `OfferToPoll`,
`MsgFiltering`, `MsgPreProc`, `MsgProcessing`). Someone cloned the panel to make the filtering one and
never changed the title or the `WHERE`. A customer spotted the duplicate before we did.

## How to verify a dashboard change (the recipe that worked)

1. Build the image (or overlay `conf/` on the last release for a quick local check — the last
   release's `conf/` must match your branch's parent byte for byte for that to be honest).
2. Run InfluxDB **with auth on** (`influxdb:1.8`, `INFLUXDB_HTTP_AUTH_ENABLED=true`, admin
   `root`/`rootroot`, RP `rumi_heartbeat_rp`) on a Docker network with the alias `influxdb.rumi.local`.
3. Start the **previous** image on a named Grafana volume, then the new one on the same volume — that
   is the upgrade path real monitors take.
4. Write line-protocol points shaped like the agent's (`XVMHeartbeatCollector` in `nvx-rumi-agent` is
   the authority for measurement, tag and field names), including a decoy container whose name is a
   suffix of the real one.
5. Check each panel two ways: `/api/ds/query` with the panel's targets (numbers), and the browser with
   `viewPanel=<id>` (rendering — units, column names). Anonymous viewer access
   (`GF_AUTH_ANONYMOUS_ENABLED=true`) avoids a login on a throwaway local instance.

## Pitfalls to remember

- **The upstream README is Grafana's.** Don't take build or licensing advice for *this* fork from it;
  `BUILD.md` and `CHANGES.md` are ours.
- **`gh` defaults to the fork parent** (`grafana/grafana`). Always pass `--repo n5rumi/nvx-rumi-monitor`
  (or `gh repo set-default`), or `gh pr create` fails with "No commits between ...".
- **The repo's husky hooks need `node_modules`.** For a conf-only commit without a JS install, the hooks
  cannot run; be deliberate about that.
- **Hetzner and Civo monitors run `:latest`.** Pushing `latest` is a release for them.
- **The RHEL 7.9 branches lag.** Fixes on `rumi-monitor-release-10.1.1` do not reach them on their own.

## How good engineers approached this

The interesting part of this project was never Grafana. It was **refusing to trust the display**. Every
bug above was a display that looked plausible: a calm "No data", a confident "30.2 s", a tidy "-". The
habit that found them was the same each time — go one layer down (the query, the raw InfluxDB value,
the agent source that wrote it) and check that the picture agrees with the numbers.
