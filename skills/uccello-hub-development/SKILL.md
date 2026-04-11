---
name: uccello-hub-development
description: "Use this skill when interacting with the Uccello Hub REST API for data storage and retrieval. Activate when: creating or reading business data, listing or filtering records, creating new tables or checking existing schema, wiring up any CRUD operation that touches the Uccello Hub API, or when the user mentions contacts, companies, tasks, dossiers, or any other Uccello Hub table. Never create local database migrations for business data — use the Uccello Hub API instead."
version: 1.0.0
license: MIT
---

# Uccello Hub Development

Uccello Hub is the single source of truth for all business data in this application. This skill governs how to interact with its REST API.

## Golden Rule

**Never create local database migrations for business data.** Sessions, jobs, cache, and other framework technical tables are the only exceptions. All business entities live in Uccello Hub.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `UCCELLO_HUB_URL` | Base URL of the Uccello Hub instance |
| `UCCELLO_HUB_API_KEY` | API key for authentication |

Read both values from the `.env` file — never hardcode them. Always access them via `config()`, not `env()`.

Expose via `config/services.php`:

```php
'uccello_hub' => [
    'url'     => env('UCCELLO_HUB_URL'),
    'api_key' => env('UCCELLO_HUB_API_KEY'),
],
```

## Authentication

Every request must include:

```
X-Api-Key: {UCCELLO_HUB_API_KEY}
```

## API Conventions

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/{slug}` | List records |
| `POST` | `/api/v1/{slug}` | Create a record |
| `GET` | `/api/v1/{slug}/{ulid}` | Get a single record |
| `PUT` | `/api/v1/{slug}/{ulid}` | Update a record |
| `DELETE` | `/api/v1/{slug}/{ulid}` | Delete a record |
| `GET` | `/api/v1/schema` | List all available tables |

### List Parameters

| Parameter | Description |
|-----------|-------------|
| `per_page` | Records per page |
| `page` | Page number |
| `search` | Full-text search |
| `sort` | Column to sort by (prefix `-` for descending) |
| `filter[{col}]` | Exact filter on a column |

### Relational Fields

Relational fields accept a **ULID** string, not a nested object:

```json
{ "company": "01arz3ndektsv4rrffq69g5fav" }
```

### Record Identifiers

All records use **ULIDs** as their primary identifier. Never assume integer IDs.

## Workflow: Adding a New Data Structure

1. Call `GET /api/v1/schema` to check if a suitable table already exists.
2. If none exists, create a new custom table via the Uccello Hub admin or API under the correct team/workspace.
3. Interact with it exclusively through the REST API — never create a local migration.

## Laravel HTTP Client Pattern

Register a named HTTP client macro in a service provider:

```php
use Illuminate\Support\Facades\Http;

Http::macro('uccelloHub', fn () => Http::withHeaders([
    'X-Api-Key' => config('services.uccello_hub.api_key'),
])->baseUrl(config('services.uccello_hub.url')));
```

### List with filters

```php
$response = Http::uccelloHub()->get('/api/v1/{table-slug}', [
    'per_page'       => 25,
    'page'           => 1,
    'search'         => $query,
    'filter[status]' => 'active',
]);
```

### Create

```php
$response = Http::uccelloHub()->post('/api/v1/{table-slug}', [
    'first_name' => $data['first_name'],
    'last_name'  => $data['last_name'],
    'email'      => $data['email'],
    'company'    => $data['company_ulid'],  // relational field → ULID
]);
```

### Update

```php
$response = Http::uccelloHub()->put("/api/v1/{table-slug}/{$ulid}", [
    'status' => 'closed',
]);
```

### Delete

```php
Http::uccelloHub()->delete("/api/v1/{table-slug}/{$ulid}");
```

## Error Handling

Always check the response status:

```php
$response->throw(); // throws on 4xx/5xx

if ($response->failed()) {
    // handle gracefully
}
```

## Common Pitfalls

- Sending an `id` field in POST bodies — the API assigns the ULID automatically; omit it.
- Using integer IDs instead of ULIDs for relational fields.
- Reading `UCCELLO_HUB_URL` / `UCCELLO_HUB_API_KEY` directly with `env()` outside config files — always go through `config()`.
- Forgetting to check if a table already exists in the schema before requesting a new one.
- Creating a local Eloquent model or migration for a business entity.
