# Dual-Plex fork of Posterr — notes

Fork of [petersem/posterr](https://github.com/petersem/posterr) (v1.21.18,
commit `4609d7e`) on branch **`dual-plex-now-screening`**. One feature:
**Now Screening polls TWO Plex servers** and merges the sessions, so a
stream from either the 4K or the 1080 Plex instance triggers Now Playing.
Everything else — on-demand posters, libraries, Coming Soon — keeps using
the primary server only.

## What changed (4 files, +117/−2)

- `consts.js` — new settings defaults: `plex2Enabled`, `plex2IP`,
  `plex2HTTPS`, `plex2Port`, `plex2Token`.
- `classes/core/settings.js` — the new keys wired through the Settings
  class, including upgrade defaults so an existing `settings.json`
  (written by upstream Posterr) gains the new keys safely.
- `myviews/settings.ejs` — a "Second Plex Server (Now Screening only)"
  fieldset inside the PLEX panel of the setup page, mirroring the primary
  server's fields.
- `index.js` — `loadNowScreening()` builds a second Plex client when the
  feature is enabled and merges its cards into the rotation using the
  **same filters** (user, local/remote, device, excluded libraries).
  A failure on the second server is caught and logged
  (`*Now Scrn. - Second Plex server unavailable`) — it cannot delay the
  poll timer or blank the primary server's Now Screening.

Also fixed in passing, in `loadNowScreening()` only: the Plex class
expects a boolean `HTTPS`, but upstream passed a key the class ignores
(`plexHTTPS:`), which silently forced plain HTTP for Now Screening. The
fork passes `loadedSettings.plexHTTPS === "true"` as a proper boolean.
(The on-demand loader still has the upstream form — left untouched; it
only matters if you ever enable the HTTPS toggle for on-demand.)

## Verified (Oct 17, 2026)

Smoke-tested against two mock Plex servers with the real app:

- Sessions from **both** servers appeared in `/getcards`.
- A session by a different user on the second server was correctly
  filtered out — the user filter applies to both servers.
- With the second server down, its error was logged and skipped, the
  10-second poll cadence held, and the primary server's cards kept
  rotating.
- The settings page renders with the new fields and saved values.

## Deploy on Unraid

Option A — GHCR (hands-off updates):

1. Add the GitHub Actions workflow file (one-time — see the
   "GitHub Actions workflow" section below), then pushes to this branch
   build the image and publish `ghcr.io/bubba925/posterr:dual-plex`.
2. In Unraid, duplicate your existing Posterr container, change only the
   repository to that image, and keep the same `/config` volume mapping
   (`/usr/src/app/config`) and port. Your existing `settings.json`
   carries over.

Option B — build on the Unraid box:

```bash
git clone <your-fork-url> && cd posterr
git checkout dual-plex-now-screening
docker build -t posterr-dualplex .
```

Then point a container at `posterr-dualplex` with the same config volume.

## Configure

Open Posterr → Settings → **PLEX** panel → **Second Plex Server**:
tick **Enable Second Plex Server**, and fill in the second server's IP,
port, and **its own Plex token** (each server has a different token —
grab it from the 1080 server per the "details" link in the form). Save.
No restart needed beyond Posterr's normal settings flow; the second
server joins the next Now Screening poll (every 10 seconds).

## GitHub Actions workflow

The workflow file (`.github/workflows/docker.yml`, in the local
checkout) could NOT be pushed through the Muse GitHub connector —
GitHub refuses workflow-file writes from app integrations without a
dedicated Workflows permission. Add it once by hand:

1. On the fork's `dual-plex-now-screening` branch, use
   **Add file → Create new file**, path `.github/workflows/docker.yml`,
   and paste the file's contents from the local checkout.
2. If builds don't start, enable Actions once under
   **Settings → Actions** (forks ship with them off).

After that, every push to the branch rebuilds and republishes
`ghcr.io/bubba925/posterr:dual-plex`.

## Publishing status

Done (Oct 17, 2026): the fork is **bubba925/posterr** and this branch
was pushed through the Muse GitHub connector as three snapshot
commits — upstream sync to v1.21.18, the dual-plex patch, and these
notes. The local checkout (with full git history, plus
`posterr-dual-plex.patch` alongside it) remains the structured source
of truth.

## Staying current with upstream

The patch is small and confined to the four files above:

```bash
git remote add upstream https://github.com/petersem/posterr.git
git fetch upstream
git rebase upstream/master   # resolve the rare conflict in the 4 files
git push --force-with-lease
```

A standalone copy of the patch (against upstream `4609d7e`) is kept
alongside this checkout as `posterr-dual-plex.patch`.
