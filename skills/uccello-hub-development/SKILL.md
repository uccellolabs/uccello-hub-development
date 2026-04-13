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

When a new table or column is needed, write an **Uccello Hub migration** — a standard Laravel migration file whose `up()` and `down()` methods call the Uccello Hub Schema API instead of `Schema::create()`.

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

### Schema Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/schema` | List all available tables |
| `POST` | `/api/v1/schema/tables` | Create a new custom table |
| `PUT` | `/api/v1/schema/tables/{slug}` | Update a custom table |
| `DELETE` | `/api/v1/schema/tables/{slug}` | Delete a custom table |

### Data CRUD

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/{slug}` | List records |
| `POST` | `/api/v1/{slug}` | Create a record |
| `GET` | `/api/v1/{slug}/{ulid}` | Get a single record |
| `PUT` | `/api/v1/{slug}/{ulid}` | Update a record |
| `DELETE` | `/api/v1/{slug}/{ulid}` | Delete a record |

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

## Schema API — Column Types & Payloads

### Column properties

| Property | Required | Description |
|----------|----------|-------------|
| `name` | yes | Display name |
| `type` | yes | See types below |
| `required` | no | `true`/`false` (default `false`) |
| `options` | no | Type-specific options (see below) |
| `id` | PUT only | UUID of the existing column |

### Available types

| Type | `options` |
|------|-----------|
| `text` | — |
| `number` | — |
| `date` | — |
| `file` | — |
| `select` | `choices: string[]` |
| `relation` | `to_table_id`, `display_column_slug` |
| `formula` | `expression` (max 2000 chars) |

### POST payload — create table

The slug is **auto-generated** from `name`. Do not send a `slug` field.

```json
{
  "name": "Contacts",
  "columns": [
    { "name": "First Name", "type": "text",     "required": true },
    { "name": "Age",        "type": "number" },
    { "name": "Status",     "type": "select",   "options": { "choices": ["active", "inactive"] } },
    { "name": "Birth Date", "type": "date" },
    { "name": "Avatar",     "type": "file" },
    { "name": "Company",    "type": "relation", "options": { "to_table_id": "<uuid>", "display_column_slug": "name" } },
    { "name": "Full Name",  "type": "formula",  "options": { "expression": "first_name || ' ' || last_name" } }
  ]
}
```

### PUT payload — update table

- `name` is **not modifiable**.
- Existing columns **must include their `id`** (UUID); omitting `id` creates a new column.
- Columns **absent from the payload are deleted**.
- The array order sets the display order.
- The **type of an existing column cannot be changed** (API returns 422).

```json
{
  "columns": [
    { "id": "existing-uuid", "name": "First Name", "type": "text", "required": true },
    { "name": "Phone",       "type": "text" }
  ]
}
```

## Uccello Hub Migrations

Instead of using `Schema::create()` or `Schema::table()`, write migration files that call the Uccello Hub Schema API. Place them in `database/migrations/` like any Laravel migration.

### Pattern 1 — Create a new table

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\Http;

return new class extends Migration
{
    public function up(): void
    {
        Http::uccelloHub()->post('/api/v1/schema/tables', [
            'name'    => 'Contacts',
            'columns' => [
                ['name' => 'First Name', 'type' => 'text', 'required' => true],
                ['name' => 'Email',      'type' => 'text'],
                ['name' => 'Status',     'type' => 'select', 'options' => ['choices' => ['active', 'inactive']]],
                ['name' => 'Company',    'type' => 'relation', 'options' => ['to_table_id' => '<companies-table-uuid>', 'display_column_slug' => 'name']],
            ],
        ])->throw();
    }

    public function down(): void
    {
        Http::uccelloHub()->delete('/api/v1/schema/tables/contacts')->throw();
    }
};
```

### Pattern 2 — Add columns to an existing table

When adding columns, `up()` sends the full column list (existing + new). `down()` restores the previous state by sending only the columns that existed before.

Existing columns **must include their UUID** (`id`). Fetch these from `GET /api/v1/schema` before writing the migration.

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\Http;

return new class extends Migration
{
    // Snapshot of columns that existed BEFORE this migration.
    // Each entry must include the UUID returned by Uccello Hub.
    private array $existingColumns = [
        ['id' => 'uuid-first-name', 'name' => 'First Name', 'type' => 'text', 'required' => true],
        ['id' => 'uuid-email',      'name' => 'Email',       'type' => 'text'],
        ['id' => 'uuid-status',     'name' => 'Status',      'type' => 'select', 'options' => ['choices' => ['active', 'inactive']]],
    ];

    public function up(): void
    {
        Http::uccelloHub()->put('/api/v1/schema/tables/contacts', [
            'columns' => [
                ...$this->existingColumns,
                ['name' => 'Phone',      'type' => 'text'],
                ['name' => 'Birth Date', 'type' => 'date'],
            ],
        ])->throw();
    }

    public function down(): void
    {
        // Restoring $existingColumns drops the columns added in up().
        Http::uccelloHub()->put('/api/v1/schema/tables/contacts', [
            'columns' => $this->existingColumns,
        ])->throw();
    }
};
```

### Workflow: Adding a New Data Structure

1. Call `GET /api/v1/schema` to check whether a suitable table already exists.
2. If not, write a **create table** migration (Pattern 1 above).
3. If the table exists but is missing columns, write an **add columns** migration (Pattern 2 above), using the column UUIDs from `GET /api/v1/schema`.
4. Run `php artisan migrate` — the migration calls the Uccello Hub API; no local schema is touched.
5. Interact with the table exclusively through the data CRUD endpoints.

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

- Using `Schema::create()` or `Schema::table()` for business data — always use Uccello Hub migrations instead.
- Sending a `slug` field in a POST payload — the slug is auto-generated from `name`; omit it.
- Omitting column `id` UUIDs in a PUT payload — existing columns without an `id` are treated as new columns.
- Attempting to change a column's `type` via PUT — the API returns 422; type changes require a new column.
- Sending an `id` field in data POST bodies — the API assigns the ULID automatically; omit it.
- Using integer IDs instead of ULIDs for relational fields.
- Reading `UCCELLO_HUB_URL` / `UCCELLO_HUB_API_KEY` directly with `env()` outside config files — always go through `config()`.
- Forgetting to check if a table already exists in the schema before creating a new one.
- Creating a local Eloquent model or migration for a business entity.
