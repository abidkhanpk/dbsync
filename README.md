# Schema Sync

Tiny Node/Prisma workspace that versions your existing PostgreSQL/MySQL or other schema and deploys migrations alongside a PHP (or any) app. Use it to introspect (pull) production, capture a baseline, and manage future changes with repeatable migrations.

---

## Why this exists
- Keep your legacy app untouched while gaining migration history.
- Baseline an already-live schema instead of rebuilding it.
- Safely diff dev vs. prod using Prisma’s tooling.
- Automate prod deploys via GitHub Actions.

---

## Project layout
```
.
├── package.json           # npm scripts for Prisma workflows
├── prisma/
│   ├── schema.prisma      # datasource definition (models after pull)
│   └── migrations/        # baseline + future migration folders
├── .env                   # local dev/shadow URLs (not committed with secrets)
├── .env.production        # prod/shadow URLs for CI/server use
└── .github/workflows/
    └── prisma-deploy.yml  # deploy migrations to production on push
```

---

## Prerequisites
- Node.js 20+
- npm
- Databases for dev, shadow, and production
- Prisma CLI (installed via `npm install`)
- GitHub repository if you plan to use the provided CI workflow

---

## Initial setup
1. **Install dependencies**
   ```sh
   npm install
   ```

2. **Configure environment variables**
   - Copy `.env` with real dev/shadow URLs.
   - Create `.env.production` with production/shadow URLs (store secrets in CI/CD, not git).

3. **Introspect existing schema**
   - Point `DATABASE_URL` at a *read-only* production user.
   - Run:
     ```sh
     npm run prisma:pull
     ```
   - Review `prisma/schema.prisma` for warnings; keep unsupported bits for SQL migrations.

4. **Create baseline migration**
   ```sh
   mkdir -p prisma/migrations/000_init
   npm run prisma:baseline:sql
   ```
   - Append any existing views/triggers/functions manually so 000_init fully recreates prod.

5. **Mark baseline as applied in production**
   - Ensure `DATABASE_URL` points to production.
   ```sh
   npm run prisma:mark:baseline:applied
   ```

6. **Apply baseline to dev**
   - Reset dev DB if needed.
   - With dev URLs active:
     ```sh
     npm run prisma:deploy
     ```

At this point dev and prod share migration `000_init`.

---

## Day-to-day workflow
1. Edit `prisma/schema.prisma` for a new feature (add columns, tables, etc.).
2. Generate and apply a dev migration:
   ```sh
   npm run prisma:dev -- --name descriptive_change
   ```
3. Review and adjust `prisma/migrations/<timestamp>_<name>/migration.sql`. Add manual SQL for views/triggers if needed.
4. Commit the migration folder plus any schema changes.
5. Open a PR, review diff (`npm run prisma:diff:dev->prod` for preview).
6. Merge to `main` and let CI deploy to production.

Avoid `prisma db push`—migrate keeps change history and deploys safely.

---

## Useful npm scripts
| Script | Description |
| ------ | ----------- |
| `npm run prisma:pull` | Introspect current DB into `schema.prisma` |
| `npm run prisma:dev` | Compare schema to dev DB, create & apply migration |
| `npm run prisma:deploy` | Apply pending migrations to the target DB |
| `npm run prisma:status` | Inspect migration state vs. DB |
| `npm run prisma:diff:dev->prod` | Preview SQL to bring prod in sync with dev |
| `npm run prisma:diff:prod->dev` | Spot drift between prod and dev |
| `npm run prisma:baseline:sql` | Emit SQL for baseline migration 000_init |
| `npm run prisma:mark:baseline:applied` | Mark migration 000_init as applied without running it |
| `npm run prisma:check` | Lint/format/validate schema and migrations |

Scripts rely on `DATABASE_URL`, `DEV_DATABASE_URL`, and `SHADOW_DATABASE_URL`. Adjust `.env` files or provide env vars when running commands.

---

## CI/CD deployment
- Workflow: `.github/workflows/prisma-deploy.yml`
- Triggers: push to `main`, manual dispatch.
- Steps: checkout → setup Node 20 → `npm ci` → `npm run prisma:deploy`.
- Required GitHub secrets:
  - `PROD_DATABASE_URL` – production connection string.
  - `PROD_SHADOW_DATABASE_URL` – optional but recommended for schema diffing.

Test by running the workflow manually after seeding secrets. Use environments/required reviewers in GitHub for extra safety.

---

## Tips & best practices
- **Backups**: take a DB snapshot before initial baseline or risky migrations.
- **Shadow DB perms**: ensure the shadow user may create/drop schemas (`migrate dev` needs it).
- **Manual SQL**: keep non-Prisma objects (views, triggers, procedures) in migration SQL files.
- **No Prisma Client**: we omit the generator; add it if you plan to use Prisma Client in code.
- **Seeding**: embed SQL in migrations or handle seeding via existing tooling.
- **Secrets**: never commit real credentials; use `.env` locally and GitHub secrets in CI.

---

## Need help?
- Prisma docs: https://www.prisma.io/docs
- CLI reference: `npx prisma --help`
- Open an issue or start a disc in your repo for team-specific workflows.
