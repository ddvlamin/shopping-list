# Shopping List REST API — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a REST API for shopping lists with recipe markdown upload and ingredient parsing.

**Architecture:** FastAPI app with async SQLAlchemy + SQLite. Recipe parser is a pure function module. Three router modules for lists, items, and the upload-recipe endpoint. Tests use async in-memory SQLite via pytest-asyncio.

**Tech Stack:** Python 3.12+, FastAPI, SQLAlchemy (async), aiosqlite, Pydantic, PyYAML, pytest-asyncio, httpx.

---

### Task 1: Update Dependencies and Install

**Files:**
- Modify: `pyproject.toml`
- File to create after install: `uv.lock` (auto)

- [ ] **Step 1: Update pyproject.toml**

Replace existing dependencies with:

```toml
[project]
name = "shopping-list"
version = "0.1.0"
description = "Shopping list REST API with recipe parsing"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "sqlalchemy[asyncio]>=2.0.0",
    "aiosqlite>=0.20.0",
    "pydantic>=2.0.0",
    "python-multipart>=0.0.12",
    "pyyaml>=6.0",
    "python-dotenv>=1.0.0"
]

[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "httpx>=0.27.0",
    "ipykernel"
]
```

- [ ] **Step 2: Install dependencies**

Run: `uv sync` (from workspace root)
Expected: All deps installed, `uv.lock` updated.

- [ ] **Step 3: Verify import works**

Run: `uv run python -c "import fastapi; import sqlalchemy; import aiosqlite; import yaml; print('OK')"`
Expected: Prints `OK`.

- [ ] **Step 4: Commit**

```bash
git add pyproject.toml uv.lock
git commit -m "chore: add REST API dependencies (fastapi, sqlalchemy, aiosqlite, pyyaml)"
```

---

### Task 2: Database Engine and Session

**Files:**
- Create: `app/__init__.py` (empty)
- Create: `app/database.py`
- Create: `app/models.py`

- [ ] **Step 1: Create `app/__init__.py`**

Empty file.

- [ ] **Step 2: Create `app/database.py`**

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

DATABASE_URL = "sqlite+aiosqlite:///shopping_list.db"

engine = create_async_engine(DATABASE_URL, echo=False)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


async def get_db():
    async with async_session() as session:
        yield session
```

- [ ] **Step 3: Create `app/models.py`**

```python
import datetime
from sqlalchemy import ForeignKey, String, Boolean, DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class ShoppingList(Base):
    __tablename__ = "shopping_lists"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime, server_default=func.now()
    )
    updated_at: Mapped[datetime.datetime] = mapped_column(
        DateTime, server_default=func.now(), onupdate=func.now()
    )

    items: Mapped[list["ShoppingListItem"]] = relationship(
        back_populates="shopping_list", cascade="all, delete-orphan"
    )


class ShoppingListItem(Base):
    __tablename__ = "shopping_list_items"

    id: Mapped[int] = mapped_column(primary_key=True)
    shopping_list_id: Mapped[int] = mapped_column(
        ForeignKey("shopping_lists.id", ondelete="CASCADE"), nullable=False
    )
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    quantity: Mapped[str | None] = mapped_column(String(50), nullable=True)
    unit: Mapped[str | None] = mapped_column(String(50), nullable=True)
    is_checked: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime, server_default=func.now()
    )

    shopping_list: Mapped["ShoppingList"] = relationship(back_populates="items")
```

- [ ] **Step 4: Verify models import**

Run: `uv run python -c "from app.database import engine; from app.models import ShoppingList, ShoppingListItem; print('OK')"`
Expected: `OK`

- [ ] **Step 5: Commit**

```bash
git add app/__init__.py app/database.py app/models.py
git commit -m "feat: add database engine and ORM models"
```

---

### Task 3: Pydantic Schemas

**Files:**
- Create: `app/schemas.py`

- [ ] **Step 1: Create `app/schemas.py`**

```python
import datetime
from pydantic import BaseModel
from typing import Optional


