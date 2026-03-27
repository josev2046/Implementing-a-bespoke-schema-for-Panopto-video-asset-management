# Implementing a bespoke schema for Panopto video asset management

[![DOI](https://zenodo.org/badge/1192616761.svg)](https://doi.org/10.5281/zenodo.19250734)

**Document version:** 1.3  
**Date:** March 2026

---

## Contents

1. [Overview](#1-overview)
2. [Panopto's native metadata capabilities](#2-panoptos-native-metadata-capabilities)
3. [Implementation approaches](#3-implementation-approaches)
4. [Recommended IMDb-style schema](#4-recommended-imdb-style-schema)
5. [Panopto REST API — key integration points](#5-panopto-rest-api--key-integration-points)
6. [Approach selection guide](#6-approach-selection-guide)
7. [Risks and considerations](#7-risks-and-considerations)
8. [Summary](#8-summary)

---

## 1. Overview

The Panopto video platform is optimised for lecture capture and internal communications. Its native metadata model is intentionally minimal, offering only fixed fields such as Title, Description, Tags, Date, and Creator. It does not support bespoke schemas, custom fields, or structured relational metadata out of the box.

This document sets out the options for implementing an IMDb-style metadata schema — with defined fields such as Director, Cast, Genre, and Release Year — against a Panopto video library. It covers what Panopto natively provides, three implementation approaches with their respective trade-offs, a recommended architecture, and a practical field schema.

---

## 2. Panopto's native metadata capabilities

Understanding what Panopto does and does not support is the essential starting point.

| Native Field / Feature | Notes and Limitations |
|---|---|
| Title | Free text. No validation or controlled vocabulary. |
| Description | Free text block. Indexed by Panopto search. Can hold structured text as a workaround. |
| Tags | Flat keyword list. No hierarchy, no field-level categorisation. Hard limit of 10 tags per video. |
| Date | Auto-set on upload. Editable, but no custom date types (e.g. Release Year vs Upload Date). |
| Creator / Owner | Single user field. No multi-value cast or crew support. |
| Folder | Hierarchical container. Can proxy for genre or category grouping, but is not a metadata field. |
| ASR (Speech Recognition) | Auto-generates a searchable transcript. Valuable for discovery, not for structured metadata. |
| OCR (On-Screen Text) | Indexes visible text from slides and screen recordings. |
| Custom Fields | Not supported natively in any Panopto tier. |
| Faceted Filtering | Not supported. Tags can be filtered via the API but not through the native UI. |

The absence of custom fields and faceted filtering is the core constraint. Panopto's search engine is powerful for free-text and transcript discovery, but it cannot substitute for a proper relational schema when structured queries are needed.

---

## 3. Implementation approaches

Three approaches are available, with significantly different levels of capability, cost, and complexity.

### 3.1 Approach A — Structured description convention

The Description field is indexed by Panopto's search engine. By adopting a consistent key-value convention in the description, you can create a lightweight pseudo-schema that is searchable.

**How it works**

Each video description is formatted with explicit field labels:

```
Director: Christopher Nolan
Cast: Cillian Murphy, Emily Blunt, Matt Damon
Genre: Historical Drama
Release Year: 2023
Rating: PG-13
Runtime: 180 min
```

**Capabilities**

| Capability | Approach A |
|---|---|
| Free-text search by field value | ✓ Works (e.g. search "Director: Nolan") |
| Faceted / dropdown filtering | ✗ Not possible |
| Sorting by field (e.g. by year) | ✗ Not possible |
| Field validation | ✗ None — degrades over time without governance |
| Multi-value fields (e.g. multiple cast) | ✓ Possible but relies on consistent formatting |
| API queryability | ✗ Panopto API returns description as a text blob only |
| Implementation effort | ✓ Very low |

**Verdict:** Suitable only as a stopgap or for very small collections (under roughly 50 items) where informal search is sufficient. Not a scalable solution.

---

### 3.2 Approach B — Panopto tags as a structured convention

Panopto tags are searchable and filterable through the REST API. By imposing a namespaced tagging convention, tags can approximate a controlled vocabulary with minimal infrastructure.

**Convention format**

```
director:nolan-christopher
genre:thriller
year:2023
cast:murphy-cillian
```

**Critical limitation:** Panopto enforces a strict, hardcoded limit of 10 tags per video. An IMDb-style schema with multi-value fields (e.g. multiple cast members) will almost always exceed this limit immediately.

**Capabilities**

| Capability | Approach B |
|---|---|
| Search by field value | ✓ Works via API and native search |
| API filtering by tag | ✓ Supported (via searchQuery parameter) |
| Faceted UI filtering | ✓ Possible if you build a thin wrapper UI |
| Sorting | ✗ Limited |
| Field validation | ✗ Still relies on human discipline |
| Multi-value fields | ✗ Severely constrained by the 10-tag maximum |
| Implementation effort | ~ Low to medium (requires a small API layer or wrapper UI) |

**Verdict:** Due to the 10-tag limit, this approach is not viable for a robust, multi-field schema like IMDb. It should only be used if you need to track 2–3 specific data points (e.g. just genre and year).

---

### 3.3 Approach C — External database + Panopto REST API (Recommended)

This is the only approach that delivers a true bespoke metadata schema with full relational query capability. Panopto is used purely as a video host and streaming layer. All metadata lives in an external system.

**Architecture overview**

```
┌─────────────────────────────────────────────────────────┐
│  METADATA STORE                                         │
│  Database / CMS / MAM — owns the bespoke schema         │
│  e.g. Director, Cast, Genre, Release Year               │
└────────────────────┬────────────────────────────────────┘
                     │  Panopto Session ID (foreign key)
                     ▼
┌─────────────────────────────────────────────────────────┐
│  CUSTOM PORTAL / CMS INTERFACE                          │
│  Faceted filters, search, browse, editorial UI          │
└────────────────────┬────────────────────────────────────┘
                     │  Embed API / iframe
                     ▼
┌─────────────────────────────────────────────────────────┐
│  PANOPTO                                                │
│  Video storage, transcoding, streaming, player          │
└─────────────────────────────────────────────────────────┘
```

**Capabilities**

| Capability | Approach C |
|---|---|
| Custom schema fields | ✓ Full support — any fields, types, constraints |
| Faceted filtering | ✓ Full support |
| Sorting and pagination | ✓ Full support |
| Field validation and controlled vocabularies | ✓ Full support |
| Multi-value fields | ✓ Full support |
| Reporting and analytics on metadata | ✓ Full support |
| Panopto native UI still usable | ✓ Yes, but users browse your portal instead |
| Implementation effort | ~ Medium to high (depends on existing infrastructure) |

**Verdict:** The correct long-term solution for any collection that needs structured discovery, reporting, or integration with other systems.

---

## 4. Recommended IMDb-style schema

The following schema is proposed for a media catalogue implementation. Fields are categorised as core (required), extended (recommended), and operational (system-managed).

### 4.1 Core fields

| Field | Type | Notes |
|---|---|---|
| title | String — required | Primary display title |
| original_title | String | Original language title if different |
| release_year | Integer | Year of original release, not upload date |
| runtime_minutes | Integer | Duration in minutes |
| genre | Multi-value, controlled | e.g. Drama, Thriller, Documentary |
| director | Multi-value, person | Links to a Person entity |
| cast | Multi-value, person + role | Actor name and character name |
| synopsis | Long text | Editorial description |
| content_rating | Controlled vocabulary | e.g. U, PG, 12A, 15, 18 (BBFC) or G/PG/R (MPAA) |
| language | Controlled vocabulary | Primary spoken language — ISO 639-1 |
| country_of_origin | Controlled vocabulary | ISO 3166-1 alpha-2 country code(s) |

### 4.2 Extended fields

| Field | Type | Notes |
|---|---|---|
| producer | Multi-value, person | |
| writer | Multi-value, person | Screenplay or source author |
| composer | String | Score or soundtrack composer |
| studio | String | Production company |
| distributor | String | Distribution company |
| keywords | Multi-value, free tag | Supplementary discovery tags |
| awards | Structured list | Award name, category, year, won or nominated |
| related_titles | Reference (Session ID) | Sequels, prequels, related works |
| trailer_url | URL | Link to trailer — external or a separate Panopto session |

### 4.3 Operational fields (system-managed)

| Field | Type | Notes |
|---|---|---|
| panopto_session_id | UUID — required | Foreign key to Panopto. Never editable by end users. |
| panopto_folder_id | UUID | For reference and sync |
| upload_date | DateTime | Date ingested into Panopto |
| metadata_created_by | User | Who created the metadata record |
| metadata_last_updated | DateTime | Auto-updated on edit |
| publish_status | Enum | Draft / Published / Archived |

---

## 5. Panopto REST API — key integration points

The Panopto REST API is the bridge between your metadata system and Panopto. The following endpoints are most relevant to this architecture.

| Endpoint / API | Use Case |
|---|---|
| `GET /api/v1/sessions/{id}` | Retrieve session details including title, description, duration, tags, and the `DownloadUrl` property for programmatic file retrieval. If `DownloadUrl` returns `#`, downloads have been disabled at the session, folder, or site level and must be enabled in session settings before programmatic download is possible. |
| `PUT /api/v1/sessions/{id}` | Update title, description, and tags from your metadata system |
| `GET /api/v1/sessions?folderId={id}&searchQuery=tag:{value}` | List sessions in a folder, optionally filtered by a tag using the search engine |
| `GET /api/v1/sessions/{id}/thumbnails` | Retrieve thumbnail for display in your custom portal |
| `POST /api/v1/sessions/{id}/tags` | Add tags to a session — note the 10-tag maximum per session (see section 3.2) |
| `GET /api/v1/folders` | Navigate the folder hierarchy |
| Upload API | Programmatic ingest from an external pipeline — see section 5.1 below |

### 5.1 Programmatic upload (ingest pipeline)

Uploading video files programmatically is a multi-step process and is more involved than a single API call. The sequence is:

1. **Initiate the session** — `POST /api/v1/sessions` to create a session record and obtain a session ID and an S3-compatible upload target.
2. **Transfer the media file** — upload the video file to the S3 endpoint returned in step 1. This uses standard S3 multipart upload semantics, not the Panopto REST API directly.
3. **Complete the upload** — send a final REST call to signal that the file transfer is finished and trigger Panopto's transcoding pipeline.

Failure to send the completion call leaves the session in a pending state. Build explicit error handling and retry logic around step 3. For large files, implement multipart chunking at the S3 layer and monitor transfer progress independently of the Panopto session status.

### 5.2 Authentication

Panopto's REST API uses OAuth 2.0. The correct client type depends on what the integration needs to access.

**For server-side integrations that access non-public folders and sessions** (the typical case for a metadata pipeline or ingest workflow), use a **User-Based Server Application** client. This client type obtains a token using the `password` grant, with a dedicated service account's credentials:

```
POST /Panopto/oauth2/connect/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=password&username=SERVICE_ACCOUNT&password=PASSWORD&scope=api
```

The resulting bearer token inherits the service account's folder-level permissions. A plain **Server Application** client using `client_credentials` will only return publicly accessible content and will return `403 Forbidden` on protected folders and sessions — this is a common source of integration failures and is not a plan-level restriction.

API access requires a Panopto admin to register the application and assign appropriate folder-level permissions to the service account. Rate limits apply — batch operations should implement exponential backoff.

---

## 6. Approach selection guide

| Scenario | Recommended Approach | Rationale |
|---|---|---|
| Small collection (under 100 items), ad hoc search only | Approach A — Description convention | Low effort, good enough for informal use |
| Very simple metadata needs (1–2 fields only) | Approach B — Tag convention with thin API wrapper | Structured enough to filter programmatically, provided you stay well under the 10-tag limit per video |
| Any collection requiring an IMDb-style schema, reporting, or integration | Approach C — External DB and API | The only approach with true schema capability and no arbitrary limits on data fields |
| Existing CMS or MAM already in use | Approach C via existing platform | Extend what you have; Panopto becomes the player |
| Public-facing catalogue with SEO requirements | Approach C — custom portal | Panopto's native pages are not SEO-friendly |

---

## 7. Risks and considerations

### 7.1 Data governance

A bespoke schema requires editorial governance to remain useful. Controlled vocabularies — Genre, Language, Content Rating — should be centrally managed and not left to free-text entry.

Person entities (Director, Cast) should be deduplicated. Consider maintaining a shared authority file, or referencing an external identifier such as an IMDb `nm` identifier, to avoid divergence between entries like "Christopher Nolan" and "C. Nolan".

### 7.2 Panopto as source of truth

Decide clearly whether Panopto or your external system is the system of record for titles and descriptions. Bidirectional sync is complex and prone to conflict.

The recommended approach is to let your external system own the metadata. Panopto holds only what is needed for its own search and ASR features. Changes flow one-way from your system to Panopto via the API.

### 7.3 Orphaned metadata records

When a Panopto session is deleted, your external metadata store will retain the corresponding record with a `panopto_session_id` that no longer resolves. Over time this produces dead foreign keys that degrade catalogue integrity.

Mitigate this by one of two means: subscribe to Panopto deletion events if your plan exposes webhooks for session lifecycle changes, or run a periodic reconciliation job that calls `GET /api/v1/sessions/{id}` for each active record and marks any that return `404` as `publish_status: Archived` (or flags them for manual review). Do not silently delete metadata records on a 404 — the session may have been temporarily unavailable or the ID may have been incorrectly stored.

### 7.4 Panopto platform changes

Panopto is an actively developed SaaS platform. API endpoints and authentication mechanisms may change. Pin integration code to a specific API version where the platform allows it, and monitor Panopto release notes for any expansion of custom metadata features, which could simplify this architecture over time.

### 7.5 User experience and access control

Users accustomed to the Panopto interface will need onboarding to a new portal. Embedding the Panopto player seamlessly within your custom UI is essential to avoid a disjointed experience.

Ensure the Panopto embed respects your organisation's access control model. Panopto's folder-level permissions should align with the visibility rules in your portal so that users cannot access content through the player that they cannot access through the catalogue. In practice, this is enforced by using Panopto's session-based authentication tokens or signed embed URLs when initialising the player — your portal backend requests a scoped token on behalf of the authenticated user and passes it to the embedded player, ensuring Panopto's own permission layer remains the authoritative gatekeeper.

---

## 8. Summary

- Panopto does not natively support custom metadata schemas or bespoke fields at this time.
- Multiple approaches exist, ranging from low-effort conventions to a full external database architecture.
- For any use case requiring faceted filtering, sorting, or structured reporting, Approach C (external database + Panopto REST API) is the most viable long-term solution.
- The Panopto Session ID can act as the foreign key linking your metadata schema to video assets.
- Schema design should include governance rules for controlled vocabularies and person entity deduplication.
- Server-side integrations must use a **User-Based Server Application** OAuth client with `grant_type=password`; a plain Server Application client will return 403 on protected content.
- Programmatic video ingest requires a three-step sequence: initiate session, transfer via S3, complete upload.
- A reconciliation strategy for orphaned metadata records (sessions deleted from Panopto) should be included in any production implementation.
- Panopto is best treated here as a streaming and player layer — not as a metadata store.

---

*Jose Velazquez, MA — Document version 1.3 — March 2026*

