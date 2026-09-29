# StashD

**One binary. No JVM. Your CI never sees a Docker Hub 429 again.**

StashD is a single-binary, self-hosted OCI/Docker registry **mirror** (pull-through cache). It sits between your infrastructure and upstream registries (Docker Hub, GHCR), caching manifests in SQLite and blob layers content-addressed on disk so the second pull of any image never touches the upstream, and your CI fleet stops colliding with anonymous rate limits.

## The problem

| Without stashd | With stashd |
|---|---|
| Docker Hub anonymous pulls are rate-limited (~100/6h per source IP) a CI fleet shares that budget and hits 429s | One upstream pull serves the whole fleet; the rest come from local disk |
| Every job re-downloads the same base layers across the WAN | Layers are cached content-addressed; dedup across images is free |
| Existing mirrors are heavyweight: Artifactory/Nexus (JVM, multi-GB), Harbor (12-container compose), `registry:2` proxy mode (zero observability) | One static Rust binary + one cache directory |
| `registry:2` proxy mode gives you no metrics, no admin API, no visibility into what the cache is doing | Prometheus metrics, admin API, cache HIT/MISS headers, structured logs |

stashd is compatible with `dockerd` `registry-mirrors`, containerd mirror config, and the normal Kubernetes image pull flow. It speaks the OCI Distribution Spec v1 read surface (`/v2/`, manifests, blobs, tag listing) including bearer-token auth against Hub/GHCR, tag revalidation via `Docker-Content-Digest`, and `Range` requests for resumable pulls.

## Quickstart (the target experience)

```bash
# 1. Run the mirror
docker run -d -p 5000:5000 -v stashd:/var/lib/stashd ghcr.io/linw00d93/stashd:latest

# 2. Point dockerd at it (or configure containerd / Kubernetes)
dockerd --registry-mirror=http://localhost:5000 &

# 3. Pull twice
docker pull redis:7-alpine    # MISS — streams through stashd, cached on disk
docker pull redis:7-alpine    # HIT — served from disk, zero Hub traffic
```


## How it works

- **Manifests** are small (KBs): stored in SQLite (`{cache_dir}/stashd.db`, WAL), keyed by `(upstream, repo, reference)` they survive restarts. Digest refs are immutable (cache forever); tag refs revalidate against upstream via HEAD + `Docker-Content-Digest` compare after a short TTL.
- **Stale-while-revalidate** (task 5.1): stale tag manifests within the `stale_while_revalidate_seconds` window (default 1h) are served instantly while a background task revalidates and if the upstream is unreachable, the stale copy is served rather than failing the pull. Your CI keeps working through Hub outages and rate-limit storms; `stashd_served_stale_total` on `/metrics` counts every stale serve honestly.
- **Tag listing** (`/v2/{repo}/tags/list`) is a live passthrough  deliberately uncached, since tag lists change independently of manifests and a stale list misleads tag-watching tooling; pagination (`n`/`last`, `link` header) forwards transparently.
- **Blobs** (the big layers) are content-addressed on disk at `blobs/sha256/{2-hex}/{64-hex}`. Cold misses **stream through**: upstream client while simultaneously writing to a `.tmp` file and hashing; the file is atomically renamed into the cache only after the assembled sha256 matches the requested digest. Corrupt or truncated fetches never poison the cache.
- **Single-flight** on both token fetches (per auth scope) and cold blob fetches (per digest): when 100 CI jobs pull the same new image at once, the upstream sees one request, not a thundering herd.
- **429s propagate honestly**: if the upstream rate limit is hit, clients see the 429 with `Retry-After`  never a hang or a fake 500.
- **Observability built in**: `GET /metrics` exposes Prometheus counters and histograms `stashd_served_from_cache_total` (the honest "Hub 429s avoided" number: only responses that used zero upstream traffic), upstream request counts/durations, 429 tallies, in-flight cold fetches, bytes served vs. bytes fetched.
- **Admin API**: `GET /api/v1/cache/stats` and `DELETE /api/v1/cache/invalidate?pattern=...` for cache inspection and targeted manifest invalidation; set `admin.token` to require a Bearer token.
- **Disk-bounded by design**: a background task evicts least-recently-served blobs whenever the cache exceeds `max_cache_size_gb`, so the disk footprint is predictable; eviction metrics are on `/metrics`.
- **Bounded in both dimensions**: blob eviction is disk-bounded, and the same background pass prunes SQLite manifest rows whose referenced content has been evicted neither store grows without bound.
- **Traceable requests**: every response carries an `X-Request-Id` (yours, if you sent one otherwise generated), logged with the request in structured JSON so client reports and server logs correlate in one lookup.
- **Per-request access logs**: one JSON line per completed request (status, latency, request id) on by default, so `docker logs` shows every pull flowing through the mirror in real time.
- **Graceful under restart**: `docker stop` drains in-flight pulls to completion before exit image layers never die mid-stream on a routine container restart.
- **Load-tested**: 100 concurrent pulls of 4 images collapse to just 8 upstream requests the upstream footprint is flat no matter how many workers stampede (dedup + single-flight); warm re-pulls cost Hub exactly zero traffic at ~190 pulls/s. Run the suite yourself: `cargo test --release --test load -- --ignored --nocapture`.


## Open-core charter

The mirror caching, revalidation, metrics, admin API is open source (MIT) and stays that way. Later paid tiers (auth/fleet management, multi-format package support) build **on top of** this wedge, never by restricting it. No hosted multi-tenant service: self-hosted on your infra sidesteps Docker Hub ToS entirely and is where the compliance-sensitive audience wants this anyway.

## License

MIT
