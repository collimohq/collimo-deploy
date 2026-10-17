# Ferrule

Ferrule is a monitoring dashboard for UniFi networks, hypervisors and
storage. It shows what the vendors' own screens mostly don't: fan speeds,
per-port error counters, SFP optics, PoE draw, temperatures, NVR disk
health, WAN failover and cable modem health. It also keeps its own record
of outages, so the record survives the controller.

This repository is how you deploy it: a Docker Compose stack, run from
Portainer or from the command line. Ferrule runs from a published image,
so nothing is built on your host.

**Full documentation: <https://docs.ferrulehq.dev>**

## What you need

- A Docker host with **Portainer** or **Docker Compose**, and outbound
  internet access to download the images.
- At least one thing to watch:
  - a UniFi console (gateway, and a UNVR for Protect);
  - or Proxmox VE, Proxmox Backup Server, vCenter/ESXi;
  - or a Synology, QNAP or TrueNAS.
- **A license key**, if you're a beta tester; you'll get it from us. Without
  one, Ferrule starts a 30-day trial of the paid features. When the trial
  ends, it keeps monitoring, with the paid features paused.

## Install

1. **Create read-only accounts** on what you'll watch, such as a UniFi API
   key and a local read-only admin. Each system's exact steps are under
   [Connect your gear](https://docs.ferrulehq.dev/connect/unifi-network/).
2. **Get the stack:**
   - **Portainer:** Stacks → Add stack → name `ferrule` → **Repository**,
     with URL `https://github.com/ferrulehq/ferrule-deploy`,
     reference `refs/heads/main`, compose path `docker-compose.yml`, and
     authentication **off**.
   - **Docker Compose:**
     ```sh
     git clone https://github.com/ferrulehq/ferrule-deploy.git ferrule
     cd ferrule
     cp stack.env.example .env
     ```
3. **Fill in the settings** from [`stack.env.example`](stack.env.example):
   your hosts and credentials. Keep `FERRULE_VERSION` as it is: it's the
   current release. Set `UNIFI_ENABLED=false` if you have no UniFi console.
   - **Portainer:** paste them under **Environment variables** (Advanced
     mode).
   - **Docker Compose:** edit `.env`.
4. **Start it:** **Deploy the stack** in Portainer, or `docker compose up -d`.
   Then open `http://<docker host>:8090`.
5. **Settings → License:** paste your key and press **Apply**. It shows
   "Waiting for the first check-in" for up to about two hours, then
   "Confirmed online".

Once it's running, **Settings → Stack environment** writes the lines for
everything else: hypervisors, storage, notifications and sign-in.

## Updating

When a new release comes out, set `FERRULE_VERSION` to it, then click
**Pull and redeploy** in Portainer, or run `docker compose up -d`. Nothing
changes until you do.

## Help and feedback

Everything is at <https://docs.ferrulehq.dev>, including
[Troubleshooting](https://docs.ferrulehq.dev/troubleshooting/).
[PORTAINER.md](PORTAINER.md) has the full reference too.

Something unclear, broken or missing? Tell us at
<feedback@ferrulehq.dev>.

Your license key is checked once a day against a signed list at
`https://license.ferrulehq.dev/revoked`. That's a plain download: nothing
about you, your network or your install is sent.

---

Ferrule is an independent project, not affiliated with Ubiquiti or the
other vendors it names. See [LICENSE](LICENSE) for the terms.
