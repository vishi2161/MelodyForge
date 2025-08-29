# Project: MelodyForge – Spotify‑like Backend (Spring Boot)

> **Purpose**: Single repository, production‑grade backend for a web music player (Spotify‑style) using Java 21 + Spring Boot 3. Focus on MVP first; iterative hardening and feature growth later. This document is our source of truth (HLD + LLD + backlog). We’ll keep it updated as we build.

---

## 0) Scope At‑A‑Glance

**In‑scope (MVP):**

* AuthN/AuthZ with OAuth2.1 (passwordless/email-magic or username+password) + JWT access/refresh tokens.
* User profile + roles (USER, ARTIST, ADMIN).
* Music catalog: Artists, Albums, Tracks, Genres, Artwork.
* Playlists (public/private), likes (tracks & playlists), follows (artists), library (saved tracks/albums).
* Playback: HTTP byte‑range streaming of MP3 (pre‑encoded), basic play logging ("scrobble").
* Search: Postgres full‑text (tsvector) across artists/albums/tracks/playlists.
* Upload pipeline (ADMIN/ARTIST): upload audio + artwork to object storage, metadata extraction, catalog ingestion.
* Recently played, top tracks (simple aggregation).
* API documentation (OpenAPI 3), versioned APIs.
* Observability: structured logs, metrics, tracing.
* CI (GitHub Actions), Docker Compose dev stack.

**Phase‑2 (Post‑MVP):** HLS transcoding, recommendations, collaborative playlists, social graph feed, Elasticsearch/OpenSearch, email notifications, rate limiting gateway, payments/subscriptions, multi‑region.

**Out‑of‑scope (for now):** DRM, complex licensing, podcasts, ads, offline downloads, ML personalization at scale.

---

## 1) Architecture (HLD)

### 1.1 Style

* **Hexagonal/Ports & Adapters** inside a **modular monolith**. Microservice split later if needed.
* Layers per feature: `api` (controllers + DTO), `domain` (entities + services), `infra` (persistence, messaging, storage adapters).

### 1.2 Tech Stack

* **Language/Runtime**: Java 21, Gradle (Kotlin DSL).
* **Framework**: Spring Boot 3.3 (Web, Security, Validation, Data JPA, OAuth2 Resource Server), Springdoc OpenAPI.
* **Database**: PostgreSQL 15+, **Migrations**: Flyway.
* **Cache**: Redis (sessions, hot metadata, rate limits later).
* **Object Storage**: S3‑compatible (MinIO in dev; S3 in prod).
* **Messaging (optional MVP)**: Kafka (events; can stub with Spring events initially).
* **Media Tools**: FFmpeg (Phase‑2 transcoding), jaudiotagger for metadata.
* **Observability**: Micrometer + Prometheus + Grafana; OpenTelemetry traces (OTLP to Tempo/Jaeger).
* **Testing**: JUnit 5, Testcontainers, WireMock, ArchUnit.

### 1.3 Context & Components

* **API Gateway (later)** ➜ **MelodyForge App** ➜ external systems (SMTP provider, S3, Auth Provider if external).
* Inside App: Auth, Catalog, Playlist, Library, Search, Streaming, Upload/Media, Analytics, Admin.

### 1.4 Deployment Topology

* Dev: Docker Compose (Postgres, Redis, MinIO, Localstack optional, Prometheus, Grafana).
* Prod: Containerized on Kubernetes (Helm), separate Stateful services for DB/Redis; S3 managed.

---

## 2) Data Model (HLD ➜ LLD)

### 2.1 Core Entities

* **User(id, email, displayName, roles\[], createdAt, status)**
* **Artist(id, name, bio, coverUrl, followerCount)**
* **Album(id, title, artistId, releaseDate, artworkUrl)**
* **Track(id, title, albumId, artistId, durationSec, trackNumber, discNumber, explicit, bitrateKbps, audioKey, waveformKey, popularity)**
* **Genre(id, name)** (m2m TrackGenre)
* **Playlist(id, ownerUserId, title, description, isPublic, coverUrl, trackItems\[playlistId, trackId, addedByUserId, position])**
* **Library Saved**: SavedTrack(userId, trackId), SavedAlbum(userId, albumId)
* **Likes**: LikeTrack(userId, trackId), LikePlaylist(userId, playlistId)
* **Follow**: FollowArtist(userId, artistId)
* **Playback**: PlaybackSession(id, userId, startedAt, userAgent, ip, device), Scrobble(id, userId, trackId, playedAt, msPlayed)
* **UploadBatch(id, createdByUserId, status, createdAt)**, **MediaObject(id, batchId, type{audio,art}, objectKey, sha256, size, mime, state)**

