# Reference: Documentation Service

This document details the service-oriented architecture for fetching, caching, and rendering project documentation on the `homepage` frontend. This system is designed to be highly performant, scalable, and fully automated.

## 1. Core Philosophy

The documentation platform is built on a decoupled, service-oriented model. The `spitikos/docs` repository is the single source of truth, but the frontend application is completely decoupled from it. The backend `api` service acts as a central orchestrator, responsible for the entire data lifecycle.

The key principles are:
- **Decoupled:** The frontend (`homepage`) has no direct knowledge of Git or the documentation's source. It only communicates with the `api` service.
- **Centralized Logic:** The `api` service owns the entire process of fetching, parsing, structuring, and caching the documentation.
- **Performant Caching:** A Redis instance is used as a high-speed, in-memory cache for the structured document data, minimizing latency for the frontend.
- **Automated Synchronization:** The entire pipeline is triggered automatically by a `git push` to the `spitikos/docs` repository.

## 2. The Data Workflow

The system is a chain of events that starts with a developer pushing to the docs repository and ends with the updated content being rendered on the website.

### Step 1: The Trigger (GitHub Webhook)

-   A GitHub Actions workflow in the `spitikos/docs` repository is configured to run on every push to the `main` branch.
-   This workflow finds all markdown files (`.md`) in the repository and compiles a JSON array of their file paths.
-   It then sends this list of paths in a `POST` request to a dedicated webhook endpoint on the `api` service: `/webhook/github/docs`.

### Step 2: Ingestion and Processing (`api` Service)

-   The `api` service's `WebhookHandler` receives the incoming request. It validates the payload using a shared secret to ensure the request is legitimately from GitHub.
-   The handler then triggers the `DocsService`'s `Sync` method.

### Step 3: Caching Strategy ("Nuke and Pave")

The `DocsService` implements a "Nuke and Pave" caching strategy to ensure the cache is always a perfect mirror of the repository.

1.  **Nuke:** The service first finds all existing document keys in Redis (by scanning for the `docs:*` pattern) and deletes them all.
2.  **Pave:** It then concurrently processes the list of file paths received from the webhook. For each path:
    -   It uses a GitHub client to fetch the raw file content.
    -   It fetches the last commit information for that file to get the modification date.
    -   It parses the content to extract the title and structures the data into a Protobuf `Doc` message.
    -   This `Doc` message is serialized to JSON and stored in Redis using a semantic key (e.g., `docs:document:overview:architecture-overview`).

This process is safe from downtime because the `homepage` frontend uses Next.js's Incremental Static Regeneration (ISR), which serves a stale-but-complete cache while revalidation happens in the background.

### Step 4: Frontend Rendering

-   When a user visits a documentation page, the Next.js server fetches the pre-processed document from the `api` service's `GetDoc` gRPC endpoint.
-   The `api` service retrieves the JSON data from Redis, deserializes it, and sends it to the frontend.
-   The frontend then renders the content. Pages are cached statically for high performance.

## 3. Required Configuration

For this system to function, the `api` service requires the following environment variables, which are dynamically injected by HashiCorp Vault at runtime:

-   `GITHUB_TOKEN`: A GitHub Personal Access Token with permission to read the `spitikos/docs` repository.
-   `GITHUB_WEBHOOK_SECRET`: The shared secret used to validate payloads from the GitHub webhook.
-   `REDIS_URL`: The internal cluster address of the Redis service (e.g., `redis-master.redis.svc.cluster.local:6379`).

## 4. Future Improvement: On-Demand Revalidation

While the current system relies on a time-based ISR cache in the frontend (e.g., 1 hour), the architecture is designed for a more advanced, instantaneous update mechanism.

**Recommendation:**
The `api` service should be updated to send a `POST` request to a new webhook endpoint on the `homepage` application (e.g., `/webhook/revalidate-docs`) after the Redis cache has been successfully repopulated.

This frontend endpoint would use Next.js's `revalidateTag` or `revalidatePath` function to purge the ISR cache on-demand. This would complete the event-driven architecture, ensuring that documentation updates are reflected on the website almost instantly after the `git push` is complete.
