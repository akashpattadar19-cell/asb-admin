# ASB LOGISTICS — admin panel and driver web app

- `index.html` — admin panel (email + password sign-in, Supabase)
- `driver/` — driver app (Iqama + PIN). The Android APK bundles the same page.

Data lives in Supabase (project `asb`) behind row-level security: only admins listed in the `admins` table
can read or change records; drivers reach only their own data through PIN-protected functions.