### 2.2 Relational Schema (initial cut)

* Postgres with **UUID PKs**; **snake\_case** tables; **created\_at/updated\_at**; soft‑delete only where needed.
* Indices: lower(name) for ILIKE, tsvector columns for search, composite for joins (e.g., playlist\_id, position).

### 2.3 Object Storage Layout

```
s3://melodyforge/
  audio/{trackId}/source.mp3
  audio/{trackId}/hls/{bitrate}/{segment}.ts    # phase‑2
  artwork/{albumId}/cover.jpg
  waveform/{trackId}.json
```

---

## 3) API (HLD)

**Versioning**: `/api/v1/**` (semantic changes create `/v2`). **Auth**: Bearer JWT; refresh flow.

### 3.1 Auth

* `POST /auth/register` – email + password, or passwordless start.
* `POST /auth/login` – returns access/refresh.
* `POST /auth/refresh` – rotate tokens.
* `POST /auth/logout` – revoke refresh.

### 3.2 Catalog & Search

* `GET /artists?query=&page=`; `GET /artists/{id}`
* `GET /albums?artistId=&query=`; `GET /albums/{id}`
* `GET /tracks?albumId=&artistId=&query=`; `GET /tracks/{id}`
* `GET /search?q=...` – unified; returns typed slices.

### 3.3 Playlists & Library

* `GET/POST /playlists`; `GET/PATCH/DELETE /playlists/{id}`
* `POST /playlists/{id}/tracks` (bulk add); `DELETE /playlists/{id}/tracks/{trackId}`; `PATCH /playlists/{id}/reorder`
* `PUT/DELETE /me/tracks/{trackId}`; `PUT/DELETE /me/albums/{albumId}`; `GET /me/library`
* `PUT/DELETE /me/following/artists/{artistId}`; `GET /me/following`
* `PUT/DELETE /me/likes/tracks/{trackId}`; `PUT/DELETE /me/likes/playlists/{playlistId}`

### 3.4 Playback & Analytics

* `GET /streams/tracks/{trackId}` – **byte‑range** support; content‑disposition inline; checks entitlement.
* `POST /playback/scrobbles` – batch scrobbles with msPlayed.
* `GET /me/recently-played` – recent scrobbles.

### 3.5 Upload & Admin

* `POST /uploads/batches` – start; `POST /uploads/{batchId}/media` – **S3 pre‑signed URL**.
* `POST /uploads/{batchId}/ingest` – finalize & create catalog entities.
* `GET /admin/ingest/jobs` – status.

---

## 4) Streaming Design (LLD)

* MVP serves **MP3** (128/192/320 kbps) already present in S3. Spring MVC controller supports **Range requests** with `ResourceRegion`.
* Entitlement check: public tracks; later subscription checks.
* Phase‑2: **HLS** via transcoding job ➜ manifest `.m3u8` + segments; CloudFront/S3 static or app proxy; signed URLs.
* Optional **waveform** JSON precomputed for UI scrubbing.

---

## 5) Module Layout (Gradle)

```
melodyforge/
  buildSrc/              # common plugins + versions
  app/                   # boot app (thin)
  modules/
    common/              # shared (errors, id types, utils)
    security/
    users/
    catalog/
    playlists/
    library/
    streaming/
    uploads/
    search/
    analytics/
    admin/
```

Each module → `api` + `domain` + `infra` packages, with interfaces for ports and adapters implementing them.

---

## 6) Security (LLD)

* Spring Security with **JWT** (access \~15m, refresh \~14d). RS256 keys (rotatable).
* Password hashing: Argon2id. Optional email OTP for passwordless.
* Roles: `ROLE_USER`, `ROLE_ARTIST`, `ROLE_ADMIN`; method security via `@PreAuthorize`.
* CSRF: stateless API (disabled), CORS configured for our React hosts.
* Refresh token store: Redis (revocation + rotation).

---

## 7) Persistence (LLD)

