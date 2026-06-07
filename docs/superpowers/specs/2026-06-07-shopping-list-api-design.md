# Shopping List REST API — Design Spec

## Overview

A lightweight REST API for managing shopping lists. Users can create lists, add items, and upload markdown recipes that are parsed for ingredients and added to a specified list.

## Tech Stack

- **Framework:** FastAPI
- **Server:** uvicorn
- **Database:** SQLite via SQLAlchemy (async)
- **Validation:** Pydantic (native to FastAPI)
- **Testing:** pytest + pytest-asyncio + httpx (test client)

## Data Model

### shopping_lists

| Column     | Type           | Notes            |
|------------|----------------|------------------|
| id         | int (PK, auto) |                  |
| name       | text           | required         |
| created_at | datetime       | server default   |
| updated_at | datetime       | on update        |

### shopping_list_items

| Column           | Type              | Notes                      |
|------------------|-------------------|----------------------------|
| id               | int (PK, auto)    |                            |
| shopping_list_id | int (FK, not null)| cascade delete             |
| name             | text              | required                   |
| quantity         | text, nullable    | e.g. "0.33", "2", "133"   |
| unit             | text, nullable    | e.g. "el", "g", "kop"     |
| is_checked       | bool              | default false              |
| created_at       | datetime          | server default             |

No dedicated `recipes` table. Recipes are parsed ephemerally — only their ingredients are persisted as items.

## API Endpoints

| Method   | Path                          | Description                                     |
|----------|-------------------------------|-------------------------------------------------|
| `POST`   | `/api/lists`                  | Create a shopping list                          |
| `GET`    | `/api/lists`                  | List all shopping lists                         |
| `GET`    | `/api/lists/{id}`             | Get a single list with its items                |
| `POST`   | `/api/lists/{id}/items`       | Add item(s) to a list                           |
| `PATCH`  | `/api/items/{id}`             | Update item (toggle checked, edit fields)       |
| `DELETE` | `/api/items/{id}`             | Remove an item                                  |
| `POST`   | `/api/lists/{id}/upload-recipe` | Upload a `.md` recipe file, parse & add items |

### Request/Response Formats

All responses use a consistent JSON envelope:

```json
{ "data": { ... } }
```

```json
{ "error": { "code": "NOT_FOUND", "message": "..." } }
```

**POST /api/lists** — body: `{ "name": "Weekend BBQ" }`

**POST /api/lists/{id}/items** — body: `{ "items": [{ "name": "milk", "quantity": "2", "unit": "l" }] }`

**POST /api/lists/{id}/upload-recipe** — multipart form with a `file` field (`.md` only)

**PATCH /api/items/{id}** — body accepts: `name`, `quantity`, `unit`, `is_checked`

## Recipe Parser

A standalone module (`app/recipe_parser.py`) with no side effects:

1. **Frontmatter** — extracts YAML between `---` delimiters. Fields: `type`, `cuisine`, `protein`, `difficulty`, `source`, `tags`. Title is taken from the first `# ` heading.
2. **Ingredient section** — finds a heading matching `## ...Ingrediënten...` or `## ...Ingredients...` (case-insensitive). Reads lines after it until the next `## ` heading.
3. **Line parsing** — each `- ` bullet is parsed with regex: optional parenthetical context before the colon is stripped, then `ingredient_name: quantity unit`. Handles: empty names, missing quantities, missing units, leading/trailing whitespace.
4. **Ignore** — instructions, nutrition tables, and any sections after ingredients.

Returns: `{ title, cuisine, protein, difficulty, source_url, tags, ingredients: [{ name, quantity, unit }] }`

## Project Structure

```
├── app/
│   ├── __init__.py
│   ├── main.py            # FastAPI app, lifespan, CORS
│   ├── database.py        # SQLAlchemy engine + session factory
│   ├── models.py          # ORM models
│   ├── schemas.py         # Pydantic request/response schemas
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── lists.py       # list CRUD + upload-recipe endpoint
│   │   └── items.py       # item CRUD endpoints
│   ├── recipe_parser.py   # markdown recipe → structured data
│   └── dependencies.py    # DB session dependency
├── tests/
│   ├── __init__.py
│   ├── conftest.py        # test DB, client fixture
│   ├── test_lists.py
│   ├── test_items.py
│   └── test_recipe_parser.py
├── pyproject.toml
└── uv.lock
```

## Dependencies to Add

- `fastapi` — web framework
- `uvicorn[standard]` — ASGI server
- `sqlalchemy[asyncio]` — ORM with async support
- `aiosqlite` — async SQLite driver
- `python-multipart` — file upload support
- `pyyaml` — YAML frontmatter parsing

## Error Handling

- **404** — list or item not found
- **422** — validation error (FastAPI auto)
- **400** — invalid file type (non-.md), unparseable recipe
- Generic 500 catch-all with structured error response

## Testing

- `pytest-asyncio` for async test support
- `httpx.AsyncClient` with FastAPI's `TestClient`-style `AsyncClient`
- In-memory SQLite for test isolation
- Fixtures: `test_db`, `client`, `sample_list`, `sample_recipe_md`

Key test cases:
- Create/get/delete lists and items
- Toggle item checked state
- Upload valid recipe → items created
- Upload invalid file → 400
- Upload recipe with no ingredient section → 400
- Recipe parser unit tests: frontmatter, ingredients, edge cases, missing fields
