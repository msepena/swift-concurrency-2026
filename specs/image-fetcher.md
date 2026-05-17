# Image Fetcher

## 1. Branch

Suggested branch: `feature/image-fetcher`

Run when ready:

```sh
git checkout -b feature/image-fetcher
```

## 2. Overview

Build a Swift image loader service that fetches image data from a remote server with built-in caching, batch downloads, and in-flight request deduplication. The user's brief: *"create a image loader service to fetch data from remote server, add caching, batch downloads and dedup"*. The service exposes a Swift Concurrency–only API (async/await), maintains a two-tier cache (in-memory hot + on-disk cold that survives process restarts), and exposes both a decoded-image accessor and a raw-bytes accessor that share the same underlying cache.

## 3. Acceptance criteria

- [ ] Public API is Swift Concurrency only — no closure callbacks, no Combine. Two methods: `image(for: URL) async throws -> PlatformImage` and `data(for: URL) async throws -> Data`.
- [ ] Both methods share the same underlying cache: a `data(for:)` call after an `image(for:)` for the same URL does not re-download.
- [ ] Two-tier cache: in-memory cache for hot results and on-disk cache for cold results. Disk cache survives process restarts.
- [ ] In-flight deduplication: concurrent requests for the same URL trigger exactly one network request; all callers receive the same result (success or failure).
- [ ] Batch download API accepts a collection of URLs and returns results keyed by URL. Repeats inside a batch are deduped.
- [ ] Cache hits perform no network I/O (verifiable via an injected `URLSession` or HTTP-call counter).
- [ ] Cache budgets are configurable at construction: memory cache (count and/or byte cap) and disk cache (byte cap with eviction).
- [ ] Cancellation semantics: cancelling a caller's `Task` cancels the network work *only when no other caller is awaiting the same URL*; otherwise the network work continues for the remaining waiter(s).
- [ ] Thread-safety: the service is safe to call from any actor or `Task`. Internal mutable state is isolated (likely an `actor`).
- [ ] Typed errors that let callers distinguish at least: network failure, HTTP status error, decode failure, cancellation, cache-only miss.

## 4. Edge cases

- Same URL requested concurrently by many callers — must dedupe to one in-flight task and share the result.
- A caller cancels mid-flight while other callers still await the same URL — continue serving the remaining waiters.
- The only awaiting caller cancels — cancel the network task and abort any partial disk write atomically.
- Disk cache exceeds its byte cap during a write — evict by the chosen policy without blocking concurrent reads of other entries.
- App backgrounded mid-download — define which URLSession configuration is used (default vs background) and what happens on app return.
- Server returns 304 Not Modified — refresh the access timestamp without re-downloading the body.
- Server returns a non-image MIME type or corrupt bytes — `image(for:)` throws a decode error; `data(for:)` still succeeds and returns the raw bytes.
- Partial or corrupt cache entry from a prior crash — detect on read (size or checksum mismatch) and treat as a miss, discarding the bad entry.
- Very large image (e.g. ≥50 MB) — disk cache accepts it; memory cache may refuse based on byte cap to avoid OOM.
- URL with volatile query parameters (e.g. signed auth tokens) — define whether the cache key is the raw URL or a canonicalized form (see Open Questions).
- Batch with a single failing URL among many — partial-failure semantics must be defined (see Open Questions).

## 5. Test cases

Unit:

- First fetch of a URL returns the decoded image; mock session sees exactly one request.
- Second fetch of the same URL hits the memory cache; mock session sees no additional requests.
- Cold start with a disk-cached entry returns the image without any network request.
- `data(for:)` after `image(for:)` for the same URL uses the shared cache (no second network call).
- Two concurrent `image(for:)` calls for the same URL produce exactly one network request and identical results.
- Cancelling one of N concurrent waiters does not cancel the result for the other N-1.
- Cancelling all concurrent waiters cancels the underlying network task.
- Memory cache evicts the least-recently-used entry when the count or byte cap is exceeded.
- Disk cache evicts the least-recently-used entry when the byte cap is exceeded; eviction does not block concurrent reads.
- Corrupt response bytes: `image(for:)` throws decode error; `data(for:)` succeeds.
- HTTP 304 response updates the cache access timestamp without rewriting the body.
- HTTP 4xx/5xx responses surface as typed errors and are not cached as success.
- Invalid URL or unsupported scheme surfaces a typed error.

Integration:

- End-to-end fetch against a local stub HTTP server.
- Disk persistence: write an entry, deallocate and reinstantiate the service, verify cache hit.
- Batch fetch of 100 URLs containing duplicates: assert request count equals the number of *unique* URLs.
- Batch fetch where some URLs fail: partial-failure mode behaves per the chosen contract (see Open Questions).

## 6. Open questions

- Which platforms must the service support? "Generic/other" was selected during clarification — the concrete list (iOS, macOS, tvOS, visionOS, Linux Swift, etc.) determines the `PlatformImage` typealias, the disk-cache path resolution, and whether `UIImage`/`NSImage`/`CGImage` is the canonical decoded type.
- Batch API surface and failure semantics: `images(for:) async throws -> [URL: PlatformImage]` (fail-fast) vs `images(for:) -> AsyncStream<(URL, Result<PlatformImage, Error>)>` (per-URL results)?
- Cache eviction policy: strict LRU, LRU + TTL, or size-aware (large entries evicted preferentially)? What are the default memory and disk byte caps?
- Should the service honor HTTP cache headers (`Cache-Control`, `ETag`, `Last-Modified`) for revalidation, or use a fixed library-managed TTL and ignore server hints?
- Cache key strategy: full URL as-is, or canonicalized (strip auth query params, normalize host casing, etc.)? If canonicalized, what's the canonicalization rule?
- `URLSession` configuration: shared, ephemeral, background, or fully injectable for testability? (Recommendation: injectable, defaulting to a private ephemeral session.)
- Concurrency limit: cap simultaneous network requests (e.g. 6) or leave unbounded? Affects performance and server fairness under batch load.
- Image decoding execution: synchronous on the awaiting context, or hopped off to a background executor (`@concurrent` or a dedicated `Task.detached`) to keep the calling actor responsive?
