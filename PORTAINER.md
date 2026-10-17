# Portainer GitOps deployment

Portainer reads the compose file from the public `ferrule-deploy`
repository and pulls the published image. Nothing is built on the Docker
host. After the initial setup, updates are a single click — or zero clicks
with a webhook.

## Before you start

- Ferrule runs from a published image, a built binary: nothing is compiled
  on your Docker host. Portainer reads only the compose file from the
  repository.
- Portainer must reach
  `https://git.uglyhousesneedlove.com/steve/ferrule-deploy`. It is public,
  so no account or token is needed.
- The Docker host needs outbound internet to pull the Ferrule and
  VictoriaMetrics images. A licensed install also fetches
  `https://license.ferrulehq.dev/revoked` once a day (see "Online
  confirmation").
- Nothing goes on the host filesystem. No bind mounts, no config files.

---

## Step 1 — Create the UniFi credentials

Two kinds, on each console. The API key alone cannot see fan speeds, port
error counters, SFP diagnostics or disk health.

**Do this on your gateway (UDM-SE, Cloud Gateway) for Network.** Then repeat
the whole thing on your **UNVR** if Protect runs there — it is a separate
UniFi OS console with its own credential store, and the gateway's API key and
admin account will not authenticate against it.

1. Sign in to that console's UniFi OS interface directly by IP, not the
   mobile app and not the cloud portal.
2. **Settings → Control Plane → Integrations → Create API Key**. Name it
   `telemetry` and copy the key — UniFi shows it exactly once.
3. **Settings → Admins & Users → Add New Admin**.
4. Choose **Restrict to Local Access Only**, set a username and password, and
   give it the **Read Only** role.
5. Leave MFA **off** for this account. Cookie login cannot satisfy a second
   factor, and that login is what reaches the legacy API and the Protect
   bootstrap document, where the interesting telemetry lives.

You should end up with two API keys and two local accounts — one pair per
console.

---

## Step 2 — The image

The image is public: Portainer pulls it with no registry login. It holds
the built binary only, never the source. Before launch it is published at
`git.uglyhousesneedlove.com/steve/ferrule`; at launch it moves to Docker Hub,
and `FERRULE_IMAGE` is the one line that changes.

---

## Step 3 — Create the stack

1. **Stacks → Add stack**.
2. Name it `ferrule` — lowercase, no spaces. Portainer uses this as
   the compose project name, which also keeps its volumes apart from any
   other stack's.
3. Choose **Repository** as the build method.
4. Fill in:
   - **Repository URL** —
     `https://git.uglyhousesneedlove.com/steve/ferrule-deploy`
   - **Repository reference** — `refs/heads/main`
   - **Compose path** — `docker-compose.yml`
   - **Authentication** — off; the repository is public. Turn it on only
     when deploying from a private copy (see "Deploying from someone else's
     private repo")
5. Leave **Enable relative path volumes** off. The stack has no bind mounts.
   It does declare two *named* volumes, `vm-data` for VictoriaMetrics and
   `ledger-data` for the outage archive. Portainer creates both itself and
   neither needs a host path, which is the point — a relative path volume is
   exactly what would break here.

---

## Step 4 — Add the stack variables

Scroll to **Environment variables** and switch to **Advanced mode** if
offered — you can paste the whole block at once. The contents of
`stack.env.example`, with your values filled in:

```
FERRULE_VERSION=0.9.0

UNIFI_HOST=https://192.168.1.1
UNIFI_API_KEY=gateway-api-key
UNIFI_USERNAME=telemetry
UNIFI_PASSWORD=gateway-password

PROTECT_HOST=https://192.168.1.4
PROTECT_API_KEY=unvr-api-key
PROTECT_USERNAME=telemetry
PROTECT_PASSWORD=unvr-password
```

Omit the four `PROTECT_*` lines if one console runs both Network and Protect;
the Network credentials are then reused automatically.

Everything else has a working default. Once the dashboard is up,
**Settings → Stack environment** writes every variable in this file for you,
starting from what the stack is running with: change what you need and it
gives the lines to paste here. It never asks for a secret. A credential from
elsewhere gets a placeholder to replace in Portainer, and the keys and
passwords that are Ferrule's own (`BACKUP_KEY`, `SETTINGS_KEY`,
`ADMIN_PASSWORD`, the database passwords) can be made in the browser, never
sent anywhere.

Add these only if you need them:

| Name | Default | When to change it |
|---|---|---|
| `FERRULE_VERSION` | **required** | the release to run, such as `0.9.0`; `qa` for what is under test. Never `latest` |
| `FERRULE_IMAGE` | `git.uglyhousesneedlove.com/steve/ferrule` | where the image is published; change it only if told to |
| `WEB_PORT` | `8090` | something else already has 8090 |
| `UNIFI_ENABLED` | `true` | `false` to watch only hypervisors and storage, or to try this stack beside an existing one without polling the controller twice |
| `SOURCE_1` … `SOURCE_8` | empty | a hypervisor or storage system per slot, see below |
| `SOURCE_n_SECRET` | empty | the matching token, password or key |
| `POLL_INTERVAL` | `30s` | values below `10s` are clamped |
| `REQUEST_TIMEOUT` | `20s` | how long one request to a console may take |
| `UNIFI_SITE` | `default` | you renamed your site |
| `UNIFI_OS` | `true` | `false` for a self-hosted standalone controller |
| `UNIFI_VERIFY_TLS` | `false` | you installed a trusted cert on the console |
| `PROTECT_ENABLED` | `true` | `false` if you do not run Protect |
| `PROTECT_BOOTSTRAP` | `true` | `false` to stay on the supported Protect API and give up disk health |
| `RETENTION` | `13w` | how long VictoriaMetrics keeps history |
| `WAN_ENABLED` | `true` | `false` if there is no WAN here worth watching |
| `WAN_POLL_INTERVAL` | `60s` | paces the v2 endpoints only; floor `30s` |
| `WAN_DASHBOARD_SCALE` | `24h` | `7d` asks the controller for its widest window |
| `SPEEDTEST_INTERVAL` | `15m` | how often speedtest history is re-read |
| `BACKFILL_ON_START` | `true` | `false` to start recording forward only |
| `LEDGER_RETENTION` | `8760h` | how long the outage archive keeps events |
| `PROBE_TARGETS` | empty | set it to turn the demarcation probe on, see below |
| `PROBE_INTERVAL` | `60s` | floor `15s` |
| `PROBE_COUNT` | `10` | echoes per target per round |
| `PROBE_TIMEOUT` | `2s` | per echo |

### Hypervisors and storage

Each system goes in a slot, `SOURCE_1` to `SOURCE_8`, as one line: the kind,
its address, then optional `name=`, `user=`, `every=` and `tls=verify`. The
credential goes in the matching `_SECRET` variable, so it never appears in the
line the Settings page shows:

```
SOURCE_1=proxmox https://pve1.lab:8006 name=pve-lab user=telemetry@pve!ro
SOURCE_1_SECRET=<API token secret>
```

Kinds: `proxmox`, `pbs`, `vmware`, `synology`, `qnap`, `truenas`. The
dashboard's **Settings → Add a source** generates both lines and says what
read-only account to create on that system.

Fixed slot names are deliberate: Portainer passes a variable into the
container only when `docker-compose.yml` names it, so a free-form name would
never arrive.

Every kind is built. Each is polled every 60 seconds unless `every=` says
otherwise (30 seconds at the least), and a self-signed certificate is
accepted unless the slot says `tls=verify`.

Where a system keeps its own alarms (vCenter, TrueNAS), they are the alerts
and Ferrule adds no thresholds beside them. Where it does not (Proxmox VE,
PBS, Synology, QNAP), Ferrule raises a few: storage at 90% full is a
warning and at 95% is bad, and anything the system itself calls degraded,
crashed, failed or offline. Alerts already true when the collector starts
are never sent as notifications.

#### VMware vCenter

vCenter 7 or 8, standalone ESXi hosts included through it. In vCenter:

1. **Administration → Single Sign On → Users and Groups**: add a user in
   `vsphere.local`, such as `telemetry`.
2. **Administration → Access Control → Global Permissions**: add it with the
   built-in **Read-only** role, and tick **Propagate to children**.

```
SOURCE_1=vmware https://vcsa.example.com name=vcsa user=telemetry@vsphere.local
SOURCE_1_SECRET=<its password>
```

Add `tls=verify` once vCenter has a certificate the container trusts; by
default the certificate is not checked, as most vCenters use their own CA.

What it reads: every host's connection, maintenance mode, CPU, memory, uptime
and ESXi build; every datastore's size, free space and reachability; every
machine's power state, and CPU and memory for the busiest fifteen; and
vCenter's triggered alarms. Templates are not counted as machines.

Alerts come from vCenter itself: each triggered alarm that nobody has
acknowledged is a `vcenter_alarm` alert, red as critical and yellow as a
warning, and notifies like any other. Ferrule adds no thresholds of its own;
vCenter's defaults already watch host connection, CPU, memory and datastore
usage, and you tune them there. **Acknowledging an alarm in vCenter clears it
in Ferrule.** Alarms already open when the collector starts are not sent as
notifications.

#### Proxmox VE, with Ceph

One slot per cluster; any node's address will do. In the web UI, under
**Datacenter → Permissions**:

1. **Users**: add `telemetry` in the `pve` realm (Proxmox VE authentication
   server). It needs no password.
2. **API Tokens**: add a token for it with the ID `ro`, **Privilege
   Separation** ticked. Copy the secret; it is shown once.
3. **Permissions → Add → User Permission**: path `/`, the user
   `telemetry@pve`, role **PVEAuditor**, **Propagate** ticked.
4. **Permissions → Add → API Token Permission**: path `/`, the token
   `telemetry@pve!ro`, role **PVEAuditor**, **Propagate** ticked.

Both are needed: with privilege separation on, a token sees only what both
it and its user may see. With only the token's, Proxmox VE answers with
empty lists rather than an error, and the panel says so.

```
SOURCE_2=proxmox https://pve1.lab:8006 name=pve-lab user=telemetry@pve!ro
SOURCE_2_SECRET=<the token secret>
```

What it reads: every node's state, CPU, memory, uptime and version; quorum;
every storage's use and availability (shared storage once, not once per
node); every guest's state, and CPU and memory for the busiest fifteen; and
Ceph, when the cluster runs it: health, each health check, OSDs and
placement groups. PVEAuditor covers Ceph as well.

