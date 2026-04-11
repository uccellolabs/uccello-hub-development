# uccello-hub-development

A Claude Code plugin providing a skill for interacting with the [Uccello Hub](https://uccellolabs.com) REST API.

## What it does

When installed, Claude automatically activates this skill whenever you work with Uccello Hub data — creating records, filtering lists, checking schema, or wiring up CRUD operations. It enforces the core rule: **never create local database migrations for business data**, always use the Uccello Hub API.

## Installation

```bash
claude plugin install github:YOUR_USERNAME/uccello-hub-development
```

Or add it to your project's plugin configuration.

## Setup

Add to your `.env`:

```env
UCCELLO_HUB_URL=https://your-hub.uccello.app
UCCELLO_HUB_API_KEY=your_api_key
```

Add to `config/services.php`:

```php
'uccello_hub' => [
    'url'     => env('UCCELLO_HUB_URL'),
    'api_key' => env('UCCELLO_HUB_API_KEY'),
],
```

## Contents

```
skills/
└── uccello-hub-development/
    └── SKILL.md    # Model-invoked skill with API conventions and code patterns
```

## License

MIT
