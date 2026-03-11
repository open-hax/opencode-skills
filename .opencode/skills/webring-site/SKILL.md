---
name: webring-site
description: "Research the live ussyco.de ring, build a distinct single-page site in this workspace, preview it locally, and optionally register it with an API key from the environment"
compatibility: opencode
metadata:
  audience: agents
  workflow: webring
  version: 2
---

# Skill: Webring Site

## Goal
Create a one-page site that feels native to the `ussyco.de` ring without copying existing members, then preview it locally and optionally register it safely.

## Use This Skill When
- The user wants a new `ussyco.de`/ussyring-style site.
- The request includes slop, a themed microsite, retro web aesthetics, or ring registration.
- You need to study the current ring before generating a fresh concept.

## Do Not Use This Skill When
- The task should follow an existing app or product design system.
- The request is only to tweak a normal website unrelated to the ring.
- The user requires immediate live registration but no public URL or deployment target exists.

## Environment
- Workspace root: `/home/err/devel`
- Default site root: `/home/err/devel/projects/webring-sites/<slug>/`
- Ring hub: `https://ussyco.de/hub/`
- Ring data: `https://ussyco.de/api/webring`
- Widget script: `https://ussyco.de/api/webring/widget.js`
- Health check: `https://ussyco.de/api/health`
- Registration endpoint: `https://ussyco.de/api/admin/members`
- Auth env var: `USSYCO_DE_API_KEY`
- Never commit, hardcode, or echo the literal API key into tracked files.

## Workflow
1. **Study the ring first**
   - Fetch `https://ussyco.de/api/webring` and inspect the hub page.
   - Browse at least 4 member sites across different aesthetics before designing anything.
   - Use `agent-browser`, `webfetch`, or a research subagent to capture tone, structure, palette, typography, and recurring content devices.
   - Record what is already crowded so the new site does not feel like a clone.

2. **Choose a concept that is novel**
   - Make it absurd, specific, and played totally straight.
   - Prefer a framing device such as bureau, archive, catalog, devotional page, public notice, museum placard, field guide, or service manual.
   - Define a premise, palette, typography direction, and 3-5 in-world content modules before writing code.
   - Avoid defaulting to crowded lanes like generic green-on-black terminal parody, mock legal paperwork, or haunted inventory unless the user explicitly wants that lane.

3. **Build in this workspace**
   - Create `projects/webring-sites/<slug>/`.
   - Default to plain HTML, CSS, and vanilla JS unless the user asks for a different stack.
   - Avoid CDN dependencies; bundle everything locally.
   - Include a hero/header, at least 3 themed sections, detailed in-world copy, and a footer.
   - Make the page work on desktop and mobile.

4. **Add the webring widget**

```html
<div id="webring" data-theme="dark"></div>
<script src="https://ussyco.de/api/webring/widget.js"></script>
```

   - Omit `data-theme="dark"` for light sites.
   - Match the widget theme to the actual page palette.

5. **Preview and verify locally**
   - Serve the site from its directory with a local static server.
   - Prefer a stable local port when repeated browser testing is useful.
   - Verify all of the following:
     - `curl` returns the expected HTML.
     - The page renders in a browser.
     - The layout still works at narrow/mobile width.
     - The widget script loads.

6. **Register only when publication is actually requested**
   - Read the API key from `USSYCO_DE_API_KEY`.
   - Do not persist the secret in repo files, screenshots, logs, or commit messages.
   - In this workspace, default registration to `mode: "redirect"` with a public URL.
   - Use `mode: "proxy"` only when the target is reachable from the ring host and direct proxying is intentionally supported.
   - If the site only exists on `localhost`, registration is blocked until the site is deployed somewhere public or moved onto the ring host.

7. **Use the current member payload shape**

```bash
curl -X POST https://ussyco.de/api/admin/members \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: $USSYCO_DE_API_KEY" \
  -d '{
    "name": "<Display Name>",
    "subdomain": "<subdomain>",
    "url": "https://<public-host-or-path>",
    "description": "<Punchy one-line description>",
    "mode": "redirect",
    "capabilities": {
      "can_redirect": true,
      "can_proxy": false,
      "can_cname": false,
      "can_profile": true
    },
    "theme": "{\"bg\":\"#111111\",\"text\":\"#f5f5f5\",\"accent\":\"#ffcc00\",\"border\":\"#444444\",\"font\":\"Georgia, serif\"}"
  }'
```

   - Use lowercase hyphenated subdomains.
   - Before posting, confirm the subdomain is not already present in `GET /api/webring`.
   - Avoid obvious system names such as `www`, `api`, `admin`, `mail`, and `localhost`.
   - If registration fails, inspect the response and adjust the payload instead of guessing.

8. **Verify publication when registration succeeds**
   - Confirm the member appears in `https://ussyco.de/api/webring`.
   - Confirm the public site URL works.
   - Confirm the hub card colors match the site's actual palette.
   - If proxy mode was used, confirm the proxied subdomain serves the expected HTML.

## Output
- Site path, concept, and visual direction
- Local preview URL and verification notes
- Registration status: `not requested`, `blocked`, or `registered`
- Live URL if registration succeeded

## Strong Hints
- The best ring sites are sincere, over-specific, and heavily authored.
- Treat content modules like props: warnings, specimen cards, forms, bulletins, inventories, notices, schedules, testimonials, decrees.
- Use motion sparingly and theatrically.
- Make the hub theme colors feel like a thumbnail of the actual site.
- The goal is similar energy, not copied structure.

## References
- Related browsing skill: `agent-browser`
- Related workspace search skill: `workspace-navigation`
- Related testing skill: `testing-e2e`
