# Algolia Integration — User Guide

Welcome! This guide walks you through everything you need to **install, configure, and operate** the
**Algolia – App Builder Integration** for Adobe Commerce. It is written for merchants and store
administrators — no developer or command-line knowledge is required.

The integration keeps your Adobe Commerce **products, categories, and CMS pages** in sync with
**Algolia** search indices automatically. Once it is set up, day-to-day catalog changes flow to
Algolia on their own; you only return to the app when you want to change how data is indexed or run
a manual synchronization.

> Looking for the architecture, event flow, and developer details instead? See the
> [project `README.md`](../../README.md).

---

## The journey at a glance

Setting the app up is a one-time, six-step process. After that you mostly let it run.

| Step | What you do | Where |
| ---- | ----------- | ----- |
| 1 | **Install** the app from Adobe Exchange, create an App Builder environment, then associate and **install** it on the instance you want to connect | Adobe Exchange → ACCS/PaaS Console → **App Management** |
| 2 | **Configure** Algolia credentials and general settings | Commerce Admin → **App Management → Configure** |
| 3 | Open the dashboard and click **Initialize Configuration** | Commerce Admin → **Apps → Algolia Configuration** |
| 4 | Map your **stores to Algolia indices** and set store URLs | Dashboard → **Stores** tab |
| 5 | Choose **product / category attributes, facets, sorting, ranking** | Dashboard → **Products** / **Categories** tabs |
| 6 | Push the configuration to Algolia, then run a **full sync** | Dashboard → **Full Sync** tab |

From then on, the app **listens for Commerce events** and keeps Algolia up to date in near real time,
with nightly jobs reconciling everything. See [How synchronization works](./10-how-synchronization-works.md).

---

## Contents

**Getting started**

1. [Installation & connecting your instance](./01-installation.md)
2. [Configuring the app in App Management](./02-app-management-configuration.md)
3. [Initializing the dashboard](./03-initialization.md)

**Configuring how data is indexed**

4. [Stores — index mapping & URLs](./04-stores.md)
5. [Products — attributes, facets, sorting & ranking](./05-products.md)
6. [Categories — attributes & ranking](./06-categories.md)
7. [CMS pages](./07-cms-pages.md)

**Running & monitoring synchronization**

8. [Full Sync — pushing config & syncing data](./08-full-sync.md)
9. [Queue Monitor — tracking sync status](./09-queue-monitor.md)
10. [How synchronization works (events & scheduled jobs)](./10-how-synchronization-works.md)

**Everyday use**

11. [Working with the app — the day-to-day process](./11-working-with-the-app.md)
12. [Troubleshooting & FAQ](./12-troubleshooting.md)

---

## Key concepts in one minute

- **Two places to configure.** Credentials and a few global flags live in **App Management** (Commerce
  Admin). Everything about *how* data is indexed (indices, attributes, facets, sorting, ranking) lives
  in the **Algolia Configuration dashboard** under *Apps*.
- **Events keep things current.** When you save or delete a product, category, or CMS page in Commerce,
  the change is queued and pushed to Algolia automatically — usually within a few minutes.
- **Scheduled jobs reconcile.** Once a day a full sync re-checks the whole catalog and removes anything
  in Algolia that no longer exists in Commerce.
- **Settings changes need a push.** Whenever you change attributes, facets, sorting, or ranking, you
  must click **Sync Indexes Configuration With Algolia** so Algolia picks up the new index settings.
  See [Working with the app](./11-working-with-the-app.md).
- **The Queue Monitor is your window.** Every product, category, and CMS page sync passes through a
  queue you can inspect at any time.

> 📸 **Screenshots:** This guide uses screenshots stored in [`images/`](./images/). If you are
> maintaining the docs, see [`images/README.md`](./images/README.md) for the capture checklist.
