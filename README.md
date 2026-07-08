# 🤖 Self-hosting Eleven

Run your own Eleven server: end-to-end-encrypted group chat where **your box
never sees a message** — the server stores only ciphertext, and every key stays
on your members' devices. One static binary, one directory of state, automatic
TLS, automatic updates.

> This page is published as the README of
> [`elevenmessenger/releases`](https://github.com/elevenmessenger/releases)
> (the public downloads repo). Its source of truth is `docs/self-hosting.md`
> in the main repo.

## What you need

- A Linux box: **Ubuntu or Debian** with systemd (what we test on), `x86_64`
  or `arm64`, 1 GB RAM is plenty to start. A €4/month VPS works.
- **Root** on it, with inbound ports **80 and 443** open.
- A **domain** (or subdomain) you can point at the box.

## Install

```sh
curl -fsSL https://github.com/elevenmessenger/releases/releases/download/server/install.sh | sudo bash
```

That downloads the latest **stable** release for your architecture, verifies
its published sha256, installs it at `/opt/lchat/lchat`, and bootstraps the
box: a locked-down service user, systemd units, the TLS router on 80/443, and
the hourly auto-update timer. (Prefer not to pipe to shell? Download
`install.sh` from the same URL, read it — it's short — then run it.)

## Create your first chat

Point a DNS **A record** at the box (`chat.example.com → your box IP`), then:

```sh
lchat add-instance chat.example.com
```

It prints a one-click **setup link**. Open it: you pick a name, your browser
generates your keys (they never leave it), and Eleven walks you through naming
the space and inviting people. The Let's Encrypt certificate issues
automatically on the first request — no certbot, no reverse proxy.

Each `add-instance` creates a **fully isolated** chat: its own database, admin
token, and subdomain. Run one for the family and another for the club; they
share nothing.

## What you get out of the box

- **End-to-end encryption**: messages, names, photos, and files are encrypted
  on devices; the server relays and stores ciphertext. If your box is
  compromised, the attacker gets nothing readable.
- **Web app + notifications**: members use it from any browser; web push
  notifications work with zero configuration.
- **Automatic updates**: the box checks our signed release feed hourly and
  applies **stable**-channel updates with a health check and automatic
  rollback. Each chat's admin can change channel (stable/beta/edge) or turn
  auto-updates off in the admin panel; `lchat update status` shows the fleet.
- **Bots, optionally with AI**: `lchat bot provision` + `lchat bot run` (same
  binary, no other runtime) adds the built-in helpers — Eleven welcomes new
  members and answers slash commands. Set `ANTHROPIC_API_KEY` and they get a
  brain; without it they're deterministic helpers. See `docs/bots.md`.

## Honest limitations

- **Native app push**: members can use the Eleven iOS/Mac apps against your
  server, but native *push notifications* route through Apple via our central
  relay, which needs a per-server token we mint. Web push is unaffected. If
  you want native push for your server, ask us for a relay token (open an
  issue on this repo) — self-serve is planned.
- **Passkey backup routes through us by default**: the optional
  "save a passkey" account backup enrolls against our home server
  (`home.elevenmessenger.com`). What's stored there is passkey-encrypted —
  we can't read it — but if you'd rather keep even that ciphertext on your own
  hardware, run the `home` service yourself and set `LCHAT_HOME_ORIGIN` in
  the instance's `/data/<host>.env`.
- **The trust root for updates is ours**: auto-updates install binaries we
  sign. That's the point of the channels — but if you fork, build your own
  binary with your own key and feed URL (`LCHAT_UPDATE_MANIFEST_URL`).

## Operating it

- **Everything lives in `/data`** — databases, TLS certs, tokens, env files.
  Back that directory up and you can rebuild the box from scratch. (For
  continuous backups, [Litestream](https://litestream.io) replicating
  `/data/*.db` to object storage works well; the installer doesn't set this
  up for you.)
- **Updates**: automatic. `lchat update status` shows every chat's channel and
  version; `lchat update run --dry-run` previews a pass; a bad release rolls
  itself back. Re-running the install one-liner also updates immediately and
  refreshes the systemd units.
- **Admin access**: `lchat admin-link chat.example.com` re-prints a chat's
  admin URL (it's also in `/data/<host>.env`).
- **Remove a chat**: `lchat remove chat.example.com --stop` stops it and
  unroutes it; its database stays in `/data` until you delete it yourself —
  removal never destroys data.
- **Hardening**: the units are already sandboxed; for the box itself, a
  firewall (22/80/443), key-only SSH, and unattended OS upgrades are the
  standard trio.

## Scaling up (optional)

The same binary contains the multi-tenant machinery we run
[11ch.at](https://11ch.at) with — a self-serve provisioner, wildcard
DNS-01 certificates (`LCHAT_WILDCARD_DOMAINS` + a DNSimple token), and a
central home server. None of it is needed for a personal box; the design
docs live in the main repo under `docs/`.
