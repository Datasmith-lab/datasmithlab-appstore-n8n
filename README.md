# Track App Store and Google Play changes with n8n and Google Sheets

An inactive starter from DataSmithLab. Keep the latest version, price, rating and chart position of selected iOS and Android apps in your own Google Sheet, and log each change as it happens.

[App Store & Google Play Tracker on Apify](https://apify.com/datasmithlab/app-store-play-tracker) · [Walkthrough](https://datasmith-lab.github.io/datasmithlab-appstore-n8n/) · [Setup guide with real data](https://datasmith-lab.github.io/datasmithlab-appstore-n8n/track-app-store-google-play-changes-google-sheets-apify-n8n.html) · [Download starter ZIP](https://github.com/Datasmith-lab/datasmithlab-appstore-n8n/releases/latest/download/app-changes-to-sheets-starter.zip)

## What it does

1. Starts one tracker run on Apify and waits for it to finish.
2. Checks the run's HEALTH report (failed lookups, blocked responses, row count) before writing anything.
3. Upserts one row per app, store and country into an **Apps** tab (the latest state).
4. Appends a row to a **Changes** tab for every app that changed (new version, price, rating, chart position) or disappeared from a country's store.

The first run returns every app as `NEW`: that is your baseline. Later runs return only new or changed apps, so a quiet day writes nothing.

## Setup

Requires your own Apify account, n8n with HTTP Request, Code, Wait and Google Sheets nodes, and a Google account with spreadsheet edit access.

1. Create two tabs in your spreadsheet. **Apps**: paste the single header row from `apps-sheet-headers.csv` into A1. **Changes**: paste the row from `changes-sheet-headers.csv`.
2. Import `app-changes-to-sheets.n8n.json`. It is inactive and contains no credentials.
3. In Configuration replace `REPLACE_WITH_SPREADSHEET_ID` with your own spreadsheet ID and list your apps: App Store ids or links and Google Play package names (1 to 25 apps, 1 to 5 countries).
4. Create an n8n **Header Auth** credential: header name `Authorization`, value `Bearer YOUR_APIFY_TOKEN`. Select it on every Apify HTTP Request node. Keep the token in the credential manager, never in a public workflow or Sheet.
5. Connect your own Google Sheets OAuth2 credential on **Save apps** and **Log changes**.
6. Run manually and check the Apps tab (every app `NEW`). Run again: the second run should save no rows.
7. Only after both runs look right, activate the **daily 09:00 UTC** schedule. Self-hosted n8n must stay running. Run one execution at a time.

The default tracks Instagram on both stores in the United States. The configured Actor charge cap is $0.10 per run. Current Store prices are authoritative: the Actor has a start fee and a per-row fee, and unchanged apps are not charged as rows. This cap is not an end-to-end n8n, Google or cloud cost guarantee.

## Columns

**Apps:** `id`, `store`, `appId`, `country`, `name`, `developer`, `category`, `version`, `lastUpdated`, `price`, `currency`, `ratingAverage`, `ratingCount`, `ranks`, `changeType`, `changes`, `diff`, `url`, `savedAt`, `runId`.

**Changes:** `eventKey`, `id`, `name`, `store`, `country`, `changeType`, `changes`, `diff`, `savedAt`, `runId`. `diff` keeps the old and new values as JSON in one cell.

## Failures and recovery

A failed or blocked lookup, a row-count mismatch, a reached charge limit, a duplicate app ID or an unexpected record stops the workflow before any write. Inspect the run on Apify before starting another.

The Actor remembers what it last reported for each app and country, shared by every task in your Apify account. If a run succeeded on Apify but the Sheet write failed, set `onlyNewOrChanged` to `false` for one run: every app comes back and **Apps** is refreshed. The change events of the failed run are not recreated. A Google Sheets upsert can partially complete; this is not a transactional delivery guarantee.

## Scope

Charts cover Apple's official top 100 Top Free and Top Paid per country only. Google Play charts and reviews, keyword rankings and ASO analysis are not included, and review rows are not stored by this starter. Google Play apps that ship different builds per device may show no version; they are tracked by update date. `REMOVED` means the store no longer lists the app in that country, not that it was withdrawn. Ratings are shown as each store reports them and are not directly comparable across stores.

## Validation

- **Real API replay:** on 12 October 2026 the workflow's exact Apify calls (start, poll, read HEALTH, read dataset) were replayed against the live Actor with a fresh app pair (Duolingo on both stores). The first call returned 2 `NEW` rows with matching HEALTH and no failed or blocked lookups. A second call returned 0 rows. Run IDs: `NwFjSWPsh4LrRzhWN` and `wB4GMmFRc67iP3JKq`.
- **Logic replay:** Prepare rows was replayed on real baseline output (5 `NEW` rows for three iOS and two Google Play apps) and on a real empty second run. Synthetic `CHANGED` and `REMOVED` rows were turned into events; no real version, price or rating change was observed during testing. Failure paths (failed lookup, blocked response, count mismatch, charge limit, duplicate ID, review record, non-HTTP URL) all stopped without rows. 31 of 31 checks passed.
- **Import:** the inactive workflow imported into a clean n8n 2.26.4 instance.
- **Live Google Sheets run:** on 12 October 2026 the complete workflow ran on a self-hosted n8n 2.26.4 against a real Google Sheet and the live Actor (Netflix on both stores, fresh tracking scope). The first run saved 2 `NEW` rows (run `nfvLRQzmx6XbTSZoe`), a second run returned 0 rows (`inTplVUn3m0MsRDRX`), and a read-back showed 2 unique app rows in Apps and none in Changes. The Changes upsert was verified live with two synthetic events (a version and rating change and a removal) written twice: 2 rows, no duplicates. No real version, price or rating change was observed.
- **Not tested:** a scheduled (timer-triggered) run of this exact workflow. The structure follows the DataSmithLab ATS and Zillow starters, whose schedules were verified on 11 October 2026. Run it manually first and inspect both tabs.

Other n8n versions and cloud instances need your own setup check. This package has not been accepted into the official n8n template directory. Independent of Apple and Google; no affiliation claimed.