* Spring Data JPA with **entity graphs** for read performance, projections for DTOs.
* Migrations: **Flyway**; SQL first (explicit), generated entities match.
* Testcontainers for integration tests; data builders + `@Sql` for fixtures.

---

## 8) Search (LLD)

* MVP: Postgres FTS: `tsvector` columns on name/title + trigram index for fuzzy.
* Phase‑2: OpenSearch for typo‑tolerant, synonyms, boosting popularity.

---

## 9) Media Ingestion (LLD)

Pipeline steps:

1. Client requests pre‑signed URLs for audio/artwork (MinIO dev).
2. Client uploads.
3. Server verifies `sha256`, size, mime.
4. Extract tags (ID3) via jaudiotagger; compute duration (ffprobe) if available.
5. Create/attach Artist/Album/Track, write metadata.
6. Mark **ready**; async thumbnails/waveforms.

State machine for `MediaObject.state`: `UPLOADING → UPLOADED → VALIDATED → INGESTED → READY | FAILED`.

---

## 10) Observability & Ops

* **Logging**: JSON logs (Logback encoders), correlation IDs; PII‑safe.
* **Metrics**: Micrometer HTTP/server/db counters + custom (streams served, scrobbles/sec).
* **Tracing**: OpenTelemetry instrumentation.
* **Health**: `/actuator/health`, liveness/readiness; info endpoint exposes git sha.
* **Feature flags**: Togglz/FF4J (optional).

---

## 11) Quality & Standards

* Code style: Google Java Style + spotless. Nullability annotations.
* Error handling: problem+json RFC 9457; global `@ControllerAdvice`.
* DTO mapping: MapStruct; Validation: Jakarta Validation.
* Architecture tests: ArchUnit to enforce layering and module boundaries.
* Security tests: Spring Security test utilities.

---

## 12) API Contracts (samples)

### 12.1 Track DTO

```json
{
  "id": "uuid",
  "title": "string",
  "artist": {"id": "uuid", "name": "string"},
  "album": {"id": "uuid", "title": "string", "artworkUrl": "https://..."},
  "durationSec": 241,
  "explicit": false,
  "popularity": 42
}
```

### 12.2 Stream Response

* `200 OK` with `Content-Type: audio/mpeg`, `Accept-Ranges: bytes`, `Content-Length`, `Content-Range` when partial.
* 403 when not entitled; 404 when track or blob missing.

---

## 13) Build, Run, Deploy

* **Local**: `docker compose up -d postgres redis minio prometheus grafana` ➜ `./gradlew :app:bootRun`.
* **Profiles**: `dev`, `test`, `prod`. Externalize secrets via env/Secrets Manager.
* **Images**: Jib/Buildpacks to build container; distroless base.
* **CI**: Build, run unit + integration (Testcontainers), static analysis (SpotBugs), SCA (OWASP Dependency‑Check), publish image, run Flyway in CD step.

---

## 14) Security & Privacy Considerations

* Minimal PII; encrypt at rest (DB column‑level if storing emails), TLS in transit.
* Token revocation on password change. Audit log for admin actions.
* Content ownership for uploads; DMCA process (admin endpoints) later.

---

## 15) Phased Roadmap & Milestones

### Phase A – MVP Core (2–3 weeks of focused work)

1. Repo scaffolding (Gradle, modules, Spotless, Spring Boot skeleton, Flyway baseline).
2. Auth (JWT) + Users module + test suites.
3. Catalog read APIs (Artists/Albums/Tracks) with seed data.
4. Streaming endpoint (byte‑ranges) backed by MinIO; simple entitlement.
5. Playlists + Library + Likes + Follows (CRUD flows).
6. Search via Postgres FTS.
7. Upload pipeline (admin only), metadata extraction, ingestion.
8. Observability + OpenAPI docs + CI pipelines.

**Exit criteria**: A user can sign up, search,
- A user can register/login and obtain JWT tokens.
- A user can search for artists, albums, and tracks.
- A user can play a track end-to-end through the streaming endpoint (with byte-range support).
- A user can like/save tracks and albums.
- A user can create, update, and share playlists (public/private).
- A user can view their recently played tracks.
- An admin can upload audio + artwork, trigger ingestion, and see the content appear in the catalog.
- Observability endpoints (metrics, health, traces) and OpenAPI docs are available.
- CI pipeline runs unit + integration tests successfully and produces deployable artifacts.