Alerts: a node that has left the cluster (`host_offline`), quorum lost,
each Ceph health check Ceph raises (`ceph_health`; HEALTH_WARN is a
warning, HEALTH_ERR is bad; **a check muted in Ceph stays quiet here**), and
storage unavailable or nearly full.

#### Proxmox Backup Server

Under **Configuration → Access Control**:

1. **User Management**: add `telemetry` in the `pbs` realm.
2. **API Token**: add a token for it named `ro`. Copy the secret.
3. **Permissions → Add → User Permission**: path `/`, the user
   `telemetry@pbs`, role **Audit**, **Propagate** ticked.
4. **Permissions → Add → API Token Permission**: path `/`, the token
   `telemetry@pbs!ro`, role **Audit**, **Propagate** ticked.

Both are needed, as for Proxmox VE: with only the token's, PBS shows no
datastores, jobs or groups, and the panel says how to fix it.

```
SOURCE_5=pbs https://pbs1.lab:8007 name=pbs1 user=telemetry@pbs!ro
SOURCE_5_SECRET=<the token secret>
```

PBS separates the token and its secret with a colon where Proxmox VE uses an
equals sign; Ferrule handles that. What it reads: every datastore's use and
PBS's own estimate of when it fills; the latest run of every job in the
last week (backups, verify, prune, sync, garbage collection); and each
backup group's last backup, oldest first.

Alerts: a job whose latest run failed is bad, one that finished with
warnings is a warning (`backup_task`); either clears when that job next
runs clean. A datastore PBS cannot read, or one nearly full. A group that
has not backed up in a while is highlighted on the page (26 hours by
default, under Settings → Highlighting) but not alerted on, because a guest
that was deleted keeps its old group forever.

#### Synology DSM

In **Control Panel → User & Group**:

1. Add `telemetry` with a long random password, and put it in the
   **administrators** group: DSM shows disk and volume health only to
   administrators.
2. Under **Applications**, deny it everything; under **Shared Folders**, give
   it no access.
3. Leave **2-step verification** off for this account. The API has no way to
   answer it, and DSM will say so.

```
SOURCE_6=synology https://synology.home:5001 name=ds920 user=telemetry
SOURCE_6_SECRET=<its password>
```

