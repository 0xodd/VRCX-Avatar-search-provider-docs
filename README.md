# VRCX Avatar Database Provider

A guide to VRCX's **remote avatar database provider** (a.k.a. "avatar search provider") — the small HTTP API that lets VRCX search third‑party avatar indexes, look up an avatar by its author, and resolve an avatar from an image file ID.

This document describes the request/response contract implemented in
[`src/coordinators/avatarCoordinator.js`](https://github.com/vrcx-team/VRCX/blob/master/src/coordinators/avatarCoordinator.js),
so you can **configure an existing provider** or **build your own**.

> ⚠️ Unofficial documentation. It is reverse‑engineered from the VRCX client source and is not an official VRCX or VRChat product. Nothing here comes from VRChat's own API.

---

## Table of contents

- [What is an avatar database provider?](#what-is-an-avatar-database-provider)
- [How VRCX talks to a provider](#how-vrcx-talks-to-a-provider)
- [The request contract](#the-request-contract)
  - [1. Search query](#1-search-query)
  - [2. Author lookup](#2-author-lookup)
  - [3. File ID lookup (optional)](#3-file-id-lookup-optional)
- [The response contract](#the-response-contract)
  - [Required fields](#required-fields)
  - [Avatar object schema](#avatar-object-schema)
- [Using a provider in VRCX](#using-a-provider-in-vrcx)
- [Building your own provider](#building-your-own-provider)
  - [Node.js / Express example](#nodejs--express-example)
  - [Rust / Axum example](#rust--axum-example)
  - [C# / ASP.NET Core example](#c--aspnet-core-example)
- [Behavior & edge cases](#behavior--edge-cases)
- [Quick checklist](#quick-checklist)
- [FAQ](#faq)

---

## What is an avatar database provider?

VRChat's public API does **not** let you search avatars by name or list a user's public
avatars. To work around this, VRCX can query an **external** HTTP service — the *avatar
database provider* — that maintains its own crowd‑sourced index of avatars.

A provider is just a **single URL** (an endpoint). VRCX appends query parameters to it and
expects JSON back. The community runs several of these; you can also self‑host one.

VRCX uses a provider for three things:

| Feature in VRCX | What it does | Query mode used |
| --- | --- | --- |
| **Avatar search** | Free‑text search for avatars by name | `search` |
| **Author lookup** | Find all indexed avatars belonging to a user | `authorId` |
| **"Find avatar" from a player** | Resolve which avatar an image belongs to | `fileId`, then `authorId` |

---

## How VRCX talks to a provider

```
                       ┌───────────────────────────────────────────┐
   VRCX client         │  avatarCoordinator.js                     │
   (Vue frontend)      │                                           │
                       │  lookupAvatars('search',  q)   ──┐        │
   User searches ─────▶│  lookupAvatars('authorId', id)  ─┤        │
   / clicks avatar     │  lookupAvatarByImageFileId(...)  ─┘        │
                       └───────────────┬───────────────────────────┘
                                       │ HTTP GET  (webApiService.execute)
                                       │ headers: Referer, VRCX-ID
                                       ▼
                       ┌───────────────────────────────────────────┐
                       │  Your avatar database provider            │
                       │  GET ?search=…  |  ?authorId=…  |  ?fileId=…│
                       │  ──▶ returns JSON array / object          │
                       └───────────────────────────────────────────┘
```

Requests are sent from VRCX's native web request layer (`webApiService.execute`), **not** from a
browser `fetch`, so ordinary browser CORS rules do not apply. Every request carries two headers:

| Header | Value | Purpose |
| --- | --- | --- |
| `Referer` | `https://vrcx.app` | Identifies traffic as coming from VRCX |
| `VRCX-ID` | *(per‑install unique ID)* | Anonymous installation identifier; useful for rate‑limiting / abuse control |

The method is always **`GET`**. There is no request body and no authentication token.

---

## The request contract

A provider is one base URL. VRCX distinguishes the three operations purely by **which query
parameter** it appends.

### 1. Search query

Triggered by the avatar search box.

```
GET {provider}?search={urlEncodedQuery}&n=5000
```

- `search` — the user's search text (URL‑encoded).
- `n` — a result cap. VRCX always sends `5000`. Treat it as "return up to N results".

**Expected response:** a JSON **array** of [avatar objects](#avatar-object-schema).

### 2. Author lookup

Triggered when VRCX wants every indexed avatar for a given VRChat user (e.g. browsing an
author, or as a fallback while resolving an avatar). VRCX calls this against **every** URL in
the configured provider list and merges the results (de‑duplicated by `id`).

```
GET {provider}?authorId={urlEncodedUserId}
```

- `authorId` — a VRChat user ID, e.g. `usr_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`.

**Expected response:** a JSON **array** of [avatar objects](#avatar-object-schema).

### 3. File ID lookup (optional)

Triggered when a user clicks "find avatar" on another player — VRCX has the avatar's **image
file ID** but not the avatar ID. VRCX first asks each provider to resolve the file ID directly;
if none support it, it falls back to author lookup (mode 2) and matches locally on `imageUrl`.

```
GET {provider}?fileId={urlEncodedFileId}
```

- `fileId` — a VRChat file ID, e.g. `file_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`.

**Expected response:** a **single** JSON [avatar object](#avatar-object-schema).

> This mode is **optional**. If your provider doesn't support it, return a non‑200 status (or
> anything that isn't a JSON object). VRCX swallows the error silently and falls back to author
> lookup. Errors from `search` and `authorId`, by contrast, surface a toast to the user.

---

## The response contract

For **all** modes, VRCX requires:

- HTTP status **`200`**.
- A body that is **valid JSON** and parses to an **object** (a JSON array counts — `typeof json === 'object'` in JS).
  - `search` and `authorId` → a JSON **array**.
  - `fileId` → a single JSON **object**.

Anything else (non‑200, invalid JSON, non‑object) is treated as an error.

VRCX merges each returned object **on top of a set of defaults**, so any field you omit falls
back to a sensible empty value. See [Required fields](#required-fields) for what you actually
need to send.

### Required fields

The API contract is deliberately forgiving. In the source, every response object is spread over a
full set of defaults:

```js
const ref = {
    authorId: '', authorName: '', name: '', description: '',
    id: '', imageUrl: '', thumbnailImageUrl: '',
    created_at: '0001-01-01T00:00:00.0000000Z',
    updated_at: '0001-01-01T00:00:00.0000000Z',
    releaseStatus: 'public',
    ...avatar   // <-- your data overrides the defaults
};
```

Because of this, **no missing field will cause an error** — VRCX just uses the default. The only
*structural* requirement is a valid JSON response with HTTP `200` (an array for `search`/`authorId`,
a single object for `fileId`).

But "won't error" isn't "will work." In practice:

| Field | Required? | Why |
| --- | --- | --- |
| **`id`** | ✅ **Yes — always** | It is the map key (`avatars.set(ref.id, ref)`) and the de‑duplication key. Without it, **every** result collapses onto the empty‑string key `''`, so VRCX effectively keeps only one avatar. This is the one field you must always send. |
| **`name`** | ⚠️ For search | It's what the user sees and the whole point of a search hit. An unnamed result is useless. |
| **`authorId`** | ⚠️ For author lookup | Needed for the `?authorId=` mode to return/link correctly and to merge results across providers. |
| **`imageUrl`** | ⚠️ For "find avatar" | Used in the `fileId` fallback match: `extractFileId(avatar.imageUrl) === fileId`. Without it, resolving an avatar from a player's image can't match. |
| `authorName`, `description`, `thumbnailImageUrl`, `releaseStatus`, `created_at`, `updated_at` | ❌ Optional | Defaults are applied automatically. |

**Bottom line:** the only field the code truly *needs* is **`id`**. For a genuinely functional
provider, send **`id` + `name` + `authorId` + `imageUrl`**; everything else is optional polish.

### Avatar object schema

```jsonc
{
  "id":               "avtr_00000000-0000-0000-0000-000000000000", // REQUIRED — VRChat avatar ID; used as the unique key
  "name":             "My Cool Avatar",
  "description":      "A short description",
  "authorId":         "usr_00000000-0000-0000-0000-000000000000",
  "authorName":       "AuthorDisplayName",
  "imageUrl":         "https://api.vrchat.cloud/api/1/file/file_.../1/file",
  "thumbnailImageUrl":"https://api.vrchat.cloud/api/1/image/file_.../1/256",
  "releaseStatus":    "public",                                    // defaults to "public"
  "created_at":       "2023-01-01T00:00:00.0000000Z",              // ISO 8601
  "updated_at":       "2023-06-01T00:00:00.0000000Z"               // ISO 8601
}
```

Field reference (defaults applied by VRCX when a field is missing):

| Field | Type | Default if omitted | Notes |
| --- | --- | --- | --- |
| `id` | string | `""` | **Provide this.** VRChat avatar ID (`avtr_…`). Used as the map key and for de‑duplication. |
| `name` | string | `""` | Avatar display name (what search matches on). |
| `description` | string | `""` | Free text. |
| `authorId` | string | `""` | VRChat user ID of the author (`usr_…`). Enables author lookup. |
| `authorName` | string | `""` | Author's display name. |
| `imageUrl` | string | `""` | Full‑size image URL. Used to match `fileId` during fallback resolution. |
| `thumbnailImageUrl` | string | `""` | Thumbnail image URL. |
| `releaseStatus` | string | `"public"` | e.g. `public`, `private`. |
| `created_at` | string | `"0001-01-01T00:00:00.0000000Z"` | ISO 8601 timestamp. |
| `updated_at` | string | `"0001-01-01T00:00:00.0000000Z"` | ISO 8601 timestamp. |

Example **array** response (for `search` / `authorId`):

```json
[
  {
    "id": "avtr_11111111-1111-1111-1111-111111111111",
    "name": "Neon Fox",
    "authorId": "usr_22222222-2222-2222-2222-222222222222",
    "authorName": "FoxMaker",
    "imageUrl": "https://api.vrchat.cloud/api/1/file/file_aaaa/1/file",
    "thumbnailImageUrl": "https://api.vrchat.cloud/api/1/image/file_aaaa/1/256",
    "releaseStatus": "public",
    "created_at": "2023-02-14T12:00:00.0000000Z",
    "updated_at": "2023-08-01T09:30:00.0000000Z"
  }
]
```

Example **single object** response (for `fileId`):

```json
{
  "id": "avtr_11111111-1111-1111-1111-111111111111",
  "name": "Neon Fox",
  "authorId": "usr_22222222-2222-2222-2222-222222222222",
  "imageUrl": "https://api.vrchat.cloud/api/1/file/file_aaaa/1/file"
}
```

---

## Using a provider in VRCX

1. Open **VRCX → Settings → Advanced**.
2. Enable the **Avatar Database Provider** (remote avatar database) option. Internally this flips
   the `avatarRemoteDatabase` setting that gates remote look‑ups.
3. Add one or more **provider URLs**.
   - The **primary** provider is used for the free‑text `search` box.
   - **All** configured providers are queried for `authorId` and `fileId` look‑ups, and their
     results are merged (de‑duplicated by `id`).
4. Search avatars from the Avatar tab, or click **"find avatar"** on another player to resolve
   what they're wearing.

If a `search` or `authorId` request fails, VRCX shows an error toast naming the provider and the
query, which is handy while testing your own endpoint.

---

## Building your own provider

You need a single HTTP GET endpoint that branches on the `search`, `authorId`, and (optionally)
`fileId` query parameters and returns the JSON described above. Any language/stack works — below
are three minimal, illustrative reference implementations backed by an in‑memory list. Each one
always sends the truly required **`id`** plus the functional **`name` / `authorId` / `imageUrl`**
(see [Required fields](#required-fields)).

### Node.js / Express example

```js
// server.js — illustrative reference provider
import express from 'express';

const app = express();

// Your index. In production this is a database of crowd‑sourced avatars.
const AVATARS = [
  {
    id: 'avtr_11111111-1111-1111-1111-111111111111', // REQUIRED
    name: 'Neon Fox',
    description: 'A glowing fox avatar',
    authorId: 'usr_22222222-2222-2222-2222-222222222222',
    authorName: 'FoxMaker',
    imageUrl: 'https://api.vrchat.cloud/api/1/file/file_aaaa/1/file',
    thumbnailImageUrl: 'https://api.vrchat.cloud/api/1/image/file_aaaa/1/256',
    releaseStatus: 'public',
    created_at: '2023-02-14T12:00:00.0000000Z',
    updated_at: '2023-08-01T09:30:00.0000000Z'
  }
];

// Pull the VRChat file ID out of an image URL, e.g. .../file/file_aaaa/1/file -> file_aaaa
const fileIdOf = (url = '') => (url.match(/file_[0-9A-Za-z-]+/) || [])[0];

app.get('/avatars', (req, res) => {
  const { search, authorId, fileId, n } = req.query;

  // Mode 3: file ID lookup -> single object (or 404 to opt out)
  if (fileId) {
    const hit = AVATARS.find((a) => fileIdOf(a.imageUrl) === fileId);
    return hit ? res.json(hit) : res.sendStatus(404);
  }

  // Mode 2: author lookup -> array
  if (authorId) {
    return res.json(AVATARS.filter((a) => a.authorId === authorId));
  }

  // Mode 1: search -> array (respect the n cap)
  if (search) {
    const q = String(search).toLowerCase();
    const limit = Number(n) || 5000;
    const results = AVATARS
      .filter((a) => a.name.toLowerCase().includes(q))
      .slice(0, limit);
    return res.json(results);
  }

  return res.status(400).json({ error: 'missing query parameter' });
});

app.listen(3000, () => console.log('provider on http://localhost:3000/avatars'));
```

Configure VRCX to point at `http://your-host:3000/avatars`.

### Rust / Axum example

```rust
// main.rs — illustrative reference provider
// Cargo.toml deps: axum = "0.7", tokio = { version = "1", features = ["full"] },
//                  serde = { version = "1", features = ["derive"] }, serde_json = "1"
use axum::{extract::Query, response::IntoResponse, routing::get, Json, Router};
use serde::Serialize;
use std::collections::HashMap;

#[derive(Clone, Serialize)]
struct Avatar {
    id: String, // REQUIRED
    name: String,
    description: String,
    #[serde(rename = "authorId")]
    author_id: String,
    #[serde(rename = "authorName")]
    author_name: String,
    #[serde(rename = "imageUrl")]
    image_url: String,
    #[serde(rename = "thumbnailImageUrl")]
    thumbnail_image_url: String,
    #[serde(rename = "releaseStatus")]
    release_status: String,
    created_at: String,
    updated_at: String,
}

fn index() -> Vec<Avatar> {
    vec![Avatar {
        id: "avtr_11111111-1111-1111-1111-111111111111".into(),
        name: "Neon Fox".into(),
        description: "A glowing fox avatar".into(),
        author_id: "usr_22222222-2222-2222-2222-222222222222".into(),
        author_name: "FoxMaker".into(),
        image_url: "https://api.vrchat.cloud/api/1/file/file_aaaa/1/file".into(),
        thumbnail_image_url: "https://api.vrchat.cloud/api/1/image/file_aaaa/1/256".into(),
        release_status: "public".into(),
        created_at: "2023-02-14T12:00:00.0000000Z".into(),
        updated_at: "2023-08-01T09:30:00.0000000Z".into(),
    }]
}

// Pull the VRChat file ID out of an image URL: .../file/file_aaaa/1/file -> file_aaaa
fn file_id_of(url: &str) -> Option<&str> {
    let start = url.find("file_")?;
    let rest = &url[start..];
    let end = rest.find('/').unwrap_or(rest.len());
    Some(&rest[..end])
}

async fn avatars(Query(q): Query<HashMap<String, String>>) -> impl IntoResponse {
    let data = index();

    // Mode 3: file ID lookup -> single object (or 404 to opt out)
    if let Some(file_id) = q.get("fileId") {
        return match data.iter().find(|a| file_id_of(&a.image_url) == Some(file_id.as_str())) {
            Some(a) => Json(a.clone()).into_response(),
            None => axum::http::StatusCode::NOT_FOUND.into_response(),
        };
    }

    // Mode 2: author lookup -> array
    if let Some(author_id) = q.get("authorId") {
        let hits: Vec<_> = data.into_iter().filter(|a| &a.author_id == author_id).collect();
        return Json(hits).into_response();
    }

    // Mode 1: search -> array (respect the n cap)
    if let Some(search) = q.get("search") {
        let needle = search.to_lowercase();
        let limit: usize = q.get("n").and_then(|v| v.parse().ok()).unwrap_or(5000);
        let hits: Vec<_> = data
            .into_iter()
            .filter(|a| a.name.to_lowercase().contains(&needle))
            .take(limit)
            .collect();
        return Json(hits).into_response();
    }

    axum::http::StatusCode::BAD_REQUEST.into_response()
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/avatars", get(avatars));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("provider on http://localhost:3000/avatars");
    axum::serve(listener, app).await.unwrap();
}
```

### C# / ASP.NET Core example

```csharp
// Program.cs — illustrative reference provider (.NET 8 minimal API)
using System.Text.Json.Serialization;
using System.Text.RegularExpressions;

var builder = WebApplication.CreateBuilder(args);
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.Never);
var app = builder.Build();

// Your index. In production this is a database of crowd‑sourced avatars.
var avatars = new List<Avatar>
{
    new()
    {
        Id = "avtr_11111111-1111-1111-1111-111111111111", // REQUIRED
        Name = "Neon Fox",
        Description = "A glowing fox avatar",
        AuthorId = "usr_22222222-2222-2222-2222-222222222222",
        AuthorName = "FoxMaker",
        ImageUrl = "https://api.vrchat.cloud/api/1/file/file_aaaa/1/file",
        ThumbnailImageUrl = "https://api.vrchat.cloud/api/1/image/file_aaaa/1/256",
        ReleaseStatus = "public",
        CreatedAt = "2023-02-14T12:00:00.0000000Z",
        UpdatedAt = "2023-08-01T09:30:00.0000000Z"
    }
};

// Pull the VRChat file ID out of an image URL: .../file/file_aaaa/1/file -> file_aaaa
static string? FileIdOf(string url) =>
    Regex.Match(url ?? "", "file_[0-9A-Za-z-]+") is { Success: true } m ? m.Value : null;

app.MapGet("/avatars", (string? search, string? authorId, string? fileId, int? n) =>
{
    // Mode 3: file ID lookup -> single object (or 404 to opt out)
    if (fileId is not null)
    {
        var hit = avatars.FirstOrDefault(a => FileIdOf(a.ImageUrl) == fileId);
        return hit is null ? Results.NotFound() : Results.Json(hit);
    }

    // Mode 2: author lookup -> array
    if (authorId is not null)
        return Results.Json(avatars.Where(a => a.AuthorId == authorId));

    // Mode 1: search -> array (respect the n cap)
    if (search is not null)
    {
        var limit = n ?? 5000;
        var hits = avatars
            .Where(a => a.Name.Contains(search, StringComparison.OrdinalIgnoreCase))
            .Take(limit);
        return Results.Json(hits);
    }

    return Results.BadRequest(new { error = "missing query parameter" });
});

app.Run("http://0.0.0.0:3000");

// VRCX expects VRChat's lowerCamelCase JSON keys, so map the property names explicitly.
class Avatar
{
    [JsonPropertyName("id")]                public string Id { get; set; } = "";
    [JsonPropertyName("name")]              public string Name { get; set; } = "";
    [JsonPropertyName("description")]       public string Description { get; set; } = "";
    [JsonPropertyName("authorId")]          public string AuthorId { get; set; } = "";
    [JsonPropertyName("authorName")]        public string AuthorName { get; set; } = "";
    [JsonPropertyName("imageUrl")]          public string ImageUrl { get; set; } = "";
    [JsonPropertyName("thumbnailImageUrl")] public string ThumbnailImageUrl { get; set; } = "";
    [JsonPropertyName("releaseStatus")]     public string ReleaseStatus { get; set; } = "public";
    [JsonPropertyName("created_at")]        public string CreatedAt { get; set; } = "0001-01-01T00:00:00.0000000Z";
    [JsonPropertyName("updated_at")]        public string UpdatedAt { get; set; } = "0001-01-01T00:00:00.0000000Z";
}
```

---

## Behavior & edge cases

- **HTTPS recommended.** VRCX will call whatever URL you configure, but a public provider should
  use HTTPS.
- **De‑duplication is by `id`.** When multiple providers are configured, VRCX merges their author/
  file results into one map keyed by avatar `id`; the first occurrence wins. Ensure `id` is stable
  and unique.
- **`fileId` mode is best‑effort.** Not all providers implement it. Returning a non‑200 (or a
  non‑object body) makes VRCX quietly fall back to author lookup + local `imageUrl` matching. So a
  correct `imageUrl` on your avatar records still enables "find avatar" even without `fileId` support.
- **`search` uses only the primary provider**, while `authorId`/`fileId` fan out to **all**
  configured providers.
- **The `n` cap.** VRCX sends `n=5000` on search. Honor it (or cap lower) so you don't return
  unbounded payloads.
- **Errors are user‑visible for search/author.** A failed `search` or `authorId` request pops an
  error toast in VRCX; keep error responses clean (proper status codes) to avoid confusing users.
- **Timestamps** should be ISO 8601 strings. Missing ones default to `0001-01-01T00:00:00.0000000Z`.
- **No auth.** There is no bearer token. Use the `VRCX-ID` and `Referer` headers if you need basic
  rate‑limiting or abuse mitigation.

---

## Quick checklist

Building a provider? Make sure your endpoint:

- [ ] Responds to `GET` with `?search=…&n=…`, `?authorId=…`, and (optionally) `?fileId=…`.
- [ ] Returns HTTP `200` with valid JSON.
- [ ] Returns a **JSON array** for `search` and `authorId`.
- [ ] Returns a **single JSON object** for `fileId` (or a non‑200 to opt out).
- [ ] Gives every avatar a unique, stable **`id`** (`avtr_…`).
- [ ] Populates `name` (searchable), `authorId`, and `imageUrl` where possible.
- [ ] Respects the `n` result cap.
- [ ] Uses ISO 8601 timestamps.

---

## FAQ

**Does VRChat provide this?**
No. VRChat's API has no avatar search. Providers are community‑run indexes that VRCX queries.

**Do I have to support all three query modes?**
`search` and `authorId` are the core. `fileId` is optional — VRCX falls back to `authorId` +
`imageUrl` matching if it's unsupported.

**Where do the avatar records come from?**
That's up to your implementation — providers are typically crowd‑sourced (users submit avatars
they encounter). This document only defines the *interface* VRCX expects, not how you populate it.

**Why lowercase `id`?**
VRCX spreads your object over its defaults and keys results by the lowercase `id` field, matching
VRChat's own JSON shape. Return `id` (lowercase).

---

## Source

Documented from VRCX's client code:
[`src/coordinators/avatarCoordinator.js`](https://github.com/vrcx-team/VRCX/blob/master/src/coordinators/avatarCoordinator.js)
(functions `lookupAvatars`, `lookupAvatarsByAuthor`, `lookupAvatarByFileId`,
`lookupAvatarByImageFileId`, and `checkAvatarCacheRemote`).

VRCX is licensed under the MIT License. This is unofficial, community documentation.
```