class ShoppingListCreate(BaseModel):
    name: str


class ShoppingListResponse(BaseModel):
    id: int
    name: str
    created_at: datetime.datetime
    updated_at: datetime.datetime

    model_config = {"from_attributes": True}


class ShoppingListItemCreate(BaseModel):
    name: str
    quantity: Optional[str] = None
    unit: Optional[str] = None


class BulkItemsCreate(BaseModel):
    items: list[ShoppingListItemCreate]


class ShoppingListItemResponse(BaseModel):
    id: int
    shopping_list_id: int
    name: str
    quantity: Optional[str] = None
    unit: Optional[str] = None
    is_checked: bool
    created_at: datetime.datetime

    model_config = {"from_attributes": True}


class ShoppingListItemUpdate(BaseModel):
    name: Optional[str] = None
    quantity: Optional[str] = None
    unit: Optional[str] = None
    is_checked: Optional[bool] = None


class ShoppingListDetailResponse(ShoppingListResponse):
    items: list[ShoppingListItemResponse] = []
```

- [ ] **Step 2: Verify import**

Run: `uv run python -c "from app.schemas import ShoppingListCreate, ShoppingListDetailResponse; print('OK')"`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add app/schemas.py
git commit -m "feat: add Pydantic request/response schemas"
```

---

### Task 4: Recipe Parser (TDD)

**Files:**
- Create: `app/recipe_parser.py`
- Create: `tests/__init__.py` (empty)
- Create: `tests/test_recipe_parser.py`

- [ ] **Step 1: Create test file with sample recipe and parser tests**

Create `tests/__init__.py` (empty file).

Create `tests/test_recipe_parser.py`:

```python
import pytest
from app.recipe_parser import parse_recipe, ParsedIngredient, ParsedRecipe


SAMPLE_RECIPE = """---
type: recipe
cuisine: Thai
protein: Chicken
difficulty: Medium
source: https://example.com/recipe
tags:
  - dinner
  - spicy
---

# Thai Red Curry

## Ingredients
- chicken breast: 2 pieces
- coconut milk: 400 ml
- red curry paste: 2 tbsp
- fish sauce: 1 tbsp
"""

RECIPE_WITHOUT_FRONTMATTER = """# Simple Soup

## Ingredients
- salt: 1 tsp
- water: 500 ml
"""

RECIPE_NO_INGREDIENTS = """# Empty Recipe

## Instructions
1. Do nothing.
"""

RECIPE_EDGE_CASES = """# Edge Cases

## Ingredients
- : 2.7 lime leaves
- (optional) garnish: 1 sprig
- salt: 1
- empty_colon:
"""


class TestParseRecipe:
    def test_parses_full_recipe(self):
        result = parse_recipe(SAMPLE_RECIPE)
        assert result.title == "Thai Red Curry"
        assert result.cuisine == "Thai"
        assert result.protein == "Chicken"
        assert result.difficulty == "Medium"
        assert result.source_url == "https://example.com/recipe"
        assert result.tags == ["dinner", "spicy"]
        assert len(result.ingredients) == 4
        assert result.ingredients[0].name == "chicken breast"
        assert result.ingredients[0].quantity == "2"
        assert result.ingredients[0].unit == "pieces"

    def test_parses_recipe_without_frontmatter(self):
        result = parse_recipe(RECIPE_WITHOUT_FRONTMATTER)
        assert result.title == "Simple Soup"
        assert result.cuisine is None
        assert len(result.ingredients) == 2

    def test_no_ingredients_section_returns_empty_list(self):
        result = parse_recipe(RECIPE_NO_INGREDIENTS)
        assert result.ingredients == []

    def test_handles_edge_cases(self):
        result = parse_recipe(RECIPE_EDGE_CASES)
        names = [i.name for i in result.ingredients]
        assert "" in names  # empty name before colon
        assert "salt" in names


class TestParsedModels:
    def test_parsed_ingredient_defaults(self):
        ing = ParsedIngredient(name="test")
        assert ing.quantity is None
        assert ing.unit is None

    def test_parsed_recipe_defaults(self):
        recipe = ParsedRecipe(title="Test", ingredients=[])
        assert recipe.tags == []
        assert recipe.cuisine is None
```