What it reads: model, DSM version, uptime, CPU, memory, the temperature and
DSM's own verdict on it; every volume's status, RAID type and use; every
drive's status, SMART verdict and temperature. Alerts: a volume degraded or
crashed, a drive DSM flags or whose SMART has failed, a volume nearly full,
and DSM saying the box is hot.

#### QNAP, over SNMPv3

QTS and QuTS hero have no read-only API account, so Ferrule reads QNAP's own
SNMP MIB, over SNMPv3 with authentication and encryption. In **Control Panel
→ Network & File Services → SNMP**:

1. **Enable SNMP service**, version **SNMPv3** only.
2. User `telemetry`, authentication **HMAC-SHA**, privacy **CBC-DES** (the
   only cipher QNAP offers), each with a passphrase of at least 8 characters.

```
SOURCE_7=qnap nas.lab name=ts453 user=telemetry
SOURCE_7_SECRET=<auth passphrase>:<privacy passphrase>
```

The address is the host name or IP, with no `https://`; add `:port` if it is
not 161. The collector must reach it on **UDP** 161. The secret is both
passphrases joined by the first colon; one passphrase with no colon means
authentication without encryption, which QNAP allows but is not
recommended.

The slot assumes SHA and DES, which is what QNAP offers. If you chose
HMAC-MD5 on the NAS, add `auth=md5` to the slot line; `priv=aes` is there for
an agent that has AES. DES is a 56-bit cipher and weak against a determined
attacker; it is still far better than SNMPv1 or v2c, which send everything,
community string included, in clear. Keep SNMP on the management network.

What it reads: model, firmware, uptime, CPU, memory, temperatures, fans;
every disk's (and M.2 drive's) status and temperature; every storage pool's
status and use, and every RAID group's status. QNAP has two MIBs: current
QTS and QuTS hero fill the newer one (QTS-MIB) and leave model, uptime and
memory out of the older (NAS-MIB). Ferrule reads both and takes the newer
wherever it answers. Alerts as for Synology.

#### TrueNAS

SCALE or CORE. In **Credentials → Users**, add `telemetry` with the
**Read-only Admin** role, then create an **API key** for it.

```
SOURCE_4=truenas https://truenas.home name=tank
SOURCE_4_SECRET=<the API key>
```

Leave `user=` off to use the key; set it to use a password instead. What it
reads: model, version, uptime, memory; every pool's status, layout and use;
every disk's state, ZFS read, write and checksum errors, model and
temperature; and TrueNAS's alert list. Like vCenter, TrueNAS's own alerts
are the alerts (`nas_alert`): each one not dismissed, WARNING as a warning
and ERROR or worse as bad. **Dismissing it in TrueNAS clears it here.**

TrueNAS 25.04 marked the REST API this uses as deprecated, in favour of
JSON-RPC over a WebSocket. It still answers through 25.10; if a later
release drops it, the slot will say so.

### Inventory reports

With a vCenter or Proxmox VE slot, the Virtualization tab has an
**Inventory** panel: every machine, disk, network card, guest partition,
snapshot, host, datastore and cluster, downloadable as **Excel**, a zip of
**CSV** files (one per sheet), or a single **HTML** page that needs nothing
else to open. The sheets follow RVTools where the two hypervisors agree, and
vCenter's custom attributes (Cost Center, Owner and the like) become columns
of their own. The read-only account each slot already uses is enough.

