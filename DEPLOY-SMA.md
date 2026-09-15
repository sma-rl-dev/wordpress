# WordPress - Deployment, Smoke, and Mutation Notes

## Quick Start

```bash
./tester-env deploy    # Start Nginx+PHP-FPM+MySQL stack, install WordPress from src/
./tester-env seed      # Populate deterministic Harbor Support Notes demo content
./tester-env verify    # Check home/login respond + admin user + seed presence
./tester-env reset     # Clean slate (stop stack + remove DB volume; images kept)
```

## CLI Commands

| Command | Description |
|---------|-------------|
| `deploy` | Generate `wp-config.php` (first run), start `db`+`php`+`web`, reset DB, install core |
| `seed` | Upsert Harbor demo content: settings, editor user, terms, 4 posts, 1 page, 3 comments |
| `verify` | Assert `/` and `/wp-login.php` serve 200, admin + editor exist, all seed slugs present |
| `reset` | `compose down -v` (stack + DB volume removed, images kept) |
| `stop` | Stop stack (preserves data) |
| `logs` | Tail stack logs |
| `status` | Show stack status and URL |

Options:

- `--run-id <id>` isolates compose project, container, and volume names for parallel scenario runs.
- `--port <port>` binds the app to a specific host port (default: 8889).

Parallel scenario example:

```bash
./tester-env deploy --run-id run-001 --port 8891
./tester-env seed --run-id run-001
./tester-env reset --run-id run-001
```

- Port: `8889`
- URL: `http://localhost:8889`

## Build

```bash
./tester-env deploy
```

Prerequisites:

- `docker` (compose v2)
- `curl`

No app image is built. All tiers are stock upstream images:

- `mysql:8.4` (DB; initialized with `tester-env.d/mysql-init.sql`)
- `wordpressdevelop/php:8.5-fpm` (PHP-FPM; `LOCAL_DIR=src` source-mount pattern)
- `nginx:alpine` (web; renders upstream `tools/local-env/default.template` via `envsubst`)
- `wordpressdevelop/cli:8.5-fpm` (one-shot WP-CLI for install/seed/verify)

The app code is the fork checkout itself: `tester-env.d/docker-compose.tester-env.yml`
mounts the repo root as `/var/www`, so edits under `src/` (e.g. mutant patches) take
effect without rebuilding anything.

## Credentials

- Admin: `admin` / `password` (override via `WORDPRESS_ADMIN_USER` / `WORDPRESS_ADMIN_PASSWORD`)
- Editor: `editor` / `editorpass` (created by `seed`)
- DB: `wordpress_develop` / `root` / `password` (password override via `WORDPRESS_DB_PASSWORD`)

## Seed Data

```bash
./tester-env seed
```

Seeded content (theme: Harbor support team; idempotent slug-keyed upserts):

- Settings: site title `Harbor Support Notes`, tagline `How-to guides from the Harbor support team`, timezone `America/New_York`
- Users: `editor` (`Dana Editor`, editor@example.com, role editor)
- Terms: categories `guides`, `release-notes`; tags `onboarding`, `troubleshooting`
- Posts: `welcome-to-harbor-support` (publish), `reset-your-password` (publish), `february-release-notes` (publish), `printer-setup-draft` (draft, unpublished)
- Page: `about-support` (publish)
- Comments: 2 on `reset-your-password` (1 approved, 1 pending), 1 approved on `welcome-to-harbor-support`
- Stock install content (`hello-world`, `sample-page`, `privacy-policy`) removed so reruns converge

## Reset

```bash
./tester-env reset
```

Then run `deploy` and `seed` to start fresh. For a full clean slate including images,
remove `tester-env-wordpress:dev` / `mysql:8.4` / `wordpressdevelop/*` images manually.

## Browser Verification

```bash
./tester-env verify
```

Checks:

- Home page (`/`) serves 200
- Login page (`/wp-login.php`) serves 200
- Admin user exists
- All 5 seed posts/pages and the editor user are present

Smoke: log in at `http://localhost:8889/wp-login.php` with `admin` / `password` and confirm
the dashboard loads; visit `/` and confirm `Harbor Support Notes` posts are listed.

## Mutation Smoke

Edit any file under `src/` (e.g. a post template) and reload the page — no rebuild needed
because the checkout is bind-mounted into the `php` and `web` containers.

## Baseline

Tag: `7.1.0`
Pinned SHA: `daaca56d3d6a9a42a0c87f6eda766c33a77c1d05`
Branch: `tester-env-baseline`
