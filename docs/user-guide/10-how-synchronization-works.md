# 10. How synchronization works

You rarely need to think about *how* data gets to Algolia — but understanding it helps you know **when a
change will appear** and **when you need to act manually**. There are two engines: **real-time events**
and **scheduled jobs**.

---

## Real-time: Commerce events (the everyday path)

During normal operation the app simply **listens for events from Adobe Commerce**. When you create,
update, or delete a **product** or **category** (and **CMS pages on PaaS**), Commerce emits an event, and
the app:

1. Receives the event and adds the affected entity to its **processing queue**.
2. Every **5 minutes**, a background job gathers everything pending in the queue, de-duplicates repeated
   changes to the same entity (the **last** change wins), and processes them in batches.
3. The entity is fetched from Commerce, transformed into the Algolia format, and pushed to your indices.

**What this means for you:** routine catalog edits show up in Algolia **within a few minutes**, with no
action required.

> ⏱️ **Timing:** queued changes are picked up every 5 minutes, so expect up to a few minutes of delay
> between saving in Commerce and seeing the change in Algolia.

---

## Scheduled jobs (the safety net)

Three jobs run automatically on a schedule. All times are **UTC**.

| Job | Schedule | What it does |
| --- | -------- | ------------ |
| **Batch processing** | Every **5 minutes** | Processes queued product / category / CMS page changes (the engine behind real-time sync, above). |
| **Daily full sync** | **03:00 UTC**, daily | Re-syncs **all** products, categories, CMS pages, **and** the attribute option cache. Reconciles the whole catalog and removes orphaned records from Algolia. |
| **Queue cleanup** | **02:00 UTC**, daily | Prunes the queue: removes **Success** records older than **7 days** and **Failed** records older than **30 days**, keeping the [Queue Monitor](./09-queue-monitor.md) fast. |

### The daily full sync (03:00 UTC)

Once a day the app performs a complete reconciliation for every store:

- **Sync phase** — fetches all eligible entities from Commerce (enabled & visible products, active
  categories, active CMS pages) and queues them for upsert.
- **Cleanup phase** — compares what's currently in Algolia against what exists in Commerce and queues any
  **orphans** (deleted or no-longer-eligible entities) for removal from Algolia.
- **Attribute cache refresh** — rebuilds the **select / multi-select option-label cache** so product
  attribute values are indexed with the correct labels.

This is what makes the integration self-healing: even if an event is ever missed, the nightly full sync
brings Algolia back in line with Commerce.

---

## Reliability features (automatic)

You get these without configuring anything:

- **Automatic retries.** A failed operation is retried up to **5 times** before it is marked **Failed**.
  Events that keep failing are skipped after 5 attempts to avoid blocking the queue.
- **Stuck-item recovery.** If an item is stuck in **Processing** for more than an hour, it's automatically
  re-queued in the next batch.
- **Hash-based change detection.** Before sending an entity to Algolia, the app compares a fingerprint of
  its data against the last synced version. **Unchanged entities are skipped**, which avoids unnecessary
  Algolia operations and keeps your usage down.
- **Per-store fan-out.** A change to a "default scope" entity is propagated to **all** the store views
  configured for it.

---

## When *do* you need to act manually?

The scheduled jobs cover most cases eventually, but you should act manually when you want a change
reflected **immediately** or when the change is to **configuration** rather than data:

| You changed… | Do this |
| ------------ | ------- |
| Product/category **attributes, facets, sorting, ranking** | Click **Sync Indexes Configuration With Algolia** ([Full Sync](./08-full-sync.md)). |
| **Select / multi-select options** (e.g. a new color) | Click **Sync Attributes** ([Full Sync](./08-full-sync.md)) — or wait for the 03:00 UTC refresh. |
| **Index ↔ store mapping** ([Stores](./04-stores.md)) | Run the relevant **Sync Products / Categories / CMS Pages**. |
| Did a **bulk import** and don't want to wait | Run a manual **full sync** for the affected entity type(s). |

See [Working with the app](./11-working-with-the-app.md) for the recommended end-to-end process.

➡️ Continue to **[11. Working with the app](./11-working-with-the-app.md)**.
