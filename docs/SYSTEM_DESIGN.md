# Documentation: Event Processing System for Algolia Sync

## System Overview

This document describes the event-driven architecture for synchronizing Adobe Commerce entities (products, categories, CMS pages) to Algolia search index using Adobe I/O Events and Runtime Actions. The system is entity-agnostic, using generic queue collections that can handle any entity type.

MongoDB stores pending entities across three collections:
- `entities_to_sync` — unified queue for both upsert and delete operations, distinguished by `operation_type` field
- `entities_indexed` for tracking what's currently in Algolia
- `entities_cache` for storing derived data (categories, attributes) used during entity building

Scheduled orchestrators (batch-event-system actions) aggregate pending entities and publish batch events to Adobe I/O Events, which automatically trigger internal batch processor Runtime Actions that handle the actual processing asynchronously.

---

## Event Flow Architecture

The system uses a multi-stage event flow pattern:

### Stage 1: Adobe Commerce Events

Adobe Commerce emits events when entities are created, updated, or deleted. The following events are handled:

**Products:**
- `observer.catalog_product_save_after` - Product created or updated
- `observer.catalog_product_delete_commit_after` - Product deleted

**Categories:**
- `observer.catalog_category_save_after` - Category created or updated
- `observer.catalog_category_delete_after` - Category deleted

**CMS Pages:**
- `observer.cms_page_save_after` - CMS page created or updated
- `observer.cms_page_delete_after` - CMS page deleted

These events are sent to Adobe I/O Events and automatically routed to the commerce consumer actions.

### Stage 2: Commerce Event Reception

Commerce consumer actions receive events from Adobe Commerce via Adobe I/O Events. Each entity type has dedicated consumer actions located at `actions-src/events/{entity-type}/commerce/`.

When an event arrives:
1. The consumer validates the event via retry count validation (maximum 5 retries per event)
2. Routes to the appropriate upsert or delete handler based on event type
3. Extracts entity data (entity_id, entity_type)
4. Resolves store configurations to determine which stores need the entity synced
5. For default store (store_id=0) events, entries are created for ALL configured store views
6. Inserts entities into the unified `entities_to_sync` queue with status `PENDING`

**For upsert events (save_after):** Entities are inserted with `operation_type: "upsert"`

**For delete events (delete_after):** Entities are inserted with `operation_type: "delete"`

### Stage 3: Batch Aggregation and Publishing

The batch-event-system orchestrator actions run periodically (configured schedule). These are located at `actions-src/batch-event-system/{entity-type}/`.

The orchestrator performs the following:

1. **Query Pending Entities with Deduplication**
   - Queries the unified `entities_to_sync` collection
   - Filters by status (PENDING, RETRY with retry_attempts < 5, or PROCESSING with created_at older than 1 hour)
   - Filters by created_at with a 5-second cutoff to prevent race conditions with incoming events
   - Includes stuck PROCESSING items (created_at older than 1 hour) for automatic recovery
   - Sorts by created_at (FIFO - oldest first)
   - Groups by (entity_id, store_id, entity_type) to deduplicate multiple events for the same entity
   - Captures the **last** `operation_type` per group (by `created_at` order) — **last operation wins**
   - Collects all MongoDB _ids in a row_ids array for batch status updates

2. **Split by Operation Type**
   - Separates aggregated entities into upsert and delete groups based on `operation_type`
   - This ensures correct ordering: if an entity receives upsert then delete, the delete wins

3. **Mark as Processing**
   - Updates ALL row_ids from aggregation results to status `PROCESSING`
   - Sets updated_at timestamp
   - This prevents the next orchestrator run from picking up the same entities

