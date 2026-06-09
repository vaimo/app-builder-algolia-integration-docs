# 12. Troubleshooting & FAQ

Common questions and fixes when working with the Algolia Integration.

---

## Setup & access

**I don't see "Apps → Algolia Configuration" in Commerce Admin.**
- Make sure you clicked **Install** in [App Management](./02-app-management-configuration.md) and that
  installation finished. The dashboard menu is registered during installation.
- The menu is provided by the Adobe **Admin UI SDK**; try a hard refresh of the admin, or re-open the
  admin after a few minutes.

**The dashboard keeps showing the "Initialize Configuration" screen.**
- That screen appears until initialization completes successfully. Click **Initialize Configuration**
  again — it's safe to retry and won't overwrite anything. See [chapter 3](./03-initialization.md).

**Initialization fails.**
- It reads stores/websites from Commerce, so it needs a valid **Commerce URL** and working
  authentication. Double-check the URL format (PaaS needs the `/rest/` suffix) in
  [App Management](./02-app-management-configuration.md), and for **PaaS** confirm the IMS integration and
  technical-account admin user are set up.

---

## Data not appearing in Algolia

**A product/category change hasn't shown up in Algolia.**
- Allow a few minutes — queued changes are processed every 5 minutes.
- Check the [Queue Monitor](./09-queue-monitor.md): filter by the **Entity ID** to see its status.
  - **Pending / Processing** → still in progress, just wait.
  - **Retry** → transient error, it will retry automatically.
  - **Failed** → click the **Error Message** for details.
- Confirm the store is **Enabled** for that entity type on the [Stores](./04-stores.md) tab.

**Nothing syncs at all.**
- Verify your **Algolia App ID** and **Admin API Key** in App Management — an Admin (write) key is
  required, not a Search-only key.
- Verify the **Commerce URL** and authentication.
- Run a manual **full sync** from the [Full Sync](./08-full-sync.md) tab and watch the Queue Monitor.

**Hash-based change detection note:** if an entity's data hasn't actually changed, the app **skips** it to
avoid redundant Algolia calls. A "skipped" item is expected behavior, not an error.

---

## Configuration not taking effect

**I changed facets / sorting / ranking but search looks the same.**
- You almost certainly need to click **Sync Indexes Configuration With Algolia** on the
  [Full Sync](./08-full-sync.md) tab. Saving alone does not push settings to Algolia. See
  [Working with the app](./11-working-with-the-app.md).

**Product attribute values show as IDs or are missing labels.**
- The **option-label cache** is out of date. Click **Sync Attributes** on the Full Sync tab (or wait for
  the daily 03:00 UTC refresh). This applies to **select / multi-select** attributes whose options
  changed.

**My per-store settings aren't applying.**
- On the Products/Categories tab, make sure **Use Default Configuration** is **unchecked** for that store
  view before editing, and that you clicked **Save for `<store>` store**. Then push with **Sync Indexes
  Configuration With Algolia**.

---

## CMS pages

**Sync CMS Pages does nothing / CMS pages don't appear.**
- CMS page sync is **PaaS-only**. On **SaaS / ACCS** it is not available because the platform doesn't emit
  CMS events or expose CMS REST endpoints. See [CMS pages](./07-cms-pages.md).

---

## Queue & cleanup

**The Queue Monitor shows old Success/Failed rows.**
- Terminal rows are pruned automatically by the nightly cleanup job: **Success** after 7 days, **Failed**
  after 30 days. You don't need to clear them manually.

**An item is stuck in Processing.**
- Items stuck in **Processing** for over an hour are automatically re-queued on the next batch — no action
  needed.

**Many Failed items.**
- Open a few **Error Messages** to find the common cause (often credentials, an unreachable Commerce URL,
  or an Algolia limit). Fix the root cause, then run a manual **full sync** to reprocess.

---

## FAQ

**Do I need to run a full sync regularly?**
No. Events keep Algolia current, and a full sync runs automatically every night. Run a manual full sync
only for first-time setup, bulk imports, or after changing index/store mappings.

**Where do I put my Algolia credentials — the dashboard or App Management?**
**App Management.** Credentials and global flags live there; the dashboard only configures *how* data is
indexed. See [chapter 2](./02-app-management-configuration.md).

**Do I configure Commerce API credentials anywhere?**
No. Commerce calls use Adobe IMS (OAuth Server-to-Server), injected automatically. PaaS instances need the
IMS integration enabled in Commerce; SaaS/ACCS needs nothing extra.

**Can multiple store views share one Algolia index?**
Yes — give them the same index name on the [Stores](./04-stores.md) tab. Use distinct names for fully
separate search experiences.

**What time zone are the scheduled jobs in?**
UTC. Full sync at 03:00 UTC, queue cleanup at 02:00 UTC, batch processing every 5 minutes.

---

## Still stuck?

For deeper diagnostics (logs, traces), administrators can enable **New Relic** forwarding via the
**New Relic License Key** in [App Management](./02-app-management-configuration.md). For architecture and
developer-level detail, see the project [`README.md`](../../README.md) and
[`SYSTEM_DESIGN.md`](../SYSTEM_DESIGN.md).
