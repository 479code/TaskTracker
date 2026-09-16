# 479Code Task Tracker — v2

Firebase (Auth + Firestore) task tracker and invoice generator for 479Code.
Static site, no build step — same deploy model as before (GitHub Pages /
Firebase Hosting).

## What changed from v1

| | v1 | v2 |
|---|---|---|
| Access control | client-side `role==='owner'` checks only | enforced in `firestore.rules` — the database itself rejects unauthorized writes |
| Membership | a `members` map field on the project doc | `projects/{id}/members/{uid}` subcollection (no more read-modify-write races on invite/remove) |
| Dashboard counts | fetched every task in every project on every load | cached `counts` field on the project doc, updated by aggregation queries after task writes |
| Live updates | manual reload after every action, notifications polled every 30s | `onSnapshot` realtime listeners throughout |
| Invoice numbers | `INV/MM/YYYY`, collided if two invoices made in the same month | atomic counter via a Firestore transaction |
| Invoice access | hidden nav button if your email ≠ owner | hidden button **and** rejected at the database if you're not the owner |
| Legacy files | `apps-script-backend.gs`, `legacy-sheets-version.html`, stale README | removed |

## Deploy order (important)

1. **Back up your Firestore data first** (Firebase console → Firestore → Export, or just export via `gcloud firestore export`).
2. Sign in to the *old* `index.html` one more time to make sure nothing is mid-write.
3. Open **`migrate.html`** in a browser (locally is fine — it talks straight to Firestore) and sign in as an account that owns every project. It converts:
   - each project's `members` map → `members` subcollection docs, and computes the initial `counts`
   - `notifications` snake_case fields → camelCase
   - `project_invites` → deterministic IDs (`{projectId}_{invitedUid}`)

   This step must run **before** the new rules are live, since the new rules
   don't recognize the old shapes as valid writes.
4. Deploy the new rules and indexes:
   ```
   firebase deploy --only firestore:rules,firestore:indexes
   ```
5. In the Firebase console, confirm **collection group queries** are enabled
   for `members` (Firestore → Indexes → the collection-group index in
   `firestore.indexes.json` will prompt for this on first deploy).
6. Replace `index.html`, `sw.js`, `manifest.json` on your host with the new
   versions here. Delete `apps-script-backend.gs`, `legacy-sheets-version.html`,
   `migrate.html` from the live site once migration is confirmed working.

## Files

- `index.html` — the app
- `firestore.rules` — server-side access control (read the comments at the top — explains the membership model)
- `firestore.indexes.json` / `firebase.json` — deploy config
- `migrate.html` — one-time data migration tool, **delete after running**
- `manifest.json`, `sw.js`, `icons/` — PWA shell

## Known trade-offs, on purpose

- Invoice access is still gated by a single hardcoded email
  (`479code@gmail.com`) rather than a roles collection — matches how it
  worked before, now enforced in rules too. If invoicing needs to extend to
  more than one person later, swap that check for a `roles/{uid}` doc lookup.
- The "breached" (overdue) count per project is computed with a
  `where('dueDate','<', today)` query filtered client-side by status,
  rather than a second aggregation query — Firestore doesn't support two
  inequality filters on different fields in one query. Fine at this scale;
  revisit if a single project accumulates thousands of overdue tasks.
