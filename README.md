# Ferrule

Ferrule is a self-hosted monitoring dashboard for UniFi networks and the
hypervisors and storage beside them. It shows what the vendors' own screens
mostly don't: per-port errors, optics, fans, temperatures, PoE, NVR disk
health, WAN failover and cable modem health, and it keeps its own record of
outages, so the record outlives the controller.

It runs as a small Docker stack, from Portainer or Docker Compose. Every
account it uses is read-only, and nothing leaves your network.

**Documentation: <https://docs.ferrulehq.dev>**

## The first release is on its way

Ferrule is in private beta. This repository is where it's deployed from:
the compose files, the settings template, and release notes for each
version. They arrive here with **0.9.0**, and the images are published to
`ghcr.io/ferrulehq/ferrule`.

## Feedback

Found something wrong, or want something new? [Open an
issue](https://github.com/ferrulehq/ferrule-deploy/issues/new/choose), or
write to <feedback@ferrulehq.dev>.

---

Ferrule is an independent project, not affiliated with Ubiquiti or the
other vendors it names. See [LICENSE](LICENSE) for the terms.
