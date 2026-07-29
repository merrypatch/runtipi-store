## Bramblekeep

[Bramblekeep](https://github.com/merrypatch/bramblekeep) is a unified, self-hosted, **single-binary** workspace — a free, open-source alternative to the proprietary all-in-one tools (Notion, Coda, Confluence, ClickUp, and the like), without the vendor lock-in. Your data stays in a single file you own: one SQLite database plus an uploads folder, both inside the app's data volume.

Rust backend (Axum + SQLite) with an embedded React frontend. No external database, no cache server, no message queue — the container is the whole installation.

![Bramblekeep home page: new page, all pages, documentation and support entry points](https://raw.githubusercontent.com/merrypatch/bramblekeep/master/.github/assets/home.png)

*Home — where to go from a standing start, with your page tree in the sidebar.*

![A page with a cover image and the slash menu open](https://raw.githubusercontent.com/merrypatch/bramblekeep/master/.github/assets/page.png)

*A page — cover with the photographer's credit, emoji icon, and the `/` menu: sub-pages, inline databases, embeds, headings, lists.*

![A database in table view with the new-column dialog open](https://raw.githubusercontent.com/merrypatch/bramblekeep/master/.github/assets/db.png)

*A database — typed columns, including relation, rollup, formula and read-only metadata.*

### Features

**Pages and blocks**

- A rich editor over a CRDT: several people type in the same page at once, and the content survives a restart of the container.
- Sub-pages nest to any depth, `@` mentions link pages together and are listed back as backlinks.
- Full-text search inside content, favourites, drag & drop page tree, 30-day trash, per-page change history.

**Databases**

- Fourteen column types plus four read-only metadata ones.
- Six views over the same rows — table, board (kanban), calendar, gallery, chart, graph — each with its own filters, sort and search.
- Rows are real pages you can open and write inside.
- Relations between databases, rollups across the link, and formulas (28 functions across logic, numbers, text and dates).

**Charts and graph**

- Bars, line, area, pie, radar, radial — grouped by hour / day / week / month, split into series by any column, with cumulative, remaining and burndown transforms.
- A force-directed graph of relation links and page references, inside a database or across all your pages.

**Sharing and accounts**

- Per-page levels (read / edit / creator / admin) inherited by the subtree, plus workspace roles (owner / admin / member).
- Public pages: a token link readable without any account, optionally covering the subtree.
- Email + password sign-in with no mail relay needed, or magic links once SMTP is configured. Opaque sessions, no JWT.

**Also**

- Built-in documentation, ten chapters shipped inside the binary, so it always matches the version you run.
- English, French and Spanish interface, light/dark themes, installable as a PWA.
- Markdown / PDF / CSV export, CSV and relation-preserving ZIP import, content-addressed uploads.
- Zero telemetry, and no outbound network call unless you ask for one.

### First run

Open the app and the first visitor creates the owner account with an email and a password — nothing else to configure. If the app is reachable before you get to that screen, set a **setup code** in the app settings first: creating the owner then requires it.

### Notes for Tipi users

- **Exposing it.** `PUBLIC_BASE_URL` is derived automatically from the domain Tipi gives the app, so magic links and invitation emails point at the right host. When you expose it over HTTPS, also turn on **Secure session cookie** in the app settings. Leave that off for plain `http://<ip>:8390` access, or the browser will discard the session cookie and sign-in will appear to do nothing.
- **Updates.** Tipi manages upgrades — pull the new version from your app store and update the app from the Tipi dashboard. Bramblekeep's own in-app update button stays hidden here on purpose: it drives a Watchtower sidecar, which would fight Tipi over the same container.
- **Inviting people.** Without SMTP, invitation and login links are written to the container logs and you hand them over yourself. Fill in the SMTP fields to have them emailed instead.
- **Your data.** Everything lives in the app's data directory (`data/` → `/data` in the container): `bramblekeep.db` and the uploaded `files/`. Tipi's app backups cover it.
