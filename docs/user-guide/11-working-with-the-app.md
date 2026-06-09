# 11. Working with the app — the day-to-day process

This chapter ties everything together into a repeatable process: how to set the app up the first time,
and how to work with it from then on. **If you read only one chapter, read this one.**

---

## The golden rules

1. **Configuration is two-step: Save → Sync.** Saving on the Products/Categories tabs stores your
   settings in the app. They only reach Algolia when you click **Sync Indexes Configuration With
   Algolia** on the [Full Sync](./08-full-sync.md) tab. **Always do both.**
2. **Data flows on its own.** Once set up, products/categories (and CMS pages on PaaS) sync automatically
   via Commerce events, with a nightly full sync as a safety net. You don't need to run a full sync for
   everyday edits.
3. **Option changes need an attribute sync.** New or renamed **select / multi-select** options require
   **Sync Attributes** (or wait for the nightly refresh) so labels index correctly.
4. **The Queue Monitor is the source of truth** for what's syncing and what failed.

---

## First-time setup — end-to-end checklist

Do these once, in order:

- [ ] **1. Install from Adobe Exchange**, create an App Builder environment, then associate and **install**
      the app on your instance via **App Management** (sets up events, the dashboard, and the database)
      → [chapter 1](./01-installation.md)
- [ ] **2. Configure credentials in App Management** — Algolia App ID, Admin API Key, Commerce URL (+ optional
      flags) → [chapter 2](./02-app-management-configuration.md)
- [ ] **3. Open Apps → Algolia Configuration** and click **Initialize Configuration**
      → [chapter 3](./03-initialization.md)
- [ ] **4. Stores tab** — review/adjust **index names**, enable the stores you want, set **store URLs**
      → [chapter 4](./04-stores.md)
- [ ] **5. Products tab** — configure **attributes, facets, sorting, ranking**; **Save** each section
      → [chapter 5](./05-products.md)
- [ ] **6. Categories tab** — configure **attributes, ranking**; **Save** each section
      → [chapter 6](./06-categories.md)
- [ ] **7. Full Sync tab → Sync Indexes Configuration With Algolia** (pushes all your settings to Algolia)
- [ ] **8. Full Sync tab → Sync Attributes** (builds the option-label cache)
- [ ] **9. Full Sync tab → Sync Products / Sync Categories** (and **Sync CMS Pages** on PaaS) to populate
      the indices for the first time
- [ ] **10. Queue Monitor tab** — watch items move to **Success**
      → [chapter 9](./09-queue-monitor.md)

After this, your indices are populated and the app maintains them automatically.

> 💡 **Recommended order matters:** push **index configuration** and run **Sync Attributes** *before* the
> first data sync, so products are indexed with the right settings and readable option labels from the
> start.

---

## Everyday operation

Once set up, most days require **nothing** from you:

- Editing products, categories, or (PaaS) CMS pages in Commerce → synced automatically within minutes.
- Overnight, the app runs a **full reconciliation (03:00 UTC)**, refreshes the **attribute cache**, and
  **cleans up the queue (02:00 UTC)**.

You only step in to change *how* things are indexed, or to force an immediate update.

---

## "If you change X, do Y" — quick reference

| If you change… | …then do this |
| -------------- | ------------- |
| A product / category / CMS page (normal edit) | Nothing — it syncs automatically. |
| **Attributes, Facets, Sorting, or Ranking** (Products/Categories tabs) | **Save**, then **Sync Indexes Configuration With Algolia**. |
| A **select / multi-select option** (new color, size, etc.) | **Sync Attributes** (or wait for 03:00 UTC). |
| **Index name** or **Enabled** toggle (Stores tab) | Save, then **Sync Products / Categories / CMS Pages** to (re)populate. |
| **Store URL** (Stores tab) | Save; new URLs apply on the next sync. |
| **Algolia / general settings** (App Management) | Save in App Management. |
| Performed a **bulk catalog import** | Run a manual **full sync** for the affected entity type(s). |

---

## Maintenance & monitoring habits

- After **any** Products/Categories settings change, get in the habit of clicking **Sync Indexes
  Configuration With Algolia** immediately — it's the single most common thing people forget.
- Glance at the **Queue Monitor** after big changes; filter by **Status = Failed** to catch problems early.
- If a sync looks wrong, a manual **full sync** of that entity type is always a safe reset — it reconciles
  and removes orphans.

➡️ If something isn't working as expected, see **[12. Troubleshooting & FAQ](./12-troubleshooting.md)**.
