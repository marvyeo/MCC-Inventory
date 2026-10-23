# MCC Inventory — Project State

> **Next chat name: `MCC Inventory ver 3`**
> Claude: your FIRST reply in a new chat must start with "Rename this chat to: **MCC Inventory ver 3**" (you can't rename chats yourself). In your first commit this session, bump the number here by 1.

**State version:** 2 (moved into the repo 2026-10-22 by the conductor chat "Life Plan Roadmap ver 3")
**Where this lives:** `PROJECT_STATE.md` in the repo. New chats `git pull` and read it; no uploads needed.
Receipt Reader's source is the published artifact (link below); read it from there rather than uploading it.

## ⚙ Session Rules (every chat that works on this app follows these)

**1. Pick the cheapest model that can do the job.** At the start of every session Claude names the model it recommends and the estimated size (below). If the chat is on a pricier model than needed, Claude says so before starting.

| Model | Use for |
|---|---|
| **Haiku** | State-file edits, text or label changes, small data packs, lookups, "where is X" questions |
| **Sonnet** (default) | Features, bug fixes, refactors, data processing (e.g. novel volumes, PullList), most app work |
| **Opus** | Only for new-app builds, cross-app architecture, or a bug Sonnet has failed on twice. Claude says why before using it |

**2. Every push updates this file in the same commit.** No exceptions, including PR branches and one-line fixes.
- `git pull` before starting, so you build on the latest state.
- Add a dated line to the change log (what changed and why, in plain words) and update any section the change made wrong.
- If a session ends without a push, nothing changes here.

**3. Size the work first, and plan the pause points.**
- Before starting, Claude lists the steps and sizes the job: **S** (under ~20k tokens), **M** (~20–60k), **L** (over ~60k).
- **M and L jobs are split into checkpoints.** Each checkpoint ends with a commit and a state update, so a cut-off session loses nothing.
- **Pause** when usage is around half the window, when the chat has grown long (heavy file reads, ~30+ turns), or before a step that won't finish in what's left. Claude can't see the remaining credit directly, so on L jobs it asks Marvin to check usage at each checkpoint.
- **On pause,** Claude writes a `▶ RESUME HERE` block at the top of this file: what's done, the exact next step, branch and files, and the first command to run. It also gives Marvin a one-line restart prompt ("Continue <App> ver X from RESUME HERE"). The next session starts from that block and deletes it once the step is done.

**4. Chat naming.** Use the **Next chat name** line at the top of this file; tell Marvin in the first reply, then bump it.

---

## Latest changes (Oct 23, 2026, conductor chat "Life Plan Roadmap ver 7", Sprints 13.4, 13.3, 13.2)
- **Moved to the shared OAuth client** (`669494100250-…`, scope `drive.file`; was `676510183503-…` with full `drive`). `drive.file` only sees files this client created or that the person picked, so on first open the app shows **"Pick the 3 files"** (Google Picker, multi-select; also searches "Shared with me"). It checks access to the three fixed IDs *before* loading and never loads as empty, so a missing grant cannot cause an empty save over real data. **Marvin and Caroline each do this once, signed in as themselves.** After it, Life Plan can find the items file by name (no separate Pick missing files for MCC).
- **Shared sign-in (`lp-sso`):** uses a valid `drive.file` token from Life Plan or another app; publishes its own after sign-in. The old per-app token key `mcc_drive_token` is discarded (`mcc_drive_token_v2` now).
- **Disconnect no longer revokes the Google token** (revoking would cancel the file grants for every app on the shared client). It clears local tokens and `lp-sso`.
- **13.3 (iOS):** "Sign-in window not opening? Try another way" (redirect sign-in, state `mcc`) and an iPhone "Share → Add to Home Screen" hint on the sign-in card. iOS cannot install web apps from an install prompt; it only uses Safari's Add to Home Screen. Needs `https://marvyeo.github.io/MCC-Inventory/` under Authorized redirect URIs on the shared client.
- **13.2:** "← Life Plan" item at the top of the sidebar, `viewport-fit=cover`, safe-area padding on sidebar and main, shared theme tokens `--lp-bg/--lp-surface/--lp-text/--lp-muted/--lp-accent`. `sw.js` cache `mccinventory-v3`.
- Not tested live. Old OAuth client `676510183503-…` is now unused.

## Earlier changes (Oct 21–22, 2026, conductor chat)
- **Installable (21 Oct):** `manifest.json`, network-first `sw.js` (cache `mccinventory-v2`), ecosystem icons. No iPhone redirect sign-in fallback yet (MCC's OAuth client is in its own Google Cloud project, so the redirect URI goes there).
- **Manifest id fixed (22 Oct):** `"id": "/MCC-Inventory/"`.
- **Life Plan** dashboard reads `upcoming[]` (`low-stock`) from the items file. (Superseded 2026-10-23: MCC is on the shared client, so the picker step happens inside MCC.)
- Verified 22 Oct: `DRIVE_FILE_ID_ITEMS`, `_LOCATIONS` and `_PHOTOS` are all filled in the deployed `index.html` (closes to-do 6).

## 1. System overview

There are two apps.

| App | What it does | Where it lives |
|---|---|---|
| **MCC Inventory** | Household inventory: items, locations, batches, expiry, price history, restock | GitHub Pages + Google Drive backend |
| **Receipt Reader** | Reads receipts (images/PDFs) with Claude vision, matches lines to inventory, outputs a changes file | Published artifact: https://claude.ai/artifact/AoQAuh374WZhHuFkxW9zPw (private) |

**End-to-end flow**
1. Open Receipt Reader. It loads **Items**, **Locations** and **Aliases** live from Drive via your Google Drive connector (falls back to the baked snapshot + manual upload if that fails).
2. Upload one or more receipts and run Analyze. Review each line: Existing or New, the qty and unit, the weight link, Add to stock?, and Include.
3. Generate. Two files download separately: one master **receipt-changes JSON** and an **updated aliases JSON**.
4. In MCC Inventory, import the receipt-changes file. Review and edit it line by line, then Apply. This writes to Drive.
5. Upload the aliases JSON to Drive with **Manage versions → Upload new version** on the existing aliases file. This keeps the file ID the same.

---

## 2. MCC Inventory

### Hosting & config
- **Live URL:** https://marvyeo.github.io/MCC-Inventory/
- **Repo:** https://github.com/marvyeo/MCC-Inventory (`index.html`, `manifest.json`, `sw.js`, `icons/`; Pages redeploys in about a minute)
- **OAuth Client ID:** `669494100250-gvromuq72r7j0djmkd4vdbdq5damncj7.apps.googleusercontent.com` (shared Life Plan client since 2026-10-23; was `676510183503-…`). It is public-safe.
- **Authorized JS origin:** `https://marvyeo.github.io`. It has to match exactly. Sign-in fails silently anywhere else, including the artifact preview.
- **Scope:** `https://www.googleapis.com/auth/drive.file` (since 2026-10-23; was full `drive`). Each user picks the three fixed-ID files once with Google Picker, which is how the second person gets access to files the primary user owns. Expect the "unverified app" warning; it is normal. Every user has to be a **Test user** in Google Auth Platform → Audience.
- After editing on GitHub, a stale browser cache can still show the old version. A hard refresh fixes it.

### Drive files (accessed by fixed ID, never by name)
| File | File ID |
|---|---|
| Items JSON | `1qTGUshdGVFfJ3cpcYYLthDSaUyPiyBZ-` |
| Locations JSON | `1idvjvAyK7ow7hxZ9oZdB3F77y7qCGijF` |
| Aliases JSON (used by Receipt Reader) | `1VkQaYWtlc7WMkamPGwrLjUIaZHfzvMHM` |
| Photos JSON (`mcc-inventory-photos.json`, `{itemId: dataURL}`) | `1uVwI26oh-kf7i05aawdcmqdxnwq1Di-z` (created 2026-10-18; set in `DRIVE_FILE_ID_PHOTOS`) |

`DRIVE_FILE_ID_ITEMS` and `DRIVE_FILE_ID_LOCATIONS` are now filled in `index.html` (2026-10-18). The items file stores `{ items, upcoming }` (not a bare array) and, since 2026-10-18, no photos.

Items, locations and aliases are shared by public link (view/download). This was a deliberate choice: "basically the same as letting people see my grocery list."

### Current data size (approx.)
- About 248 items on Drive, 18 locations and about 155 aliases.
- Items JSON: 7.1MB → 160KB after the 2026-10-18 photo split. Photos JSON is about 7MB.
- **Deploys:** Claude pushes changes to GitHub as a PR and merges it (PR #1 speed update, PR #2 photos ID + popup fix).

### Features built
- Dashboard, plus browsing by location, search, and filters (food/non-food, subcategory).
- Items have name, **brand**, food flag, subcategory, location, unit, photo, notes and a **restock target**.
- **Multiple batches** per item, each with its own qty and expiry. Expiry is **auto-estimated** from subcategory when none is given (`estimateExpiry`).
- **Price history** records store, price, qty, date and promo. Stats come from `priceStats`, `normalizePurchasePrice` and `itemUnitLabel`.
- **Price tracking measure** per item: per pack, or by weight/volume ($/kg, $/L). The fields are `priceTrackingBasis`, `packageWeight` and `packageWeightUnit`.
- **Running Low**, **Out of Stock** and **Expired** pages. Out-of-stock items sort last elsewhere.
- **Duplicate item** (does not copy the photo).
- **Backup/restore** (`exportBackup(includePhotos)`, `handleImportBackup`).
- **Import receipt changes**: `handleImportReceiptChanges` opens the `importReview` view, where every line can be edited or excluded, then `applyReviewedImport` applies them.
- Custom toast/modal system replaces native `alert`/`confirm`, which were silently blocked.
- **Background saves** (2026-10-18): edits render instantly; `persistItems()` queues and coalesces Drive writes. Status dot: amber "Saving to Drive…", teal "Synced", rust on failure. Leaving mid-save prompts.
- **Photos in a separate Drive file**, loaded after the inventory appears; only re-uploaded when a photo changes.
- Refresh from Drive (button, plus automatic refresh on tab focus/visibility) now does a metadata `version` check first and only downloads files that changed; it never runs while a save is pending. Connection dot and Disconnect. These sit in the sidebar footer and show icon-only on mobile.
- Missing-location ("⚠ Missing location") warnings flag orphaned items.
- Mobile layout uses the viewport meta tag, an auto-fit grid and card-style tables.
- There are no hardcoded default locations. Locations come only from Drive.

### Locations (stable IDs, examples)
`loc-kitchen-pantry`, `loc-kitchen-fridge`, `loc-master-toilet-cabinet`, … (18 in total). The rest are in the locations JSON on Drive.

---

## 3. Receipt Reader

### Hosting & constraints (updated 2026-10-18)
- **Published artifact** (private): https://claude.ai/artifact/AoQAuh374WZhHuFkxW9zPw. Update it from a chat by passing that URL. Local source: `receipt-reader.html`.
- Capabilities: `mcp` → Google Drive connector, tool `download_file_content` (reads by file ID, returns base64); `sample` with images (Claude vision on the viewer's own account; PDFs are rendered to page images with pdf.js 3.11.174 first); `downloads`.
- Startup: reads locations first (triggers the Drive consent prompt), then aliases + items. A source card shows live / partial / snapshot status with per-file errors; manual dropzones sit in a collapsed "Use different files instead" section.
- **Cannot write to Drive in place**: the connector has no "update file content" tool (only metadata updates, or creating a new file, which would change the ID). So the aliases file is still downloaded and re-uploaded with Manage versions.
- Outside a published view (e.g. the old chat preview) it falls back to the direct API fetch + snapshot + manual upload.
- The old dormant API-key code (`GOOGLE_DRIVE_API_KEY`, `pullFromDrive`) was removed.

### Baked defaults
- `BAKED_AT = '2026-08-16 (refreshed)'`
- `BAKED_ITEMS`: a lean 214-item snapshot. It is stale, since Drive has about 248 items.
- `BAKED_LOCATIONS`, `BAKED_ALIASES`
- Re-baking means pulling the full items JSON (about 5MB) through the Drive connector, which uses a lot of quota. It's cheaper to upload fresh files each session and re-bake only now and then.

### Features
- Several receipts can go in at once (images and PDFs).
- `extractReceipt` schema: `marketplace`, `seller` (chain name only, no outlet), `date|null`, `lineItems` (with `variation` folded into rawName, and `packageWeight`), `orderLevelDiscounts`, `paymentOffsets`.
- **Discounts** (vouchers, shipping discounts, deal savings) are **prorated into item prices**. **Payment offsets** (SG60 vouchers, FairPrice e-vouchers, Shopee coins, gift cards) **do not change prices**; they are only noted in `promo`.
- Matching checks the alias table first (normalized exact match) and falls back to an LLM match (`llmMatchLines`).
- Review table:
  - Shows the existing item's unit next to qty, plus a per-unit price preview (useful for bundles like "4 packs for $54.25").
  - Weight link toggle.
  - Existing/New toggle with a searchable catalog picker.
  - Add to stock? (off means a price-only entry) and Include.
  - Mobile data-labels.
- A receipt date is required. Undated Shopee screenshots need one entered by hand. A location is also required for new items.
- Output is one master changes file plus the aliases file, each with its own download button. Browsers block a second automatic download.

### receipt-changes JSON schema
```json
{
  "source": "receipt-tool",
  "generatedAt": "ISO timestamp",
  "receipts": [{ "marketplace": "Shopee|null", "seller": "FairPrice", "date": "YYYY-MM-DD", "lineCount": 4 }],
  "changes": [
    { "action": "add-batch", "itemId": "<uuid>",
      "batch": { "qty": 1, "expiry": "YYYY-MM-DD" },          // optional; omitted = price-only
      "purchase": { "store": "Shopee — Hooga Official Store", "price": 7.985, "qty": 1,
                    "date": "2026-07-06", "promo": "…",
                    "pricing": { "basis": "pack" } } },
    { "action": "create-item", "tempId": "<uuid>",
      "item": { "name": "Picnic Ham (200g)", "brand": "Fairprice", "isFood": true,
                "subcategory": "Fresh Protein", "locationId": "loc-kitchen-fridge", "unit": "pcs" },
      "purchase": { "…": "…",
                    "pricing": { "basis": "weight", "packageWeight": 200,
                                 "packageWeightUnit": "g", "pricePerBaseUnit": 10.568 } } }
  ]
}
```
Store format: `"<Marketplace> — <Seller>"` for marketplaces, otherwise just the chain name.

### Aliases JSON
It maps normalized receipt text to an item ID. Entries are added only when a line is confirmed against an item.

---

## 4. Known issues / open to-dos
1. ~~"undefined items loaded"~~ **Fixed 2026-10-18**: Receipt Reader now accepts both the `{ items, upcoming }` wrapper MCC saves and a bare array (`toItemsArray`).
2. **The baked snapshot is stale** (214 baked vs about 248 on Drive). Re-bake only when it's worth the quota.
3. **Bundle quantities** (e.g. 4 packs in one line) still need a manual qty fix in review.
4. **Truncated Shopee names** need the catalog picker or manual edits.
5. ~~Receipt Reader isn't published~~ **Done 2026-10-18** (live Drive read). The live items read works now that the items file is 160KB (at 7MB it failed with "connector response exceeded the 16 MB limit"). PDF receipts via pdf.js are untested in the live viewer.
6. ~~Check `DRIVE_FILE_ID_*`~~ **Done 2026-10-22**: all three are set.
7. Claude chat quota: avoid having Claude pull the about-5MB items JSON through the connector. Uploading the files yourself costs nothing.

---

## 5. Lessons learned
- Claude artifact `window.storage` is unreliable for real data. That's why the inventory moved to GitHub Pages + Drive.
- Artifacts can't reach outside APIs (CSP), and OAuth only works from the registered origin. Exception (as of 2026-10): published artifacts can call the viewer's claude.ai connectors (e.g. Google Drive) through the `mcp` capability.
- Native `alert`/`confirm` can be silently blocked, so use custom modals.
- Hidden file inputs triggered by `.click()` are unreliable. Use an `opacity:0` input overlaid on the button. Camera/photo pickers don't work in the Claude mobile app; they do work on the web.
- **Never delete and re-upload a Drive file.** That changes its ID. Use Manage versions.
- Location IDs must be stable, or items get orphaned.
- Users want to **review before applying**: no blind imports, and no clunky extra steps like "export lean copy".
- A full write-up of the hosting pattern is in `google-drive-github-hosting-setup.md`.

---

## 6. How to resume in a new chat
1. `git pull` and read this file. Read Receipt Reader from its artifact link if the task touches it.
2. Say what you want to work on next, e.g. "fix the backup-wrapper issue in Receipt Reader" or "re-bake the items snapshot."
3. If a task needs live data, attach the items, locations and aliases JSONs yourself instead of asking Claude to pull them from Drive.

## 7. Change log
- **2026-10-18:** latency update (background saves, photos split out); Receipt Reader published.
- **ver 2 (2026-10-22):** moved into the repo as `PROJECT_STATE.md`; added Session Rules; recorded installability and the manifest id fix.
- **2026-10-23 (conductor, Sprints 13.4 + 13.3 + 13.2):** shared OAuth client + `drive.file` with a one-time file-pick gate, shared `lp-sso` sign-in, no token revoke on Disconnect, iOS redirect fallback and hint, "← Life Plan" link, safe areas, theme tokens; `sw.js` cache `mccinventory-v3`. Not tested live.