4. **Publish to Adobe I/O Events**
   - Publishes sync batch events for upsert entities and delete batch events for delete entities
   - Each event contains: batch_id, entities array, row_ids array for MongoDB updates
   - Exits immediately after publishing (doesn't wait for processing)

**Internal Event Codes:**
- `algolia_integration.internal.product.batch.sync`
- `algolia_integration.internal.product.batch.delete`
- `algolia_integration.internal.category.batch.sync`
- `algolia_integration.internal.category.batch.delete`
- `algolia_integration.internal.cms-page.batch.sync`
- `algolia_integration.internal.cms-page.batch.delete`

### Stage 4: Batch Processing

Internal batch consumer actions are automatically triggered by Adobe I/O Events when batch events are published. These are located at `actions-src/events/{entity-type}/internal/consumer/`.

**Sync Processing Flow:**

1. **Receive Batch Event** - Action receives batch_id, entities array, and row_ids

2. **Fetch Entity Data** - Calls Adobe Commerce REST API:
   - For products: Retrieves entities by their IDs from the batch (no status/visibility filters applied to allow fetching disabled/invisible products for proper deletion handling)
   - For categories/CMS pages: Retrieves entities by their IDs from the batch

3. **Categorize Entities** - For each entity in the batch:
   - **Returned by API**: Entity is visible/enabled and should be synced
   - **Not returned by API**: Entity is invisible/disabled or doesn't exist
     - Check `entities_indexed` table
     - If previously indexed: Add to `entities_to_sync` with `operation_type: "delete"` for cleanup
     - If never indexed: Skip (nothing to delete)
     - Mark original queue row as SUCCESS

4. **Build Entity Payload** - Transform entity data to Algolia format:
   - Create objectID in format `{entity_id}`
   - Enrich with cached data (categories, attribute labels)
   - Apply entity-specific transformations

5. **Hash-Based Change Detection** - Compute SHA-256 hash of each entity payload:
   - Compare with previously saved hash
   - Skip unchanged entities (no need to sync)
   - Only sync entities with changed or new hashes

6. **Send to Algolia** - Batch API call using saveObjects

7. **Update Tracking** - On successful Algolia sync:
   - Upsert into `entities_indexed` table
   - Save new entity hashes
   - Update queue status to SUCCESS

8. **Handle Failures** - On failure:
   - Update queue status to FAILED or RETRY
   - Increment retry_attempts
   - Store error_message

**Delete Processing Flow:**

1. Receive batch event with entities to delete
2. Construct objectIDs as `{entity_id}_{store_id}`
3. Delete from Algolia using deleteObjects
4. Remove from `entities_indexed` table
5. Remove stored hashes
6. Update queue status to SUCCESS or FAILED

---

## Queue Collections and Status Management

### entities_to_sync (Unified Queue)

Stores all pending operations (upsert and delete). Multiple rows can exist for the same entity (deduplication happens at processing time). The `operation_type` field distinguishes between upsert and delete operations. During aggregation, the **last operation wins** — if an entity has both upsert and delete events, the most recent one (by `created_at`) determines whether the entity is synced or deleted.

**Fields:**
- entity_id - Adobe Commerce entity ID
- entity_type - Type of entity (product, category, cms-page)
- store_id - Store view ID
- operation_type - Operation type (`"upsert"` or `"delete"`)
- status - Queue status
- created_at - When the event was received
- updated_at - Last status update
- retry_attempts - Number of failed attempts
- error_message - Last error message

### entities_indexed (Tracking Table)

Maintains a 1:1 mapping of entities currently indexed in Algolia. This is the single source of truth for "what's in Algolia".

**Fields:**
- entity_id - Adobe Commerce entity ID
- entity_type - Type of entity
- store_id - Store view ID
- indexed_at - When the entity was first indexed
- updated_at - Last update timestamp

### entities_cache (Cache Table)

Stores derived data used during entity building, such as category hierarchies and attribute option labels.

### Status Values

- **PENDING** - Initial state, waiting to be processed
- **PROCESSING** - Currently being processed by a batch action
- **SUCCESS** - Successfully completed
- **FAILED** - Failed after maximum retry attempts
- **RETRY** - Needs to be retried

---

## Retry Mechanism

### Event-Level Retries

Each Adobe Commerce event has a retry count tracked via temporary storage. Events are validated before processing:
- Maximum 5 retries per event
- If exceeded, the event is skipped with a warning

### Queue-Level Retries

Each queue item tracks retry_attempts:
- When an operation fails, status changes to RETRY and retry_attempts increments
- On the 5th attempt (retry_attempts >= 4), status automatically changes to FAILED
- Failed items are excluded from future orchestrator queries

### Stuck Processing Recovery

Items stuck in PROCESSING status are handled by the orchestrator:
- The aggregation query includes PROCESSING items with created_at older than 1 hour
- Items in PROCESSING for longer than 1 hour are automatically included in the next batch
- This provides automatic recovery without manual intervention or a separate process

---

## Entity-Specific Processing

### Products

Product building is the most complex process as the REST API doesn't contain all necessary information.

**Data Sources:**
1. **Products REST API** - Core product data (name, SKU, description, attributes)
2. **Categories from entities_cache** - Category names and hierarchical paths for faceted navigation
3. **Attributes from entities_cache** - Labels for select/multiselect attribute values
4. **Product Render Info (PaaS)** - Stock and pricing information via products-render-info endpoint
5. **GraphQL Products (SaaS)** - Stock and pricing information via GraphQL API

**Deletion Criteria:**
- Visibility is NOT_VISIBLE_INDIVIDUALLY
- Status is DISABLED

**Partial Batch Processing:**
For SaaS deployments, the products GraphQL endpoint may not return data immediately after changes. The system handles this with partial batch processing:
- Products with complete data are synced immediately
- Products missing render/price info are marked for retry
- A batch might sync 35 products successfully while 15 need retry
- Maximum 5 retry attempts per product

**Build Process:**
- General attributes (name, description, SKU, URL)
- Category data with hierarchical paths (fetched from cache)
- Stock/availability information
- Media attributes (images)
- Price information (base currency for PaaS, multi-currency for SaaS)
- Custom attributes with whitelist support and option label mapping

### Categories

Categories have simpler processing than products.

**Data Sources:**
1. **Categories REST API** - Category data

**Additional Processing:**
- Categories are saved to `entities_cache` after successful sync
- This cache is used by product building to resolve category names and paths

### CMS Pages

CMS pages have the simplest processing.

**Data Sources:**
1. **CMS Pages REST API** - Page content and metadata

**Deletion Criteria:**
- Based on enabled/active status

---

## Full Sync Process

Full sync handles initial catalog indexing or periodic complete reindexing.

### Orchestration Flow

1. Full-sync action is invoked for an entity type (product, category, cms-page)
2. Loads all store configurations
3. For each store:
   - **Sync Phase**: Fetches all entities from Commerce using paginated REST API calls
     - For products: Filters by status (Enabled) and visibility (Catalog, Search, Catalog & Search) in the REST API call to only fetch indexable products
     - For categories: Filters by active status (is_active = 1) in the REST API call to only fetch active categories
     - For CMS pages: Filters by active status (active = 1) in the REST API call to only fetch active pages
     - Determines which entities should be deleted (inactive/disabled)
     - Inserts entities into `entities_to_sync` queue with `operation_type: "upsert"`
     - Respects page size (50 entities per API call)
   - **Cleanup Phase**: Handles orphaned entities
     - Gets all currently indexed entities from `entities_indexed`
     - Compares with fetched entities to find orphans
     - Queues orphaned entities to `entities_to_sync` with `operation_type: "delete"`
4. Returns counts of synced and queued for deletion

### Attribute Handling

Attributes are handled differently from other entities during full sync:
- Attributes are processed directly into `entities_cache` (not queued)
- They provide labels for select/multiselect product attribute values
- The REST API returns option IDs, not labels, requiring this mapping
- Attributes are refreshed every hour via scheduled full-sync

---

## Hash-Based Change Detection

The system implements SHA-256 hash comparison to optimize sync operations:

1. **Compute Hash** - Generate deterministic hash of each Algolia object payload
2. **Compare** - Check against previously stored hash
3. **Sync Decision**:
   - Hash changed or new: Sync to Algolia
   - Hash unchanged: Skip (no API call needed)
4. **Persist** - Save new hashes only after successful Algolia sync
5. **Cleanup** - Remove hashes when entities are deleted from Algolia

This significantly reduces unnecessary Algolia API calls when entities haven't actually changed.

---

## Data Integrity and Edge Cases

### Phantom Delete Prevention

When a sync event arrives but the API returns nothing (entity is now invisible/disabled):
1. Check `entities_indexed` table for (entity_id, store_id)
2. If exists: Entity was previously indexed, add to delete queue
3. If not exists: Entity was never indexed, skip silently

This prevents:
- Endless delete attempts for never-indexed entities
- Orphaned records in Algolia

### Deduplication and Last-Operation-Wins

Multiple events for the same entity within a short time window are deduplicated:
- 5-second cutoff prevents race conditions
- MongoDB aggregation groups by (entity_id, store_id, entity_type)
- The **last operation_type** (by `created_at`) determines whether the entity is synced or deleted
- This resolves race conditions: e.g., `upsert → upsert → delete` correctly results in a delete
- All related queue rows are updated together

### Store View Expansion

When an event arrives for store_id=0 (default store):
- The system expands this to ALL configured store views
- Creates separate queue entries for each store
- Each store syncs independently with store-specific data

---

## Cleanup and Maintenance

### Automatic Cleanup

TTL indexes can be configured on queue collections to automatically remove old SUCCESS records after a retention period (e.g., 30 days).

### Stuck Processing Recovery

The orchestrator query automatically includes items stuck in PROCESSING status beyond a timeout threshold, providing self-healing capabilities.

### Manual Cleanup

A cleanup action can be scheduled to:
- Delete SUCCESS records older than configured retention (e.g., 7 days)
- Reset PROCESSING records stuck for too long
- Archive or handle FAILED records

---

## Scalability Characteristics

### Horizontal Scaling

- **Orchestrators**: Single instance per entity type, fast operations (query + publish)
- **Batch Processors**: Adobe I/O Runtime auto-scales based on event volume
- **Multiple instances process batches in parallel**

### Throughput

Depends on:
- Adobe I/O Runtime auto-scaling
- Adobe Commerce API rate limits
- Algolia API rate limits
- Batch size configuration

### Backpressure

- Adobe I/O Events queues events naturally
- Runtime Actions invoked at sustainable rate
- Cannot overwhelm downstream APIs

---

## Monitoring Considerations

Key metrics to track:
- Queue depth (pending items in unified queue, by operation_type)
- Processing rate (entities per orchestrator run)
- Error rate (percentage of FAILED items)
- Stuck processing count (items in PROCESSING too long)
- Runtime Action metrics (invocation count, failure rate, execution time)
- Tracking table health (compare with actual Algolia index size)
