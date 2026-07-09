# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Fork workflow — read this first

This is **pauldiee's fork** of `lcoscia/vcf-planner` (remote `upstream`), used as a staging ground to preview changes before proposing them upstream as PRs and issues.

- **This CLAUDE.md exists only on the fork's `main`** — it must never end up in a PR to upstream. To guarantee that, create feature branches from `upstream/main`, not from the fork's `main`: `git fetch upstream && git checkout -b <branch> upstream/main`. PR diffs then contain only the intended change.
- PRs target `lcoscia/vcf-planner`; push branches to `origin` (the fork).
- Sync the fork with `git fetch upstream && git merge upstream/main` on `main` (the CLAUDE.md commit stays on top).

## Commands

No build step, no bundler — the website is `index.html` + native ES modules in `core/`.

```bash
# Serve the site (ES modules don't load over file://)
python -m http.server 8000        # → http://localhost:8000/index.html

# Sizing regression tests (3 end-to-end scenarios vs core/sizing.js)
node --test mcp/test/scenarios.test.js

# Verify LT lookup tables against the official Excel workbook
# (requires the gitignored vcf-9.1-planning-and-preparation-workbook-updated.xlsx next to index.html)
python tools/check_lt_constants.py

# MCP server (optional, Node >= 20)
cd mcp && npm install && npm start   # → http://localhost:3000/
```

There is no linter. The only automated test is the scenarios test above — run it after touching `core/sizing.js` or `core/data.js`.

## Architecture

**Single source of truth: `core/`.** The sizing math and reference tables live once in `core/*.js` (native ES modules) and are consumed by two frontends that must never drift apart:

- `index.html` (~3300 lines) — the entire website: Alpine.js 3.14.1 + Tailwind CDN, all inline. Its Alpine methods are thin wrappers delegating to `Core.*` (imported as `window.vcfPlanner`). `NAV_GROUPS`, the VCF-Installer import/export helpers, and `vcfPlanner()` stay inline here.
- `mcp/` — optional remote MCP server (Express + @modelcontextprotocol/sdk) that imports `../core/` and exposes the same calculator/reference data as tools.

**Rule of thumb:** sizing formulas, lookup tables, form schema, and validation go in `core/`; UI rendering, navigation, and export logic go in `index.html`. Never duplicate a formula in `index.html` — call the `core/` function.

Inside `core/`:
- `reference.js` — `ALL_PAGES`: the form schema driving all 19 pages. Array of `{ id, title, sections[] }`, each section has `fields[]` with `type`, `showWhen: f => ...` (conditional visibility off form state), `optionsFn: f => [...]` (dynamic dropdowns), `required`, `docLink`. Adding a field = adding an entry here; adding a page also needs a `NAV_GROUPS` entry in `index.html`.
- `sizing.js` — pure calculator (`calcHosts`, `calcTotalDisk`, `computeSizing`, …) mirroring the Excel workbook's `Management Domain Sizing` formulas, including its exact order of operations and a float-safe `roundUp()` (`Math.ceil(x - 1e-9)`).
- `data.js` — `LT` lookup tables + `SUBNET_MASKS`, verified by `tools/check_lt_constants.py` against the official workbook.
- `validation.js` — IP/CIDR/FQDN checks + VLAN/IP/CIDR conflict detection.
- `ports.js` — 1083-row Ports & Protocols matrix.

**Fidelity constraint:** this app is a re-implementation of Broadcom's official VCF 9.1 Planning & Preparation Workbook (Excel). Changes to sizing formulas or lookup values should trace back to the workbook or official Broadcom docs — the README's changelog documents each alignment pass, and the few intentional web-only deviations are listed under "Sizing calculator — web-only additions" in README.md.

Persistence is `localStorage` key `vcf-planner-v1` (stores `{ form, sizing, currentPage, openGroups, openSections }`).