Each inventory is compared with the one before it, and the **Changes** sheet
lists what was added, removed or reconfigured: memory, CPUs, disks resized
or moved, networks, folders, notes, attributes. Readings that move on their
own (power state, usage, which host DRS chose, the guest's IP addresses) are
not changes. A source that could not be read is left out of the comparison,
so an unreachable vCenter does not read as every machine removed.

Taking an inventory by hand and downloading the newest is free. The daily
run, the history of runs, the Changes sheet, the date-range report and the
database copy are paid. Runs are kept on the archive volume, one compressed
file each, and travel with archive backups. License keys are never read.

**What changed between two dates.** Pick a From and To date on the panel and
download one report: the net difference between the inventory in force when
the range began and the newest inside it, every change each inventory in
between recorded (so a machine added and removed inside the range still
shows), and the inventories it read, with their fingerprints.

**Evidence that history was not changed.** Each inventory records a SHA-256
fingerprint of itself and of the one before, so the runs form a chain:
editing or deleting a run breaks every fingerprint after it. The Sources
sheet shows both. Ferrule checks the whole chain when it starts and after
every run, and raises **Inventory history changed** if anything fails. A
run that `INVENTORY_KEEP` expires is marked expired, not deleted, so the
chain stays whole. Because whoever can change the files could rebuild a
chain kept beside them, each fingerprint is also written to every backup
target as `fingerprint-<run>.json` (never pruned) and to the database copy,
which Ferrule checks against its files. For the strongest evidence, keep the
backup share under a snapshot schedule the stack's credentials cannot
delete (Synology and QNAP immutable snapshots, TrueNAS periodic snapshots),
or use S3 with Object Lock.

**A copy in PostgreSQL**, for Grafana, Power BI or anything else with a
PostgreSQL connector, no plugin needed. Every run goes into schema
`ferrule`: a `runs` table, one table per sheet (`vms`, `disks`, `nics`,
`partitions`, `snapshots`, `hosts`, `datastores`, `clusters`, `health`,
`sources`, `changes`), `vm_attributes` for vCenter's custom attributes, and a
`latest_` view of each sheet for the fleet as the newest run saw it. Every
row carries `run_id` and `taken`. It is a copy: the files stay the master,
runs taken before the database was connected are copied in on the first
sync, and the copy can be dropped and rebuilt from the files at any time.

For the stack's own database, add `postgres` to `COMPOSE_PROFILES` and set
`INVENTORY_DB_PASSWORD`; nothing else is needed. To change the password
later, change `INVENTORY_DB_PASSWORD` and redeploy the whole stack: the
database container sets the new one within a minute. (Postgres itself
keeps the password it was first made with; the stack's health check puts
the new one in.) Should that ever not take, removing the stack's
`postgres-data` volume loses nothing: the database is a copy, and Ferrule
copies every run back in. To use a PostgreSQL you
run, set `INVENTORY_DB_URL` instead. Set `INVENTORY_DB_READER_PASSWORD`, at
least 12 characters, and Ferrule makes a read-only `ferrule_reader` role for
reporting tools to sign in as, so they never hold the writer's password. A
shorter one is refused: the role is not made, the inventory panel says why,
and everything else carries on.

| Variable | Default | |
|---|---|---|
| `INVENTORY` | `on` | `off` stops the daily run; taking one by hand still works |
| `INVENTORY_AT` | `02:30` | when the daily run is taken, in `TZ` |
| `INVENTORY_KEEP` | `365` | days of runs to keep, in Ferrule's files and the database copy alike; `2557` keeps seven years, as SOX asks. The newest two are always kept |
| `INVENTORY_DB_URL` | empty | `postgres://user:password@host:5432/database?sslmode=verify-full` for a PostgreSQL you run |
| `INVENTORY_DB_PASSWORD` | empty | the writer's password; on its own, the stack's `postgres` profile is used |
| `INVENTORY_DB_READER_PASSWORD` | empty | at least 12 characters; makes or updates `ferrule_reader`, sent as a SCRAM verifier, never as text |
| `INVENTORY_DB_CA` | empty | your CA certificate, PEM, for a server whose certificate it signed; turns on `verify-full` |
| `INVENTORY_DB_BIND` | `127.0.0.1` | where the bundled database listens on the Docker host; `0.0.0.0` for Power BI on another machine |
| `INVENTORY_DB_PORT` | `5432` | the bundled database's port on the Docker host |

What Proxmox VE cannot tell a read-only token is left out rather than asked
for: a VM's IP addresses, partitions and guest OS need the QEMU guest agent
and a privilege PVEAuditor does not have. A container's static address is in
its config, and is shown.

### The assistant (experimental, paid)

Off until a model is set. With one, each alert in Needs attention has
**Explain** and each source panel **Review**. Either opens a panel that
shows which model will answer, exactly what will be sent, and its size
against that model's context, before anything goes. The model's findings
come back as cards: severity, what is wrong, steps, and the readings that
support each one. It advises only: it cannot change anything.

What it sends: the readings and alerts for that source or alert, and, if
ticked, the last 7 days of the archive about it. Never a password, key or
secret. With **Replace names, addresses and serials** ticked (the default
for a model outside your network), those are swapped for stand-ins such as
`SOURCE-1` before sending and put back in the answer. Every question goes in
the audit log, naming the model, and every answer is kept in the archive
with the model that gave it.

**Recommend fixes**, when ticked, asks for a fix for each problem as well:
steps in order, what carrying them out could disrupt, and how to check it
worked. Each shows on its finding as a suggested fix, to check before
running. It is advice: Ferrule runs nothing, and the model is told to give
an exact command or menu path only where it is sure of it.

The assistant is paid, whichever model answers: the trial or a license. When
neither is in force, Explain and Review say so, and answers already given
stay readable.

**Up to four models**, one per slot, `AI_1` to `AI_4`, and whoever asks
picks one in the panel. Each slot is one line: the provider, then the address
of a model you run, then optional `name=`, `model=`, `context=` and
`timeout=`. A hosted model's key goes in the slot's own key variable,
`AI_1_KEY` to `AI_4_KEY`.
**Settings → Add a model** writes the lines.

```
AI_1=llamacpp http://llm-box.lan:8080/v1 name=GPU-box
AI_2=claude name=Claude
AI_2_KEY=<from console.anthropic.com>
```

| In the slot line | Default | |
|---|---|---|
| provider | | a hosted model: `claude`, `openai` (ChatGPT), `gemini` or `xai` (Grok). One you run: `llamacpp`, `lmstudio`, `ollama`, or `local` for any other OpenAI-compatible server |
| address | the provider's own | needed only for a model you run, such as `http://llm-box.lan:8080/v1` for llama.cpp, `:1234/v1` for LM Studio, `:11434/v1` for Ollama |
| `name=` | the provider | what the panel calls it |
| `model=` | `claude-opus-5-5` for Claude | the model's name as the provider lists it. Needed for OpenAI, Gemini, xAI, LM Studio and Ollama; a name the provider does not know is refused with the list of names your key can use |
| `context=` | `32768` local, `128000` OpenAI, Gemini and xAI, `1000000` Claude | the context window in tokens. A question that will not fit is refused before it is sent. llama.cpp reports its own |
| `timeout=` | `AI_TIMEOUT`, `5m` | how long to wait for an answer |

Who may ask: anyone with the operator role may ask a model on your network.
A model **outside** your network is for admins only; an operator sees it in
the list, marked so. Whether a model is outside is decided by its address
alone: a private or loopback address, or a host name with no public suffix
(or ending `.lan`, `.local`, `.home` and the like), is on your network, and
anything else is not, with no setting to say otherwise. With sign-in off
there are no roles, as for everything else.

Each model takes one question at a time; asking one that is busy says so at
once, and another model can be picked. Every five minutes Ferrule asks each
model for its list of models, which costs nothing, and the panel shows
whether it answered, refused its key, or could not be reached. There is no
automatic switch to another model: a question goes only where its preview
said it would.

**From before slots:** `AI_PROVIDER`, `AI_URL`, `AI_MODEL`, `AI_API_KEY` and
`AI_CONTEXT` still work, as slot 1, when `AI_1` is not set, and `AI_2` to
`AI_4` work beside them. With `AI_1` set they are ignored, and Settings says
so. In **Settings → Stack environment**, each slot is a provider list and
its fields, and **Move to AI_1**, under the old single-model settings,
writes the old variables as an `AI_1` line and clears them; paste the key
into `AI_1_KEY`, since Ferrule never sees it. A slot that cannot be read is shown
on Settings with the reason and never stops the stack.

**LM Studio:** in the Developer tab, start the server and turn on **Serve on
local network**. Load the model with a context of 16K or more (LM Studio
often defaults to 4K, which is too small), and set `context=` to match.
The first question reads the whole case file before it answers; on a
desktop machine that is usually 20 to 60 seconds.

**llama.cpp:** start `llama-server` with `--host 0.0.0.0` so it listens on
the network, and `--jinja` for the model's own chat template. Ferrule asks
it for its settings at `/props`, so the panel shows "llama.cpp" with the
loaded model's name, and the context each question gets, which is `-c`
divided by the server's parallel slots (`-np`); `context=` is then not
needed. With `-np 4` and `-c 32768` a question gets 8K, so run `-np 1` if
the server is only for this. `model=` is not needed.

```
llama-server -m model.gguf --host 0.0.0.0 --port 8080 -c 32768 -np 1 --jinja
AI_1=llamacpp http://llm-box.lan:8080/v1
```

**Hosted models:** Claude, OpenAI, Gemini and xAI replace names, addresses
and serials by default. Claude is asked through its own API, with its
instructions cached between questions and a fallback model if it declines;
the panel shows each question's cost. OpenAI, Gemini and xAI are asked
through their OpenAI-compatible endpoints, which bill by their own price
lists.

```
AI_3=gemini model=<a model name AI Studio lists>
AI_3_KEY=<from Google AI Studio>
```

### License

| Name | Default | Meaning |
|---|---|---|
| `FERRULE_LICENSE` | empty | a license key, one long line starting `FERRULE1.`. Optional: it can be pasted on Settings instead |

Without a key, a new install runs a **30-day trial** of the paid features,
counted from its first start and remembered on the `ledger-data` volume,
then carries on as **Community**: free for personal use, with everything for
one site. Settings → License shows the edition, the days left and what is
paid. Whatever happens to a license, monitoring and alerts never stop: only
the paid features pause.

**Online confirmation.** A key is checked by its signature, on your host.
Ferrule also fetches a signed list of withdrawn keys once a day, from
`https://license.ferrulehq.dev/revoked`, so a key that leaks can be
withdrawn. The fetch is a plain download: nothing about you, your network
or your install is sent. A key that has not been
confirmed for 30 days pauses the paid features until the list can be
reached again, and Settings → License shows how long is left, with a
warning a week before. For a site that cannot reach the internet, ask for
an **offline key**: it never checks in, and runs for up to a year. The
trial and Community have no key, so they never check.

**Entering a key without Portainer.** An admin can paste a key on Settings →
License and press Apply. It is checked there and then, used at once with no
restart, and kept on the archive volume as `license.json`, so it travels with
archive backups and comes back with a restore. **Remove license** takes it
off again. Each change goes in the audit log. When `FERRULE_LICENSE` is set,
it always wins, and the box on Settings is locked.

### Sign-in

Off until you turn it on, so a dashboard behind a VPN or Cloudflare Access
keeps working as it is. Settings → Access shows what is on.

**Local admin, free in every edition:**

| Name | Default | Meaning |
|---|---|---|
| `ADMIN_PASSWORD` | empty | turns sign-in on; at least 12 characters |
| `ADMIN_USERNAME` | `admin` | |
| `SESSION_TTL` | `168h` | how long a sign-in lasts |
| `METRICS_TOKEN` | empty | lets a Prometheus scraper read `/metrics` with `Authorization: Bearer <token>`; otherwise `/metrics` needs a session once sign-in is on |

Changing `ADMIN_PASSWORD` signs everyone out. Five wrong passwords in fifteen
minutes lock that username, and that address, for five minutes. `/healthz`
never needs a session, so the container health check keeps working.

**Local users and groups, paid:** for a shop with no directory. Sign in as
the admin above, then add people under **Settings → Users and groups**. Each
user has a role of their own, and groups that give a role; they get the
highest. A change applies at once, even to someone signed in: disabling a
user or giving them a new password signs them out everywhere. Local users
can change their own password on Settings.

| Name | Default | Meaning |
|---|---|---|
| `SETTINGS_KEY` | empty | needed for local users: 32 random bytes in base64 (`openssl rand -base64 32`, or Settings → Users and groups → Make a key). Users are kept in `settings.json` on the archive volume, encrypted with it, so a copied volume or backup holds no names or password hashes. **Keep a copy of the key outside the stack**: without it the users cannot be read back |

Users travel with archive backups, still encrypted, and **Export settings**
downloads them as one file to import into another install with the same key.
`ADMIN_USERNAME` always stays the environment's admin; no local user may take
its name. Without `ADMIN_PASSWORD`, Ferrule refuses any change that would
leave no local admin. Adding users and groups needs a license; when one
lapses, everyone already added keeps signing in, and can still be disabled,
deleted or given a new password.

**Audit log:** every sign-in, failed sign-in and sign-out, and every change
to users, groups and settings, with the address it came from, is kept in the
archive beside outages and alerts, and goes with it into backups. Admins see
it on Settings and can download it as CSV for an auditor.

**Single sign-on, paid:** OIDC and LDAP, with three roles. A viewer sees
everything; an operator can also send test notifications; an admin can do
everything. Who gets which role is a list in `OIDC_ROLES` or `LDAP_ROLES`:
entries separated by `;`, each `who:role`. `who` is a group (a name, an ID,
or an LDAP DN), an email address, `@domain` for every verified address at
that domain, or `*` for anyone the provider signs in. Someone who matches
nothing is refused, so pointing OIDC at Google does not let in every Google
account.

```
OIDC_ROLES=ferrule-admins:admin; it-team:operator; @example.com:viewer
LDAP_ROLES=CN=Ferrule Admins,OU=Groups,DC=corp,DC=example:admin; CN=Helpdesk,OU=Groups,DC=corp,DC=example:viewer
```

| Name | Meaning |
|---|---|
| `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `OIDC_ROLES` | required together. Register Ferrule with your provider as a web application with the redirect URI `<DASHBOARD_URL>/auth/oidc/callback` |
| `OIDC_NAME` | the button's label: "Sign in with …" |
| `OIDC_GROUPS_CLAIM` | where the provider puts groups; default `groups`. Entra ID sends group object IDs there once "groups claim" is turned on for the app |
| `OIDC_SCOPES` | default `openid profile email`; some providers need `groups` added |
| `OIDC_REDIRECT_URL` | only if it is not `DASHBOARD_URL` plus `/auth/oidc/callback` |
| `LDAP_URL` | `ldaps://dc1.corp.example`, or `ldap://` with `LDAP_STARTTLS=true`. Plain `ldap://` is refused: it would send passwords unencrypted |
| `LDAP_BIND_DN`, `LDAP_BIND_PASSWORD` | a read-only service account that can search for users |
| `LDAP_BASE_DN` | where to search, e.g. `DC=corp,DC=example` |
| `LDAP_ROLES` | required: group DNs to roles, from each user's `memberOf` |
| `LDAP_USER_FILTER` | default `(\|(uid={user})(sAMAccountName={user})(userPrincipalName={user}))`, which covers Active Directory and OpenLDAP |
| `LDAP_GROUP_ATTR` | default `memberOf`. Direct membership only; nested groups are not followed |
| `LDAP_CA_CERT` | your internal CA's certificate, as PEM text, when the directory's certificate is not publicly trusted |
| `LDAP_VERIFY_TLS` | `true`; set `false` only to try things out |

If the license or trial ends, single sign-on stops but the dashboard does not
open up: the local admin still works, and the login page says why. Keep an
`ADMIN_PASSWORD` set as the way in that never depends on a license, or a
directory being up.

SAML is not supported. A SAML-only provider can sit behind Keycloak or
Authentik, which speak OIDC.

### Notifications

Any number of channels can be on at once; each is off until its variables
are set, and setting only some of a channel's required variables stops the
collector at startup with the reason. The dashboard's **Settings →
Notifications** shows every channel's state, tests each one, lets you pick
what notifies per tab and per alert type, and writes the lines to set here.

| Channel | Variables | Notes |
|---|---|---|
| Gotify | `GOTIFY_URL`, `GOTIFY_TOKEN` | your Gotify, or `http://gotify` for the bundled one; an application token |
| ntfy | `NTFY_URL`, optional `NTFY_TOKEN` | the topic's full URL, e.g. `https://ntfy.sh/my-ferrule-alerts`; on ntfy.sh, pick a topic nobody could guess |
| Pushover | `PUSHOVER_TOKEN`, `PUSHOVER_USER` | an application token, and your user or group key |
| Discord | `DISCORD_WEBHOOK_URL` | Server settings → Integrations → Webhooks |
| Email | `SMTP_HOST`, `SMTP_FROM`, `SMTP_TO`; optional `SMTP_PORT` (587), `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_TLS` | `SMTP_TO` takes several addresses, comma separated. `SMTP_TLS` is `starttls` (the default), `tls` (the default on port 465) or `none`; a password is never sent unencrypted |
| Slack *(paid)* | `SLACK_WEBHOOK_URL` | an incoming webhook from a Slack app |
| Microsoft Teams *(paid)* | `TEAMS_WEBHOOK_URL` | a **Workflows** webhook: in the channel, Workflows → "Post to a channel when a webhook request is received". The old Office 365 connector webhooks were retired in 2026 and do not work |
| PagerDuty *(paid)* | `PAGERDUTY_ROUTING_KEY` | an Events API v2 integration key. Critical alerts only; a clear resolves the incident |
| Apprise *(paid)* | `APPRISE_URL`, and `APPRISE_URLS` or `APPRISE_KEY` | an apprise-api server, such as the bundled one (`http://apprise:8000`), reaching more than a hundred services. `APPRISE_URLS` lists them as Apprise URLs and travels with each message, so the server keeps nothing; `APPRISE_KEY` names a configuration stored on the server instead |
| Webhook *(paid)* | `WEBHOOK_URL`, optional `WEBHOOK_SECRET` | JSON for Home Assistant, n8n or anything else; with a secret, `X-Ferrule-Signature` carries `sha256=` and the HMAC of the body |

Paid channels send during the trial or with a license. When neither is in
force they send one message saying they have paused, PagerDuty resolves what
it opened, and the free channels carry on.

What notifies, and when, is shared by every channel:

| Name | Default | Meaning |
|---|---|---|
| `DASHBOARD_URL` | empty | where tapping a notification goes, e.g. `https://ferrule.example.com` |
| `NOTIFY_DEFAULT` | `bad` | lowest severity that notifies: `bad`, `warn` or `off` |
| `NOTIFY_RULES` | empty | overrides by tab or alert type, e.g. `wan=warn port_slow=off` |
| `NOTIFY_DELAY` | `2m` | only send if still active this long later |
| `NOTIFY_ON_CLEAR` | `true` | also send when an alert clears |
| `NOTIFY_REPEAT` | `off` | re-send a still-active critical alert, e.g. `4h` |
| `NOTIFY_QUIET` | empty | e.g. `22:00-07:00`: only critical gets through; warnings arrive as one summary after |
| `TZ` | UTC | the zone quiet hours are read in, e.g. `America/Los_Angeles` |

A rule naming an unknown alert type or tab stops the collector at startup
with the reason, so a typo cannot silently mute anything.

#### Gotify and iGotify in this stack

**Already running Gotify and iGotify?** Leave `COMPOSE_PROFILES` unset and set
`GOTIFY_URL` to your server and `GOTIFY_TOKEN` to a new application's token.

**Starting from nothing?** The stack can run them too, each behind its own
profile so neither starts unless asked:

```
COMPOSE_PROFILES=gotify,igotify     # or just igotify, beside an existing Gotify
GOTIFY_ADMIN_USER=admin             # the default
GOTIFY_ADMIN_PASSWORD=<choose one>  # unset, Gotify's admin password is "admin"
GOTIFY_URL=http://gotify
GOTIFY_PORT=8091                    # host ports, for a tunnel or proxy to use
IGOTIFY_PORT=8092
```

Deploy once, open Gotify, create an application, put its token in
`GOTIFY_TOKEN`, and update the stack. iGotify relays to the iPhone app
through Apple's push service; the phone has to reach Gotify and iGotify, so
away from home put them behind a tunnel or reverse proxy. `IGOTIFY_GOTIFY_URLS`,
`IGOTIFY_CLIENT_TOKENS` and `IGOTIFY_SECNTFY_TOKENS` are only needed when
they are not reachable over public domains, and then all three.

Whether your Portainer passes `COMPOSE_PROFILES` through from the stack's
variables depends on its version. If the services do not start, run Gotify
and iGotify as their own small stack and point `GOTIFY_URL` at it.

#### Apprise in this stack

Add `apprise` to `COMPOSE_PROFILES` and the stack runs apprise-api as
`ferrule-apprise`, on host port `APPRISE_PORT` (8093). Then:

```
APPRISE_URL=http://apprise:8000
APPRISE_URLS=tgram://bottoken/chatid mailto://user:pass@example.com
```

Separate several services with spaces. Each service's URL format is in
Apprise's own documentation.

### Backups

The archive (every outage, alert and event the controller forgets, and when
the trial began) is backed up **daily**, and the history in VictoriaMetrics
**weekly**, to a share, to S3-compatible storage, or both. A **fresh
container with an empty archive restores the newest backup by itself**,
before it starts: rebuild the stack anywhere, point it at the same backups,
and it comes back as it was. History is re-imported into a VictoriaMetrics
that holds nothing older than the backup. Restoring is never paid, even from
S3 after a license ends. Settings → Backups shows each target, what is
there, and a "Back up now" button.

| Name | Default | Meaning |
|---|---|---|
| `BACKUP_NAME` | `ferrule` | names this site's backups. Give each site its own when several share a target; a rebuilt site must keep its name to find its backups |
| `BACKUP_KEY` | empty | encrypts backups (AES-256-GCM). 32 random bytes in base64: `openssl rand -base64 32`, or Settings → Backups → Make a key. **Keep a copy outside the stack**: without it an encrypted backup cannot be restored |
| `BACKUP_AT` | `03:30` | the time of day, in `TZ` |
| `BACKUP_HISTORY` | `true` | also back up history, weekly |
| `BACKUP_HISTORY_DAY` | `sunday` | which day |
| `BACKUP_KEEP` | `14` | archive backups kept per target |
| `BACKUP_HISTORY_KEEP` | `4` | history backups kept per target |
| `BACKUP_RESTORE` | `auto` | `off` to never restore by itself |

A backup missed while the stack was down is taken a couple of minutes after
it comes back. A failed backup raises a "Backup failed" warning, which
notifies like any other alert (`NOTIFY_RULES` tab `settings`).

#### To a share (NFS or SMB), free

Docker mounts the share as a named volume, described by an overlay file, so
the stack still has no bind mounts and no host paths. In Portainer, open the
stack's **Git** settings and add one file under **Additional paths**, beside
`docker-compose.yml`:

| Share | Overlay | Variables |
|---|---|---|
| NFS | `docker-compose.backup-nfs.yml` | `BACKUP_NFS_SERVER`, `BACKUP_NFS_PATH`, optional `BACKUP_NFS_OPTIONS` (default `nfsvers=4.1,rw,soft,timeo=150`) |
| SMB | `docker-compose.backup-smb.yml` | `BACKUP_SMB_SERVER`, `BACKUP_SMB_SHARE`, `BACKUP_SMB_USERNAME`, `BACKUP_SMB_PASSWORD`, optional `BACKUP_SMB_VERSION` (`3.0`) |

```
BACKUP_NFS_SERVER=192.168.1.20
BACKUP_NFS_PATH=/volume1/backups
```

Backups land in `<share>/<BACKUP_NAME>/`. Things to know:

- The Docker host needs the client: `nfs-common` or `cifs-utils` on Debian
  and Ubuntu. Most NAS and server installs have both.
- The collector writes as UID/GID 65534. For NFS, the export must allow it:
  on a Synology, set the NFS rule's squash to "Map all users to admin";
  elsewhere use `all_squash,anonuid=65534,anongid=65534`, or own the
  directory by 65534. The SMB overlay mounts the share as 65534 already.
- An SMB password cannot contain a comma.
- Docker reads the share's options only when it creates the volume. To
  change them, such as a new password, stop the stack, remove its
  `backup-target` volume (**Volumes** in Portainer; your backups are on the
  share, not in the volume), and redeploy.
- If your Portainer has no Additional paths field, merge the overlay's
  `volumes` and `environment` into `docker-compose.yml` in your fork.

#### To S3-compatible storage, paid

Wasabi, Backblaze B2, Cloudflare R2, MinIO and AWS all work. No overlay is
needed:

```
BACKUP_S3_ENDPOINT=s3.us-east-1.wasabisys.com
BACKUP_S3_REGION=us-east-1
BACKUP_S3_BUCKET=my-ferrule-backups
BACKUP_S3_ACCESS_KEY=...
BACKUP_S3_SECRET_KEY=...
```

Backups land in `<bucket>/<BACKUP_S3_PREFIX>/<BACKUP_NAME>/`, with the
prefix `ferrule` by default. Make the bucket first, and give the key access
to that bucket only: list, read, write and delete. For Wasabi, the endpoint
and region must match the bucket's region; Backblaze B2 shows its S3 endpoint
on the bucket's page. Without a license or trial, S3 backups pause and the
share carries on; restoring from S3 keeps working. Set `BACKUP_KEY` too:
these backups leave your network.

### Reaching it from outside: Cloudflare Tunnel

Ferrule does not ship a tunnel. If you already run `cloudflared`, add public
hostnames to that tunnel in the Cloudflare dashboard (Zero Trust → Networks →
Tunnels → your tunnel → Public Hostname):

| Hostname | Service |
|---|---|
| `ferrule.example.com` | `http://<docker host>:8090` |
| `gotify.example.com` | `http://<docker host>:8091`, if you run the bundled Gotify |
| `igotify.example.com` | `http://<docker host>:8092`, if you run the bundled iGotify |

If `cloudflared` runs as a container on the same Docker network, use the
container names and their internal ports instead: `http://ferrule:8080`,
`http://ferrule-gotify:80`, `http://ferrule-igotify:8080`.

Turn on sign-in (above) before the dashboard goes on a tunnel, or put it
behind Cloudflare Access (an Access application with a policy for your own
email), or both. Gotify and
iGotify must stay reachable by the phone apps, which cannot pass an Access
login; Gotify has its own, and iGotify only relays. Then set
`DASHBOARD_URL=https://ferrule.example.com` so tapping a notification opens
it.

### Deploying from someone else's private repo

To deploy a repo you can read but do not own, ask its owner for a read-only
account on their Git server, then create an access token for it there (on
Gitea: Settings → Applications → Generate New Token, with read access to
repositories). In Step 3, turn **Authentication** on and use that account's
username and the token as the password. GitOps updates pull from it like
any other repo; the account cannot change anything.

### HMS-Docker hosts

`docker-compose.hms-docker.yml` puts the dashboard, and Gotify and iGotify
when their profiles are on, behind HMS-Docker's Traefik on
`hms-docker_proxy_net`, with the same `internal-ipallowlist@file` middleware
HMS-Docker uses. Add it under the Git stack's **Additional paths**, and set
your own hostnames:

```
TELEMETRY_HOSTNAME=telemetry.example.com
GOTIFY_HOSTNAME=gotify.example.com
IGOTIFY_HOSTNAME=igotify.example.com
```

`TRAEFIK_MIDDLEWARES` replaces `internal-ipallowlist@file` on all three, for
a host whose Traefik names its middlewares differently.

If your Portainer has no Additional paths field, merge the overlay's
`networks` and `labels` into `docker-compose.yml` in your fork.

### Running beside an existing stack

This compose file names its containers `ferrule` and
`ferrule-victoria-metrics`, on port 8090, so it can run on
the same Docker host as an earlier deployment of this project. Set
`UNIFI_ENABLED=false` on it while you try it, or both stacks will poll the
same controller.

### The demarcation probe

Off until you give it targets. Three roles are meaningful:

```
PROBE_TARGETS=lan_gateway=192.168.1.1,wan_next_hop=203.0.113.1,internet_control=1.1.1.1
```

`wan_next_hop` must be your ISP's first router, written as a literal address.
It sits on a directly connected subnet on the WAN interface, so it stays
reachable even while that WAN has failed over — which is exactly when you need
the measurement, and which a default route would not give you. Clean to your
own gateway while that hop is lossy is the evidence that the fault is upstream
of the demarcation.

Do not point anything at `192.168.100.1`. It is both the DOCSIS modem
management convention and the Starlink dish address, so what it reaches
depends on your network; modem state is read from the controller instead.

The probe opens an unprivileged ICMP socket, which needs the
`net.ipv4.ping_group_range` sysctl the compose file already sets. The
container keeps `cap_drop: ALL`. If the sysctl does not survive your
deployment the dashboard says the probe is unavailable and nothing else
changes.

---

## Step 5 — Turn on GitOps updates

In **GitOps updates**, pick one:

- **Polling** — Portainer checks the repo on an interval (`5m` is a sensible
  floor). Commit and walk away.
- **Webhook** — Portainer gives you a URL; add it to your repo's webhook
  settings and pushes redeploy immediately. This is the true zero-click
  option.

"Re-pull image" can be on or off: `pull_policy: always` on the `telemetry`
service already asks the registry on every deploy.

Select **Deploy the stack**. The first deploy takes a minute while the images
download.

---

## Versions and updates

`FERRULE_VERSION` says which release runs, and nothing changes until you
change it: set the new version in the stack's variables and update the
stack. `qa` is the build under test, and moves: every redeploy picks up
the newest. Releases are built for Intel/AMD and ARM, so the same version
runs on a NAS or a Raspberry Pi.

## Building from source (development only)

For development, or to try a branch before it is released, add
`docker-compose.build.yml` under the stack's Git settings, **Additional
paths**. It builds Ferrule from the repository on the Docker host, tagged
`ferrule-local:latest` so a build never stands in for a pulled release, and
rebuilds on every redeploy. `FERRULE_VERSION` is still required; any value,
such as `dev`, will do. The host then needs to pull `golang:1.24-alpine`
too.

---

## Step 6 — Confirm it works

The stack should show two running containers. Give `telemetry` about 45
seconds to report healthy — that is the healthcheck's start period.

Container logs on a healthy start:

```
ferrule gitops listening on :8080, polling https://192.168.1.1 every 30s, and 2 source(s)
```

Silence after that is correct. The collector logs failures and state
changes only.

Do not try to open a **Console** on the `telemetry` container — the image is
`scratch` and has no shell. Check from the host instead:

```sh
curl -s http://localhost:8090/healthz
```

You want `"network_ok":true` and `"protect_ok":true`.

Then confirm the interesting metrics are populated:

```sh
curl -s http://localhost:8090/metrics | grep -E 'fan_level|sfp_receive_power|protect_disk'
```

Empty SFP results are worth interpreting rather than debugging. Your USW
switches should report optical power. Your UDM or Cloud Gateway SFP+ ports
quite possibly will not, even with a DDM-capable optic seated — a
long-standing Ubiquiti gap, not a mistake in your setup. The port detail
panel says so explicitly when it happens.

Dashboard: `http://<docker-host>:8090`.

---

## From here on

To move to a new release, change `FERRULE_VERSION` and update the stack.
With GitOps updates on, a change to the compose file in the repository
redeploys on its own, with a webhook or within the polling interval, or on
**Pull and redeploy**.

Changing a credential or a setting does not need a commit — edit the stack
variables in Portainer and update the stack.

---

## Troubleshooting

**`set FERRULE_VERSION to the release to run`.** The stack has no
`FERRULE_VERSION`. Add it, such as `FERRULE_VERSION=0.9.0`, and update the
stack.

**`pull access denied` or `unauthorized` pulling the Ferrule image.** Check
`FERRULE_IMAGE`: the image is public, so a pull needs no login, and this
usually means a mistyped address.

**`manifest unknown` pulling the Ferrule image.** No release has that
version. Check `FERRULE_VERSION` for a typo.

**`configuration: UNIFI_HOST is required`.** The stack variables did not
reach the container. Confirm they are saved under the stack, then update it.

**`login rejected: check the credentials, and that the account is local-only
with MFA disabled`.** MFA on the account is the usual cause, followed by a
UniFi cloud account being used instead of a local one. If only Protect is
failing, check you created the account on the **UNVR** rather than reusing
the gateway's — they are separate credential stores.

**`Protect is enabled on ... but has no credentials for it`.** `PROTECT_HOST`
points somewhere other than `UNIFI_HOST`, so the collector will not silently
reuse the gateway's credentials. Add `PROTECT_API_KEY`, or
`PROTECT_USERNAME` and `PROTECT_PASSWORD`.

**The dashboard says the archive is NOT persisted, with `permission denied`.**
The `ledger-data` volume was created before the image shipped its own
`/var/lib/ferrule` directory. Docker creates a mountpoint as root when
the path does not exist in the image, and the collector runs as uid 65534, so
it can never write there. Redeploy on a build that includes the fix, and if the
message survives that, remove the volume so Docker re-seeds it from the image:

```sh
docker compose down
docker volume rm ferrule_ledger-data
docker compose up -d
```

Nothing is lost by removing it in that state — the failure means nothing was
ever written to it. Confirm with `infra_archive_persistent`, which reads 1 when
events are reaching disk.

**`x509: certificate signed by unknown authority`.** `UNIFI_VERIFY_TLS` is
`true` while the console still has its self-signed certificate. Set it back
to `false`.

**Build fails fetching the toolchain** (only with `docker-compose.build.yml`).
The Docker host has no outbound internet, or an internal registry mirror is
in the way.

**Dashboard loads but shows no devices.** Check the logs for a
`poll: network:` line. When the controller is unreachable the last good
reading stays on screen marked stale rather than blanking — that is
deliberate.

**Port detail says history unavailable.** VictoriaMetrics has not
accumulated enough yet. The six-hour view fills in over six hours.