- [ ] **Step 2: Write failing test run**

Run: `uv run pytest tests/test_recipe_parser.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.recipe_parser'`

- [ ] **Step 3: Create `app/recipe_parser.py`**

```python
import re
import yaml
from typing import Optional
from pydantic import BaseModel


class ParsedIngredient(BaseModel):
    name: str
    quantity: Optional[str] = None
    unit: Optional[str] = None


class ParsedRecipe(BaseModel):
    title: str
    cuisine: Optional[str] = None
    protein: Optional[str] = None
    difficulty: Optional[str] = None
    source_url: Optional[str] = None
    tags: list[str] = []
    ingredients: list[ParsedIngredient]


def parse_recipe(markdown: str) -> ParsedRecipe:
    frontmatter = _parse_frontmatter(markdown)
    body = _strip_frontmatter(markdown)
    title = _parse_title(body)
    ingredients = _parse_ingredients(body)
    return ParsedRecipe(
        title=title,
        cuisine=frontmatter.get("cuisine"),
        protein=frontmatter.get("protein"),
        difficulty=frontmatter.get("difficulty"),
        source_url=frontmatter.get("source"),
        tags=frontmatter.get("tags", []),
        ingredients=ingredients,
    )


def _parse_frontmatter(markdown: str) -> dict:
    match = re.match(r"^---\s*\n(.*?)\n---", markdown, re.DOTALL)
    if match:
        try:
            result = yaml.safe_load(match.group(1))
            return result if isinstance(result, dict) else {}
        except yaml.YAMLError:
            return {}
    return {}


def _strip_frontmatter(markdown: str) -> str:
    return re.sub(r"^---\s*\n.*?\n---\s*\n", "", markdown, flags=re.DOTALL)


def _parse_title(body: str) -> str:
    match = re.search(r"^#\s+(.+)$", body, re.MULTILINE)
    return match.group(1).strip() if match else ""


def _parse_ingredients(body: str) -> list[ParsedIngredient]:
    heading_match = re.search(
        r"^##\s.*(?:Ingrediënten|Ingredients|Ingredientes).*$",
        body,
        re.MULTILINE | re.IGNORECASE,
    )
    if not heading_match:
        return []

    start = heading_match.end()
    end_match = re.search(r"^##\s", body[start:], re.MULTILINE)
    section = body[start : start + end_match.start()] if end_match else body[start:]

    ingredients = []
    for line in section.split("\n"):
        line = line.strip()
        m = re.match(r"^-\s+(.+)$", line)
        if not m:
            continue
        content = m.group(1).strip()
        parsed = _parse_ingredient_line(content)
        if parsed is not None:
            ingredients.append(parsed)
    return ingredients


def _parse_ingredient_line(line: str) -> Optional[ParsedIngredient]:
    line = re.sub(r"^\([^)]*\)\s*", "", line)
    if ":" not in line:
        return ParsedIngredient(name=line.strip()) if line.strip() else None

    name_part, rest = line.rsplit(":", 1)
    name = name_part.strip()
    rest = rest.strip()

    parts = rest.split(None, 1)
    if parts:
        quantity = parts[0]
        unit = parts[1] if len(parts) > 1 else None
    else:
        quantity = None
        unit = None

    return ParsedIngredient(name=name, quantity=quantity, unit=unit)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_recipe_parser.py -v`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add app/recipe_parser.py tests/__init__.py tests/test_recipe_parser.py
git commit -m "feat: add recipe markdown parser with tests"
```

---

### Task 5: Test Database Fixtures (conftest)

**Files:**
- Create: `tests/conftest.py`

- [ ] **Step 1: Create `tests/conftest.py`**

```python
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

from app.database import get_db
from app.models import Base
from app.main import app


TEST_DATABASE_URL = "sqlite+aiosqlite://"

