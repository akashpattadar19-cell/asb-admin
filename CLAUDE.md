# ASB LOGISTICS — project handoff (read this first)

Delivery-company employee/driver management for a Saudi company. The owner speaks Bengali; admin UI has English / Bengali / Arabic.
No secrets are stored in this file. Never paste service keys, passwords or tokens in chat.

## Parts
| Part | Where | Notes |
|---|---|---|
| Admin panel | `index.html` (single file, vanilla JS) | hosted by GitHub Pages at `asblogistics.github.io` |
| Driver web app | `driver/index.html` | same hosting; also bundled inside the Android app |
| Android APK | private repo `akashpattadar19-cell/asb-driver-app` | WebView shell + notifications; CI builds a signed APK on every push to `main` |
| Backend | Supabase project `hpkfwmlauxjompovpuvq` | Postgres + RLS + SECURITY DEFINER functions. No server code |
| Code repo | `akashpattadar19-cell/asb-admin` (this repo, public) | branches `main` and `gh-pages`; mirror site repo `asblogistics/asblogistics.github.io` |

The Supabase anon key inside the code is public by design. Security = RLS + token-checked functions.

## Data model (Supabase)
- Generic table `docs(coll text, id text, data jsonb)`: employees, payroll, totals, daily, fuel, perf, orddetail, fines, advances, ledger, fuelneed, internet, monthly (archives), notices, config (rules, month, settings), expense, invoice, duty, presence …
- `admins(email, role, name, perms jsonb)`; roles SUPER_ADMIN / SUB_ADMIN / VIEWER. Sub-admin edit limits (`perms.edit`) are enforced in the UI only; VIEWER writes are blocked in the DB (`can_write()`).
- Other tables: driver_sessions, audit_log, audit_questions, admin_logins, login_attempts, loc_last.
- RLS is on everywhere. `docs` is admin-only. Anon has no table access. **Drivers only use RPCs** that check a PIN session token: `driver_login`, `driver_data` (returns everything a driver sees), `driver_update_profile`, `driver_notice_peek`, `driver_fuel_need`, `driver_set_duty`, `driver_audit`, … Admin RPCs: `admin_set_login`, `admin_delete_notice`, `audit_delete`, `admin_backup`, `my_admin_role`.
- Full schema/function/policy code is exported by the admin panel: Admins page → "Download backup" (super admin). The JSON has `docs`, `admins` and `schema_sql`.

## Business rules (don't break)
- Pay cycle runs the 2nd → 1st, keyed by start month. Manual month switch: dashboard "Close month" writes `monthly/<YYYY-MM>` and sets `config/month.cur`. `monthKey()` uses it.
- Rank vs Unrank pay ladders (380 / 500 / 600 orders) come from `config/rules`; see `planOf`, `payFor`, `fuelAllowance`.
- Performance and Rank stay locked in the driver app until 100 orders in the month.
- Dues = `fines` (category field) + `advances`; `history` array on each doc is the deduction record; `ledger/*` holds one DEDUCT or REVERSE entry per change.
- Payroll deductions: ADVANCE / COMPANY / ACCIDENT / TRAFFIC columns are linked to Dues (cannot exceed what is left; applied on Publish; reversed on Unpublish; double publish blocked). PETROL, SUPERVISOR, KEETA are payroll-only. Nothing is deducted automatically — all by hand at month end.
- Imports (fuel, order details, performance) are staged: test preview → Submit / Reset. Nothing reaches drivers until Submit.
- Drivers cannot edit name or Iqama; vehicle model is Suzuki/Hyundai; petrol card is `logistic <1-99>`.
- Employee fields: `isActive` (working / not working; blocks fuel request), `inPayroll` (false = left out of payroll; `payroll/<m>.extra` lists manual additions).

## Working on the code
- Verify before deploy: extract the big `<script>` and run `node --check`; also run it once with DOM/supabase stubs to catch top-level runtime errors (a TDZ bug once froze the admin on "Loading…").
- i18n: three dicts `I18N.en/bn/ar`; new keys go in `Object.assign(I18N.xx, {...})` blocks. Every key must exist in all three. Later blocks override earlier ones.
- Patching a DB function: `do $$ declare f text; begin f := pg_get_functiondef('public.driver_data(text)'::regprocedure); f := replace(f, OLD, NEW); execute f; end $$;` — check the replace really changed it.
- Deploy: a script copies this folder to `asblogistics.github.io`, replaces `__BUILD__` with a timestamp, writes `v.txt` and `driver/v.txt`, pushes `main` + `gh-pages`. Equivalent manual steps: commit and push this repo to `main` and `gh-pages`; copy files to the site repo, stamp the build id, push.
- The APK repo: push to `main` → GitHub Actions builds `ASB-Driver.apk` into a Release. It contains a copy of `driver/index.html`; keep it in sync. `NoticeService` (foreground, checks every minute) + `NoticeJob` (15-minute fallback) show notifications; Firebase push is not set up yet.

## Recovery — if the Claude account is lost
1. Code is safe in GitHub (repos above). Clone them. Data and functions are safe in Supabase.
2. In a new Claude account: connect the GitHub and Supabase connectors (sign in as the owner), clone `asb-admin` and `asb-driver-app`, and say: "Read CLAUDE.md and continue the ASB project".
3. If Supabase is ever lost: create a project, run `schema_sql` from a backup file, restore `docs` and `admins` with Admins page → "Restore from backup" (or upsert the JSON), then put the new project URL and anon key into `index.html`, `driver/index.html` and the APK `NoticeJob.java`, and redeploy.
4. Keep these safe yourself: the GitHub login, the Supabase login, a recent backup file, and the APK signing key (`app/asb-release.jks` in the private APK repo) — without that key new APKs cannot update installed ones.

## Publishing a new APK to drivers
After the CI build for a push to `main` succeeds, run `bash /home/claude/publish_apk.sh "notes"` (copies the release APK to the public site `app/ASB-Driver.apk` and bumps `app/version.json`). The installed app checks that file on every open/resume (every 20 s at most) and shows a mandatory "Update" dialog when its versionCode is lower.
