# Framer VSL Call Funnel — Production Skill

A Claude Code skill that turns an existing Optimally **demo** landing page into a **production-ready
VSL call funnel inside Framer**, end to end — a funnel page whose CTAs open an opt-in modal (Optimally
checkout embed) → `/typeform` redirect → Typeform booking → a `/confirmed` show-rate thank-you page.

This is **separate** from the demo-production skill (`vsl-funnel-demo`, which makes the static HTML demo).
This skill rebuilds that demo as a fully editable, responsive Framer site with a working opt-in modal,
the `/typeform` redirect, and a confirmation page.

## What it covers
- Recreating the demo's premium VSL structure as editable Framer sections (responsive).
- Reusable components: **CTA Button** (brand colour, label control, breathe loop, size variants),
  **VSL Embed** (16:9, URL/HTML control), **Footer** (logo, links, Meta disclaimer, "Site made by
  Optimally" link), **Opt-in Modal** (progress bar + title + checkout embed + close).
- Full-width VSL sizing (fixed px desktop/tablet, fluid phone).
- The opt-in modal wiring (one overlay per CTA, shared modal component).
- Site-wide custom code (config block + `buildOptimallyCheckoutUrl` + path-gated `/typeform` redirect
  + SPA observer) — the workaround for Framer's site-level-only code.
- The blank `/typeform` redirect page and the `/confirmed` confirmation page.
- All the Framer-agent gotchas that cost time (px font sizes, aspectRatio rules, overlay/control
  canonical-id quirks, loop-effect default spin, etc.).

## Use it
Install the `framer-vsl-funnel/` folder into your Claude Code skills directory
(`~/.claude/skills/`), or have a routine pull this repo and copy it in. It requires the `@framer/agent`
CLI (`npx @framer/agent@latest setup`) and the generated `framer` skill loaded first.

The skill's full instructions live in [`framer-vsl-funnel/SKILL.md`](framer-vsl-funnel/SKILL.md).

## Inputs (per client, from the client DB)
`client_id`, production `redirect_url` (`/typeform`), `typeform_url`, optional `use_webinar` + `webinar_id`.

---
Built for Optimally. Funnel CTAs forward all tracking params (utm/ad/partner/closer/setter/adset) into the
checkout, and the footer credits `optimally.ltd/client-site?client=<client_id>`.
