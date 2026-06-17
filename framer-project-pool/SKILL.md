---
name: framer-project-pool
description: >-
  Provision (or top up) a pool of pre-authorized BLANK Framer projects into the Baserow `Framer Project
  Data` table, so the `framer-vsl-funnel` routine can claim a project and build headlessly without the
  interactive `project new` browser step. Use when the user says "top up the Framer pool", "make more
  Framer projects", "we hit 50 / the pool is empty", "provision Framer projects", or "refill the project
  pool". This pre-pays the one-time browser-approval cost in bulk; everything after is headless.
  Mandatory precondition: run `npx @framer/agent@latest setup` and read the `framer` skill first.
---

# Framer Project Pool — bulk provisioning / top-up

Creates N blank, authorized Framer projects and lands them in Baserow as `Status = Available` rows that
the [[framer-vsl-funnel]] routine claims. Each Framer project carries its own `fr_…` API key, which is
the portable headless credential (`project auth <id> <key>` re-authorizes on any machine, no browser).

**The ONE manual cost:** a human must click each project's authorize link (browser approval). This skill
makes that a fast click-through and does all extraction/verification/DB-loading automatically.

## Hard constraints (learned the hard way — heed them)
- **`project new` is interactive.** With `BROWSER=none` it prints a one-time URL
  `https://framer.com/projects/server-api/auth?callback=http://127.0.0.1:<port>/callback&state=…&createProject=true`,
  starts a **local callback server on `<port>`, and BLOCKS until the link is clicked + approved.**
- **Click on the SAME machine** running the process (the callback is `127.0.0.1`).
- **~10-minute timeout** per link → `Browser authorization timed out`. So batch + click promptly; don't
  leave 50 clocks ticking at once.
- **RAM:** each `project new` ≈ 70MB (+ its npx node). Check free RAM (`Get-CimInstance Win32_OperatingSystem`)
  and size batches so total stays well under free memory — **default batch = 10**.
- **Keep processes alive:** launch a batch as ONE backgrounded command that ends in `wait`, so the parent
  holds the children (a lone `( … ) &` can be killed when the tool shell exits).
- **Node sees `/tmp` as `D:\tmp` on Windows** — read logs via the Bash tool (grep/tail), not `node fs` on `/tmp`.
- **npm-cache disk:** repeated `npx` fills `~/AppData/Local/npm-cache`; clear it if disk runs low.

## Inputs
- **target** — desired total Available projects (default top up to **50**).
- **Baserow table** — `Framer Project Data`, id `1033106` (db 453125). Fields: `Project ID` (primary),
  `Project URL`, `API Key`, `Status`, `Claimed By`, `Claimed At`, `Notes`, `Authorize URL`.

## Procedure (repeat in batches of ~10 until target reached)

**0. Size the job.** List the table; count `Status = Available`. `need = target − available`. Batches of
`min(10, need)`.

**1. Launch a batch** (background, kept alive by `wait`), logging each to a file:
```bash
cd "$HOME"
for i in $(seq -w 1 10); do
  ( BROWSER=none npx @framer/agent@latest project new > "/tmp/fpool/b_$i.log" 2>&1 ) &
done
wait
```
Sleep ~20s, then scrape each log's URL: `grep -o 'https://framer.com/[^ ]*' b_$i.log | head -1`. If a log
is empty (slow npx start), relaunch just that slot with `npx … project new` (run_in_background) and grab its URL.

**2. Write the links to Baserow** — one row per link via `create_rows`: `Project ID="PENDING-<n>"`,
`Authorize URL=<link>`, `Status="Awaiting Auth"`, `Notes="<round/port>"`. Keep an ordered list mapping each
log file → its row id (the harvest depends on it).

**3. Human clicks the batch** from the table's `Authorize URL` field, within ~10 min, on this machine. Wait
for the launcher task to complete (all clicked) or the user to confirm.

**4. Harvest.** Each saved process's log ends with `Project <id> saved` (exact log→project map). For each
row: read its project's `fr_` key from `~/AppData/Roaming/framer/projects.json` (via a `node -e` with the
ids hardcoded — don't read `/tmp` from node), then `update_rows` to set `Project ID=<id>`,
`Project URL="https://framer.com/projects/<id>"`, `API Key=<key>`, `Status="Available"`, `Authorize URL=""`.

**5. Verify routine-readiness** (the real test): for at least a sample, headless
`project auth <id> <key>` then `session new <id>` (returns a session id). Any failure → set that row
`Status="Error"` + note instead of Available.

**6. Repeat** from step 1 until `available == target`. Finish by listing the table and confirming the
Available count.

## Exclusions
Never add the pre-existing **real** client/demo projects to the pool (e.g. Essential Experiences, Sunnyhill,
Spicy Circumstances). Filter `projects.json` to only the ids your batches just created (match the `… saved`
log lines), not everything in the cred store.

## Notes
- The `fr_` key is a secret; Baserow is its store — keep that table's access locked down.
- Custom domains are connected later by remixing the built project (dashboard), so the pool table omits
  Live-URL/domain fields by design.
- See [[framer-agents]] for the underlying CLI behaviour and [[framer-vsl-funnel]] for how rows get claimed.
