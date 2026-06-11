# Shopping List REST API

A lightweight, asynchronous REST API for managing shopping lists and their items, powered by FastAPI and SQLAlchemy. This application also features an ephemeral markdown recipe parser that extracts ingredients from uploaded recipe files and adds them directly to your shopping list.

## Tech Stack

- **Framework:** FastAPI
- **Server:** Uvicorn
- **Database:** SQLite (Async via SQLAlchemy & aiosqlite)
- **Validation:** Pydantic
- **Testing:** Pytest (with pytest-asyncio and HTTPX)

## Quick Start

### Prerequisites

Ensure you have [uv](https://github.com/astral-sh/uv) installed.

### Setup and Installation

1. Clone the repository and navigate to the project directory:
   ```bash
   git clone <repository-url>
   cd shopping-list
   ```

2. Install the dependencies and set up the virtual environment:
   ```bash
   uv sync
   ```

### Running the Application

Start the development server using Uvicorn:
```bash
uv run uvicorn app.main:app --reload
```

The API will be accessible at `http://localhost:8000`.

### Running Tests

Run the test suite to verify the installation:
```bash
uv run pytest
```

## API Endpoints

The API interacts using JSON payloads and returns structured JSON responses.

### Shopping Lists

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/api/lists` | Create a new shopping list. |
| `GET` | `/api/lists` | List all shopping lists. |
| `GET` | `/api/lists/{id}` | Get a single shopping list with all of its items. |
| `POST` | `/api/lists/{id}/items` | Add one or more items to a shopping list in bulk. |
| `POST` | `/api/lists/{id}/upload-recipe` | Upload a markdown (`.md`) recipe file, parse its ingredients, and add them to the list. |

### Shopping List Items

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `PATCH` | `/api/items/{id}` | Update item fields (such as changing the name, quantity, unit, or toggling `is_checked`). |
| `DELETE` | `/api/items/{id}` | Remove an item from its shopping list. |

## Interactive Documentation

FastAPI automatically generates interactive documentation for the API:

- **Swagger UI (Interactive API Docs):** [http://localhost:8000/docs](http://localhost:8000/docs)
- **ReDoc (Alternative API Docs):** [http://localhost:8000/redoc](http://localhost:8000/redoc)

## Recipe Upload Example

The application supports parsing markdown recipes with YAML frontmatter. You can upload the included `sample_recipe.md` to a shopping list with ID `1` using `curl`:

```bash
curl -X POST http://localhost:8000/api/lists/1/upload-recipe \
  -F "file=@sample_recipe.md"
```

### Supported Recipe Markdown Format

Uploaded recipe files must be in markdown format (`.md`) and have the following structure:

1. **YAML Frontmatter (Optional):** Delimited by `---` lines at the top of the file. It can contain metadata fields like `cuisine`, `protein`, `difficulty`, `source`, and `tags`.
2. **Title:** The first `#` heading is parsed as the recipe title.
3. **Ingredients Section:** A heading named `## Ingredients` or `## Ingrediënten` contains a bulleted list of ingredients.
4. **Ingredient Line Format:** Bullet points should be in the format: `- ingredient name: quantity unit` (e.g., `- chicken breast: 2 pieces`).

Refer to `sample_recipe.md` for a complete example.