@pytest_asyncio.fixture
async def db_session():
    engine = create_async_engine(TEST_DATABASE_URL, echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async_session_factory = async_sessionmaker(
        engine, class_=AsyncSession, expire_on_commit=False
    )
    async with async_session_factory() as session:
        yield session
    await engine.dispose()


@pytest_asyncio.fixture
async def client(db_session):
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()


@pytest_asyncio.fixture
async def sample_list(db_session):
    from app.models import ShoppingList

    lst = ShoppingList(name="Test List")
    db_session.add(lst)
    await db_session.commit()
    await db_session.refresh(lst)
    return lst
```

- [ ] **Step 2: Just create the file, we'll verify it when we run tests later.**

- [ ] **Step 3: Commit**

```bash
git add tests/conftest.py
git commit -m "feat: add test fixtures with in-memory SQLite"
```

---

### Task 6: Lists Router (TDD)

**Files:**
- Create: `app/routers/__init__.py` (empty)
- Create: `app/routers/lists.py`
- Create: `tests/test_lists.py`

- [ ] **Step 1: Write list endpoint tests**

Create `tests/test_lists.py`:

```python
import pytest
from httpx import AsyncClient


class TestCreateList:
    @pytest.mark.asyncio
    async def test_create_list(self, client: AsyncClient):
        response = await client.post("/api/lists", json={"name": "Weekend BBQ"})
        assert response.status_code == 200
        data = response.json()["data"]
        assert data["name"] == "Weekend BBQ"
        assert "id" in data
        assert "created_at" in data

    @pytest.mark.asyncio
    async def test_create_list_empty_name_returns_422(self, client: AsyncClient):
        response = await client.post("/api/lists", json={"name": ""})
        assert response.status_code == 422


class TestGetLists:
    @pytest.mark.asyncio
    async def test_list_lists(self, client: AsyncClient):
        await client.post("/api/lists", json={"name": "List 1"})
        await client.post("/api/lists", json={"name": "List 2"})
        response = await client.get("/api/lists")
        assert response.status_code == 200
        data = response.json()["data"]
        assert len(data) == 2

    @pytest.mark.asyncio
    async def test_list_lists_empty(self, client: AsyncClient):
        response = await client.get("/api/lists")
        assert response.status_code == 200
        assert response.json()["data"] == []


class TestGetListDetail:
    @pytest.mark.asyncio
    async def test_get_list_with_items(self, client: AsyncClient, sample_list):
        await client.post(
            f"/api/lists/{sample_list.id}/items",
            json={"items": [{"name": "milk", "quantity": "2", "unit": "l"}]},
        )
        response = await client.get(f"/api/lists/{sample_list.id}")
        assert response.status_code == 200
        data = response.json()["data"]
        assert data["name"] == "Test List"
        assert len(data["items"]) == 1
        assert data["items"][0]["name"] == "milk"

    @pytest.mark.asyncio
    async def test_get_list_not_found(self, client: AsyncClient):
        response = await client.get("/api/lists/999")
        assert response.status_code == 404


class TestUploadRecipe:
    @pytest.mark.asyncio
    async def test_upload_recipe_adds_items(self, client: AsyncClient, sample_list):
        recipe_content = "---\ncuisine: Thai\n---\n# Curry\n\n## Ingredients\n- basil: 10 leaves\n"
        response = await client.post(
            f"/api/lists/{sample_list.id}/upload-recipe",
            files={"file": ("recipe.md", recipe_content, "text/markdown")},
        )
        assert response.status_code == 200
        data = response.json()["data"]
        assert data["title"] == "Curry"
        assert data["cuisine"] == "Thai"
        assert len(data["ingredients_added"]) == 1
        assert data["ingredients_added"][0]["name"] == "basil"

    @pytest.mark.asyncio
    async def test_upload_non_md_file_returns_400(self, client: AsyncClient, sample_list):
        response = await client.post(
            f"/api/lists/{sample_list.id}/upload-recipe",
            files={"file": ("recipe.txt", "plain text", "text/plain")},
        )
        assert response.status_code == 400

    @pytest.mark.asyncio
    async def test_upload_recipe_list_not_found(self, client: AsyncClient):
        response = await client.post(
            "/api/lists/999/upload-recipe",
            files={"file": ("recipe.md", "# Test\n\n## Ingredients\n- salt: 1 tsp", "text/markdown")},
        )
        assert response.status_code == 404
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_lists.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.routers'`

- [ ] **Step 3: Create `app/routers/__init__.py`**

Empty file.

- [ ] **Step 4: Create `app/routers/lists.py`**

```python
from fastapi import APIRouter, Depends, HTTPException, UploadFile, File
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.database import get_db
from app.models import ShoppingList, ShoppingListItem
from app.schemas import (
    ShoppingListCreate,
    ShoppingListResponse,
    ShoppingListDetailResponse,
    BulkItemsCreate,
    ShoppingListItemResponse,
)
from app.recipe_parser import parse_recipe

router = APIRouter()


@router.post("/api/lists", response_model=dict)
async def create_list(body: ShoppingListCreate, db: AsyncSession = Depends(get_db)):
    lst = ShoppingList(name=body.name)
    db.add(lst)
    await db.commit()
    await db.refresh(lst)
    return {"data": ShoppingListResponse.model_validate(lst)}


@router.get("/api/lists", response_model=dict)
async def list_lists(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(ShoppingList).order_by(ShoppingList.created_at.desc()))
    lists = result.scalars().all()
    return {"data": [ShoppingListResponse.model_validate(lst) for lst in lists]}


@router.get("/api/lists/{list_id}", response_model=dict)
async def get_list(list_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(ShoppingList).where(ShoppingList.id == list_id)
    )
    lst = result.scalar_one_or_none()
    if lst is None:
        raise HTTPException(status_code=404, detail="List not found")
    return {"data": ShoppingListDetailResponse.model_validate(lst)}


@router.post("/api/lists/{list_id}/items", response_model=dict)
async def add_items(
    list_id: int, body: BulkItemsCreate, db: AsyncSession = Depends(get_db)
):
    result = await db.execute(select(ShoppingList).where(ShoppingList.id == list_id))
    lst = result.scalar_one_or_none()
    if lst is None:
        raise HTTPException(status_code=404, detail="List not found")

    items = [
        ShoppingListItem(shopping_list_id=list_id, **item.model_dump())
        for item in body.items
    ]
    db.add_all(items)
    await db.commit()
    for item in items:
        await db.refresh(item)
    return {
        "data": [ShoppingListItemResponse.model_validate(item) for item in items]
    }


@router.post("/api/lists/{list_id}/upload-recipe", response_model=dict)
async def upload_recipe(
    list_id: int, file: UploadFile = File(...), db: AsyncSession = Depends(get_db)
):
    if not file.filename or not file.filename.endswith(".md"):
        raise HTTPException(status_code=400, detail="Only .md files are accepted")

    result = await db.execute(select(ShoppingList).where(ShoppingList.id == list_id))
    lst = result.scalar_one_or_none()
    if lst is None:
        raise HTTPException(status_code=404, detail="List not found")

    content = await file.read()
    try:
        recipe = parse_recipe(content.decode("utf-8"))
    except Exception:
        raise HTTPException(status_code=400, detail="Could not parse recipe file")

    if not recipe.ingredients:
        raise HTTPException(
            status_code=400, detail="No ingredients found in recipe"
        )

    items = [
        ShoppingListItem(
            shopping_list_id=list_id,
            name=ing.name,
            quantity=ing.quantity,
            unit=ing.unit,
        )
        for ing in recipe.ingredients
    ]
    db.add_all(items)
    await db.commit()
    for item in items:
        await db.refresh(item)

    return {
        "data": {
            "title": recipe.title,
            "cuisine": recipe.cuisine,
            "protein": recipe.protein,
            "difficulty": recipe.difficulty,
            "source_url": recipe.source_url,
            "tags": recipe.tags,
            "ingredients_added": [
                ShoppingListItemResponse.model_validate(item) for item in items
            ],
        }
    }
```

- [ ] **Step 5: Run tests**

Run: `uv run pytest tests/test_lists.py -v`
Expected: Most tests pass or some fail due to missing `app.main` (which we haven't created yet). This is OK — we'll wire it up next.

- [ ] **Step 6: Commit**

```bash
git add app/routers/__init__.py app/routers/lists.py tests/test_lists.py
git commit -m "feat: add lists router with CRUD and recipe upload"
```

---

### Task 7: Items Router (TDD)

**Files:**
- Create: `app/routers/items.py`
- Create: `tests/test_items.py`

- [ ] **Step 1: Write items endpoint tests**

Create `tests/test_items.py`:

```python
import pytest
from httpx import AsyncClient


class TestUpdateItem:
    @pytest.mark.asyncio
    async def test_toggle_item_checked(self, client: AsyncClient, sample_list):
        add_resp = await client.post(
            f"/api/lists/{sample_list.id}/items",
            json={"items": [{"name": "milk", "quantity": "1"}]},
        )
        item_id = add_resp.json()["data"][0]["id"]

        response = await client.patch(
            f"/api/items/{item_id}", json={"is_checked": True}
        )
        assert response.status_code == 200
        data = response.json()["data"]
        assert data["is_checked"] is True
        assert data["name"] == "milk"

    @pytest.mark.asyncio
    async def test_update_item_name(self, client: AsyncClient, sample_list):
        add_resp = await client.post(
            f"/api/lists/{sample_list.id}/items",
            json={"items": [{"name": "milk"}]},
        )
        item_id = add_resp.json()["data"][0]["id"]

        response = await client.patch(
            f"/api/items/{item_id}", json={"name": "oat milk"}
        )
        assert response.status_code == 200
        assert response.json()["data"]["name"] == "oat milk"

    @pytest.mark.asyncio
    async def test_update_item_not_found(self, client: AsyncClient):
        response = await client.patch("/api/items/999", json={"name": "x"})
        assert response.status_code == 404


class TestDeleteItem:
    @pytest.mark.asyncio
    async def test_delete_item(self, client: AsyncClient, sample_list):
        add_resp = await client.post(
            f"/api/lists/{sample_list.id}/items",
            json={"items": [{"name": "eggs"}]},
        )
        item_id = add_resp.json()["data"][0]["id"]

        delete_resp = await client.delete(f"/api/items/{item_id}")
        assert delete_resp.status_code == 200

        get_resp = await client.get(f"/api/lists/{sample_list.id}")
        assert len(get_resp.json()["data"]["items"]) == 0

    @pytest.mark.asyncio
    async def test_delete_item_not_found(self, client: AsyncClient):
        response = await client.delete("/api/items/999")
        assert response.status_code == 404
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_items.py -v`
Expected: FAIL — router not created yet

- [ ] **Step 3: Create `app/routers/items.py`**

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.database import get_db
from app.models import ShoppingListItem
from app.schemas import ShoppingListItemUpdate, ShoppingListItemResponse

router = APIRouter()


@router.patch("/api/items/{item_id}", response_model=dict)
async def update_item(
    item_id: int, body: ShoppingListItemUpdate, db: AsyncSession = Depends(get_db)
):
    result = await db.execute(
        select(ShoppingListItem).where(ShoppingListItem.id == item_id)
    )
    item = result.scalar_one_or_none()
    if item is None:
        raise HTTPException(status_code=404, detail="Item not found")

    update_data = body.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(item, field, value)
    await db.commit()
    await db.refresh(item)
    return {"data": ShoppingListItemResponse.model_validate(item)}


@router.delete("/api/items/{item_id}", response_model=dict)
async def delete_item(item_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(ShoppingListItem).where(ShoppingListItem.id == item_id)
    )
    item = result.scalar_one_or_none()
    if item is None:
        raise HTTPException(status_code=404, detail="Item not found")

    await db.delete(item)
    await db.commit()
    return {"data": {"deleted": True}}
```

- [ ] **Step 4: Run tests**

Run: `uv run pytest tests/test_items.py -v`
Expected: Tests may still fail due to missing `app.main` — that's expected.

- [ ] **Step 5: Commit**

```bash
git add app/routers/items.py tests/test_items.py
git commit -m "feat: add items router with update and delete"
```

---

### Task 8: Wire Up Main App

**Files:**
- Create: `app/main.py`

- [ ] **Step 1: Create `app/main.py`**

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.database import engine
from app.models import Base
from app.routers import lists, items


@asynccontextmanager
async def lifespan(app: FastAPI):
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield


app = FastAPI(title="Shopping List API", lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(lists.router)
app.include_router(items.router)
```

- [ ] **Step 2: Run all tests to verify everything passes**

Run: `uv run pytest tests/ -v`
Expected: All tests PASS

- [ ] **Step 3: Manual smoke test**

```bash
uv run uvicorn app.main:app --reload &
sleep 2
# Create a list
curl -s -X POST http://localhost:8000/api/lists -H "Content-Type: application/json" -d '{"name":"Groceries"}'
# Add items
curl -s -X POST http://localhost:8000/api/lists/1/items -H "Content-Type: application/json" -d '{"items":[{"name":"milk","quantity":"2","unit":"l"},{"name":"eggs","quantity":"6"}]}'
# Get list with items
curl -s http://localhost:8000/api/lists/1
# Toggle item
curl -s -X PATCH http://localhost:8000/api/items/1 -H "Content-Type: application/json" -d '{"is_checked":true}'
# Upload recipe
curl -s -X POST http://localhost:8000/api/lists/1/upload-recipe -F "file=@sample_recipe.md"
kill %1
```

Expected: All requests return 200 with correct data.

- [ ] **Step 4: Commit**

```bash
git add app/main.py
git commit -m "feat: wire up FastAPI app with routers and lifespan"
```

---

### Task 9: Sample Recipe File and README

**Files:**
- Create: `sample_recipe.md`
- Modify: `README.md`

- [ ] **Step 1: Create `sample_recipe.md`**

```markdown
---
type: recipe
cuisine: Italian
protein: Chicken
difficulty: Easy
source: https://example.com/parmesan-chicken
tags:
  - dinner
  - italian
---

# Parmesan Chicken

## Ingredients
- chicken breast: 2 pieces
- breadcrumbs: 100 g
- parmesan cheese: 50 g
- eggs: 2
- salt: 1 tsp
- olive oil: 3 tbsp
```

- [ ] **Step 2: Update `README.md` with API docs**

Write a concise README covering:
- Project description
- Quick start (uv sync, uv run uvicorn app.main:app)
- Available endpoints table
- Example: uploading the sample recipe
- Link to OpenAPI docs at `/docs`

- [ ] **Step 3: Commit**

```bash
git add sample_recipe.md README.md
git commit -m "docs: add sample recipe and README with API docs"
```

---

## Self-Review Checklist

1. **Spec coverage:** Every spec requirement maps to a task:
   - Create shopping list → Task 6 (POST /api/lists)
   - Add items → Task 6 (POST /api/lists/{id}/items)
   - List items → Task 6 (GET /api/lists/{id})
   - Upload recipe markdown → Task 6 (POST /api/lists/{id}/upload-recipe)
   - Parse ingredients from markdown → Task 4 (recipe_parser.py)
   - Update/toggle/delete items → Task 7 (PATCH/DELETE /api/items/{id})
   - Persistent SQLite storage → Task 2 (database.py, models.py)
   - Pydantic validation → Task 3 (schemas.py)
   - Test coverage → Tasks 4, 6, 7 (test files)

2. **Placeholder scan:** No TBD, TODO, or incomplete sections.

3. **Type consistency:** All Pydantic models reference the same field names. Router function signatures match schemas. No type drift between tasks.

4. **Scope:** Focused on one API with ~7 endpoints and a recipe parser. No extra subsystems.
```
