# Framer VSL Call Funnel — Production Skills

Claude skills that turn an existing Optimally **demo** landing page into a **production-ready VSL call
funnel inside Framer**, end to end — a funnel page whose CTAs open an opt-in modal (Optimally checkout
embed) → `/typeform` redirect → Typeform booking → a `/confirmed` show-rate thank-you page.

This is **separate** from the demo-production skill (`vsl-funnel-demo`, which makes the static HTML demo).

## Contents
- [`framer-vsl-funnel/SKILL.md`](framer-vsl-funnel/SKILL.md) — the build skill. Resolves the demo from
  Baserow, pulls its code from the `demos` repo, claims a project from the pool, rebuilds it as a
  responsive Framer funnel (components, opt-in modal, `/typeform`, `/confirmed`, site code, animations),
  runs a mandatory self-audit, publishes, and hands off (Baserow + GHL + Slack).
- [`framer-project-pool/SKILL.md`](framer-project-pool/SKILL.md) — provision/top-up the pool of
  pre-authorized blank Framer projects the build skill claims from.

## Remote / routine usage (no local skill — fetch this repo)
A remote routine has nothing installed. It should:
```bash
git clone https://github.com/ReubenShears/framer-vsl-funnel-skill
npx @framer/agent@latest setup          # installs the `framer` skill + CLI
```

### Network egress (REQUIRED in the remote container)
The Framer agent session is a **WebSocket to `api.framer.com`** — if it's not in the container's egress
allowlist, `session new` fails with `FramerAPIError: No connection to the server` (while `project auth`
misleadingly "saves" locally). The environment must allow:
- **`*.framer.com`** — api + auth (the actual blocker is `api.framer.com`)
- **`framerusercontent.com`** — screenshots (the mandatory audit downloads these) + asset uploads
- **`*.framer.app`** — published-site verification
- plus `github.com`, the npm registry, and `api.anthropic.com` (usually already allowed).
Then **read and follow** `framer-vsl-funnel-skill/framer-vsl-funnel/SKILL.md` end to end. If it reports the
project pool is empty, follow `framer-vsl-funnel-skill/framer-project-pool/SKILL.md` to top it up. The
SKILL.md files are plain instructions — the agent reads and executes them; they don't need to be "installed".

## Local usage
Copy `framer-vsl-funnel/` (and `framer-project-pool/`) into `~/.claude/skills/`. Requires the `framer`
skill loaded first (`npx @framer/agent@latest setup`).

## Notes
- These run **headless**: they claim a pre-authorized project from the Baserow pool (`Framer Project Data`)
  and `project auth` with its per-project key — never the interactive `project new`.
- `[[double-bracket]]` references in the SKILL.md files are optional internal Optimally memories; everything
  load-bearing (the Meta disclaimer text, Slack house style, the GHL field ID, the Framer gotchas) is inline,
  so the skills are self-contained for a remote run.

---
Built for Optimally. Funnel CTAs forward all tracking params (utm/ad/partner/closer/setter/adset) into the
checkout; the footer credits `optimally.ltd/client-site?client=<client_id>`.
