# 7. CMS pages

The integration can synchronize **CMS pages** to Algolia so they appear in search alongside products
and categories. CMS page support depends on your deployment type.

---

## ⚠️ Availability: PaaS only

| Deployment | CMS page sync |
| ---------- | ------------- |
| **PaaS** (on-premise / Adobe Commerce Cloud) | ✅ Supported |
| **SaaS / ACCS** (Adobe Commerce as a Cloud Service) | ❌ **Not available** |

**Why it's not available on SaaS/ACCS:**

- ACCS does **not** emit the Commerce CMS page events (`cms_page_save_after` /
  `cms_page_delete_after`), so real-time CMS page sync cannot be subscribed. (These event subscriptions
  are intentionally disabled in the app so that installation succeeds on ACCS.)
- The SaaS REST API does **not** expose CMS page endpoints, so the manual / scheduled CMS full sync has
  no data source to read from either.

On a SaaS/ACCS instance you can leave the CMS Pages index configured, but the **Sync CMS Pages** action
will not produce results. Products and categories are unaffected.

---

## How CMS pages are configured (PaaS)

There is no dedicated CMS tab. CMS page indexing is configured and run from the tabs you already know:

- **Index name & Enabled toggle** — set on the [Stores](./04-stores.md) tab, under **CMS Pages Index**.
- **Manual sync** — run **Sync CMS Pages** on the [Full Sync](./08-full-sync.md) tab.
- **Automatic sync** — on PaaS, CMS page changes flow through Commerce events automatically, and the
  nightly full sync reconciles them. See [How synchronization works](./10-how-synchronization-works.md).

➡️ Continue to **[8. Full Sync](./08-full-sync.md)**.
