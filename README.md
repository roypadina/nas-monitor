# nas-monitor

Cloud-side uptime monitor for a Synology NAS. A scheduled GitHub Actions job
probes the NAS every ~5 minutes; while it's unreachable it pushes a nudge to
[ntfy](https://ntfy.sh) on every run, and sends a single recovery ping when the
NAS comes back.

This is **layer 2** of a two-layer setup. Layer 1 is **Synology Active Insight**
(native cloud backstop) — see the bottom of this file.

## How it works

- `.github/workflows/nas-monitor.yml` runs on `cron: */5 * * * *` (and a manual
  **Run workflow** button).
- It `curl`s `NAS_URL` up to 3 times (10s apart) — any HTTP response = up. Only a
  total connection failure counts as **down**, so a brief network blip won't page you.
- **Down** → push to `NTFY_URL` with `Priority: urgent`. Fires every run while down
  = the nudge.
- **Back up** → one `NAS back online` push on the down→up transition.
- `state.txt` holds `up`/`down`; it's committed back **only when state changes**,
  so history stays clean and there's no false recovery ping on the first run.

Minutes are **free and unlimited** because this repo is **public**. The
`NAS_URL` and ntfy topic live in encrypted repo **Secrets**, which are masked in
logs and never exposed by a public repo.

## Setup (one time, ~3 min)

1. **Install the ntfy app** (iOS / Android). Subscribe to a *secret* topic — the
   random string IS the password, e.g. `nas-roy-9f3kx72q4w`. Set it to high
   priority + a loud sound.
2. **Add repo Secrets** — Settings → Secrets and variables → Actions → New
   repository secret:
   - `NAS_URL` — your NAS public endpoint, e.g. `https://your-ddns.synology.me:5001`
     or your QuickConnect URL.
   - `NTFY_URL` — `https://ntfy.sh/<your-secret-topic>`.
3. **Test** — Actions tab → **NAS monitor** → **Run workflow**. With the NAS up you
   get no push. To prove the nudge path, temporarily set `NAS_URL` to a dead
   port (e.g. `https://your-ddns.synology.me:65000`) and run again — you should
   get a `NAS DOWN` push. Restore the real value after.

## Acknowledge / stop the nudge

- **Mute** the ntfy topic in the app → instant silence, monitor keeps running.
- Or **disable** the workflow: Actions → NAS monitor → ⋯ → Disable workflow.

## Caveats

- 5-minute granularity (GitHub's minimum cron); scheduled runs can lag a few
  minutes under GitHub load.
- GitHub **pauses scheduled workflows after 60 days of no repo commits**. A
  down/up transition commits `state.txt` and resets that timer; if the NAS never
  goes down for 60 days, hit **Run workflow** once (or push any commit) to keep
  it alive.

## Layer 1 — Synology Active Insight (native backstop)

Do this in DSM, independent of this repo:

1. Package Center → install **Active Insight** (if not already).
2. Open it → sign in with your Synology Account → register this NAS.
3. Lower the **"Disconnected from Active Insight server"** warning threshold to
   its minimum.
4. Install the **Synology Active Insight** mobile app → sign in → allow push.

Active Insight is cloud-side (the NAS heartbeats out), so it catches a full
outage even if this GitHub job lags or is paused — but it's slow (~1h default)
and doesn't nudge. It's the safety net; this repo is the fast pager.
