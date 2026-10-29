# Upgrade this instance to OpenShortLink v0.11.0

## Scope

- Base: `733eef2998c33879a87f0b6f2a1e91c6ae0f4148` (imported May 17, 2026).
- Upstream release: `idhamsy/openshortlink` tag `v0.11.0`, commit `7e05244381ab8c02f4569b7c05729c5429ce0e7f`.
- Preserve `wrangler.toml` byte-for-byte, including the existing Worker, D1, KV, Analytics Engine and cron configuration.
- Do not import upstream's unrelated AI issue-summary workflow.
- Commit a dependency lockfile for reproducible installs. Use Node.js 24 LTS (Node 22.13+ also supports the SQLite test helpers).
- Use patched Wrangler 4.155.0, matching Workers types 5.20261029.1, and Vitest 4.1.12. Unbounded upstream Wrangler dependencies currently cause an npm peer conflict with Workers types v4. The older toolchain also reports security advisories.
- Compatibility-only TypeScript changes assert required Hono path parameters inside matching routes and accept only the `waitUntil` part of the execution context in redirect handling; no runtime behavior changes.

## Before merging or deploying

This PR is preparation only. No remote migration, backup, secret update, or deployment has been executed.

1. In Cloudflare, locate the actual Worker and its connected repository. Confirm production branch is `main` and check build, deploy and preview commands. GitHub has no deployment/check records for the base commit. The available Cloudflare token cannot inspect Pages or Workers Builds, and the account visible to it lists no Workers; deployment linkage therefore remains unverified.
2. Ensure preview builds never execute `npm run deploy` or remote migrations against the production DB. Use a separate DB/KV for functional previews, or disable preview builds for this upgrade branch. A preview version sharing production resources is not an isolated staging environment.
3. Record the currently deployed Worker version/commit and export the **existing** D1 database (after authenticating to the correct account):

   ```sh
   npx wrangler d1 export DB --remote --output=./openshortlink-before-v0.11.0.sql
   ```

   Keep the SQL backup private and outside Git. Check the existing migrations ledger before applying changes:

   ```sh
   npx wrangler d1 migrations list DB --remote
   ```

4. Preserve all secrets, especially `SETUP_TOKEN`: it encrypts MFA secrets in newer releases. Do not rerun first-time setup or create replacement D1/KV resources.
5. Review the upstream `CHANGELOG.md`, especially CSV import duplicate handling (`on_existing`), bulk limits (100), analytics normalization and MFA changes.

## Production rollout (requires approval)

Install with `npm ci`, then use **`npm run deploy`**, which applies outstanding migrations before deploying. For Cloudflare Git integration, set the production deploy command to `npm run deploy`; keep remote migrations out of the build command and preview command.

Do not use only `wrangler deploy` or `npm run deploy:only`: schema through `0023_add_pixel_library.sql` is required. Migrations 0018–0023 are new relative to the imported repository. Migration 0022 deduplicates analytics dimension rows and normalizes NULLs; review the backup before running it on real data.

After deployment verify:

- Dashboard login and MFA (if enabled).
- Existing short links, including a cache miss.
- Create/read/update/delete a disposable short link.
- Existing domain routes, destination query parameters and redirects.
- Analytics collection and scheduled aggregation.

## Validation performed during preparation

- Clean dependency installation.
- TypeScript typecheck.
- 37 test files / 269 tests passed.
- `npm audit`: zero reported vulnerabilities in the locked dependency set.
- Wrangler production bundle dry-run, without upload.
- Applied all original migrations to an empty local SQLite DB, then applied 0018–0023; integrity check passed and no foreign-key violations were reported. This is a schema rehearsal, not a production data rehearsal.
- Existing `wrangler.toml` is unchanged.

## Rollback

Retain the pre-upgrade Worker version and database export. Code rollback does not reverse migrations. Do not drop new tables or restore production D1 blindly; assess schema compatibility and any writes made after upgrade. Database restore can discard newer data and requires a separate approved recovery procedure.
