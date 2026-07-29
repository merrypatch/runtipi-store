# MerryPatch App Store for Runtipi

A custom [Runtipi](https://runtipi.io) app store publishing **[Bramblekeep](https://github.com/merrypatch/bramblekeep)** — a self-hosted, single-binary workspace and a free, open-source alternative to Notion, Coda, Confluence and ClickUp.

Add this store to your Runtipi instance and Bramblekeep installs like any other app: one click, no compose file to write, data kept in Runtipi's app-data directory and covered by its backups.

<p align="center">
  <img src="https://raw.githubusercontent.com/merrypatch/bramblekeep/master/.github/assets/home.png" alt="Bramblekeep home page: new page, all pages, documentation and support entry points" width="840">
</p>
<p align="center"><sub>Home — where to go from a standing start, with your page tree in the sidebar.</sub></p>

<p align="center">
  <img src="https://raw.githubusercontent.com/merrypatch/bramblekeep/master/.github/assets/page.png" alt="A page with a cover image and the slash menu open" width="840">
</p>
<p align="center"><sub>A page — cover with the photographer's credit, emoji icon, and the <code>/</code> menu: sub-pages, inline databases, embeds, headings, lists.</sub></p>

<p align="center">
  <img src="https://raw.githubusercontent.com/merrypatch/bramblekeep/master/.github/assets/db.png" alt="A database in table view with the new-column dialog open" width="840">
</p>
<p align="center"><sub>A database — typed columns, including relation, rollup, formula and read-only metadata.</sub></p>

## Install Bramblekeep on Runtipi

1. In your Runtipi dashboard, go to **Settings → App Stores → Add App Store**.
2. Paste the repository URL:

   ```
   https://github.com/merrypatch/runtipi-store
   ```

3. Give it a name (for example `MerryPatch`) and save.
4. Open the **App Store** tab, find **Bramblekeep**, and click **Install**.

Requires Runtipi **4.5.0** or newer. The image is multi-arch (`amd64` + `arm64`, Raspberry Pi 64-bit included).

Every setting is optional — install it as-is and the first visitor creates the owner account with an email and a password. To pull a newer Bramblekeep release later, click **Update App Stores** in Runtipi settings, then update the app.

### Settings worth knowing about

| Setting | When you need it |
| --- | --- |
| Setup code | The app is reachable before you get to the sign-up screen, and you want to be the one who claims it. |
| Secure session cookie | You expose the app over **HTTPS**. Leave it off for plain `http://<ip>:8390`, or the browser drops the session cookie and sign-in silently fails. |
| SMTP host / port / username / password / from | You want magic links and invitations emailed. Without SMTP, password sign-in works and those links are written to the container logs. |
| Unsplash access key | You want the cover-image picker. Can also be set later in Bramblekeep's own settings. |

`PUBLIC_BASE_URL` is derived automatically from the domain Runtipi assigns the app, so email links point at the right host with no configuration.

## Repository structure

```
apps/
└── bramblekeep/
    ├── config.json           # app metadata + the form shown at install time
    ├── docker-compose.json   # service definition Runtipi builds the real compose from
    ├── docker-compose.yml    # same thing in compose + x-runtipi form
    ├── data/                 # seeded into ${APP_DATA_DIR} before first start
    └── metadata/
        ├── description.md    # long description shown in the dashboard
        └── logo.jpg          # square 1:1 logo
```

`apps/whoami/` is the upstream template's test app; it is harmless and useful for checking that a freshly added store works at all.

## Development

Runtime is [Bun](https://bun.sh).

```bash
bun install
bun test          # validates every app against Runtipi's own schemas
```

Container image tags are kept current by Renovate (`.github/workflows/renovate.yml`, nightly), which bumps the image in both compose files and then runs `scripts/update-config.ts` to raise `tipi_version` and `version` in `config.json`.

## Documentation

- [Create Your Own App Store](https://runtipi.io/docs/guides/create-your-own-app-store) — the guide this store follows
- [Bramblekeep](https://github.com/merrypatch/bramblekeep) — the app itself
