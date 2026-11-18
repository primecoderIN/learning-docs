# Module 15: Async Persistence with SQLAlchemy 2.0 & pgvector

Welcome to **Module 15** of the Normora Python & AI Masterclass.

Just as Entity Framework Core (EF Core) is the enterprise ORM standard in .NET, **SQLAlchemy 2.0 (Async)** is the enterprise database standard in Python.

In an AI platform like **Normora**, the database is not just a place to store relational rows—it is where **relational data meets high-dimensional vector embeddings**.
* Multi-tenant metadata (`Tenants`, `Memberships`, `Departments`) is stored in strictly isolated PostgreSQL schemas.
* Document chunks are stored with their **768-dimensional Google Gemini vector embeddings** using **`pgvector`**.
* Queries execute lightning-fast approximate nearest neighbor (ANN) searches using **HNSW (Hierarchical Navigable Small World)** indexes.

In this module, you will master modern asynchronous SQLAlchemy 2.0, connection pooling, Alembic migrations, and pgvector operations.

Every concept is followed by **at least 5 real-world Normora examples**.

---

## 1. Modern SQLAlchemy 2.0 Declarative Models

In SQLAlchemy 2.0, models are defined using modern Python static typing with **`Mapped[]`** and **`mapped_column()`** (identical in concept to EF Core POCO entities and Data Annotations).

### Comparison: EF Core vs. SQLAlchemy 2.0
| Feature | EF Core (.NET) | SQLAlchemy 2.0 (Python) |
|---|---|---|
| Entity Definition | `public class Tenant { public Guid Id { get; set; } }` | `class Tenant(Base): id: Mapped[uuid.UUID] = mapped_column(...)` |
| Primary Key | `[Key]` or Fluent API `.HasKey(t => t.Id)` | `mapped_column(primary_key=True)` |
| Database Context | `DbContext` | `AsyncSession` |
| Execution Engine | `DbContext.Database` | `AsyncEngine` (`create_async_engine`) |
| Async Query | `await _context.Tenants.FirstOrDefaultAsync(...)` | `await session.scalar(select(Tenant).where(...))` |

### 5 Real-World Normora Examples:

#### Example 1.1: The `Tenant` Entity Model
```python
import uuid
from datetime import datetime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, Boolean, DateTime

class Base(DeclarativeBase):
    pass

class Tenant(Base):
    __tablename__ = "tenants"
    __table_args__ = {"schema": "tenants"}  # Multi-tenant schema isolation!

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    slug: Mapped[str] = mapped_column(String(50), unique=True, index=True, nullable=False)
    plan: Mapped[str] = mapped_column(String(20), default="Pro")
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

#### Example 1.2: The `Department` Entity Model
```python
from sqlalchemy import ForeignKey

class Department(Base):
    __tablename__ = "departments"
    __table_args__ = {"schema": "tenants"}

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.tenants.id"), nullable=False, index=True)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

#### Example 1.3: The `TenantMembership` Entity Model
```python
class TenantMembership(Base):
    __tablename__ = "memberships"
    __table_args__ = {"schema": "tenants"}

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.tenants.id"), nullable=False, index=True)
    user_id: Mapped[str] = mapped_column(String(100), nullable=False, index=True)
    role: Mapped[str] = mapped_column(String(20), default="Member")  # "Owner", "Admin", "Member"
```

#### Example 1.4: The `Document` Entity Model
```python
from sqlalchemy import Integer, Text

class Document(Base):
    __tablename__ = "documents"
    __table_args__ = {"schema": "documents"}

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(nullable=False, index=True)
    title: Mapped[str] = mapped_column(String(255), nullable=False)
    file_size_bytes: Mapped[int] = mapped_column(Integer, nullable=False)
    storage_path: Mapped[str] = mapped_column(String(500), nullable=False)
    department_id: Mapped[uuid.UUID | None] = mapped_column(nullable=True)
```

#### Example 1.5: The `ConversationSession` Entity Model
```python
class ConversationSession(Base):
    __tablename__ = "sessions"
    __table_args__ = {"schema": "conversations"}

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(nullable=False, index=True)
    user_id: Mapped[str] = mapped_column(String(100), nullable=False)
    title: Mapped[str] = mapped_column(String(200), default="New Chat")
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

---

## 2. Async Connection Pooling & Session Factory (`asyncpg`)

In asynchronous Python, SQLAlchemy communicates with PostgreSQL via the ultra-fast C-based **`asyncpg`** driver.

```mermaid
flowchart TD
    App["FastAPI Request Workers"] --> Pool["create_async_engine Connection Pool<br>(pool_size=20, max_overflow=10)"]
    Pool --> C1["PostgreSQL Connection 1"]
    Pool --> C2["PostgreSQL Connection 2"]
    Pool --> C3["PostgreSQL Connection 3"]
```

### 5 Real-World Normora Examples:

#### Example 2.1: Initializing the Async Engine with Connection Pool Tuning
```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

DATABASE_URL = "postgresql+asyncpg://normora_user:secure_pass@localhost:5432/normora_db"

engine = create_async_engine(
    DATABASE_URL,
    echo=False,  # Set True to view raw SQL queries in development logs
    pool_size=20,          # Keep 20 persistent connections open
    max_overflow=10,       # Allow up to 10 extra burst connections during high load
    pool_timeout=30.0,     # Wait max 30s for an available connection from pool
    pool_recycle=1800      # Recycle connections every 30 minutes to avoid stale sockets
)

# Factory that produces scoped AsyncSessions:
async_session_factory = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False  # Keep attributes accessible in memory after commit
)
```

#### Example 2.2: FastAPI Scoped Database Dependency
```python
from collections.abc import AsyncGenerator

async def get_async_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

#### Example 2.3: Querying a Single Record with `select()` and `scalar_one_or_none()`
Equivalent to EF Core's `FirstOrDefaultAsync()`:
```python
from sqlalchemy import select

async def get_tenant_by_slug(session: AsyncSession, slug: str) -> Tenant | None:
    # Generates: SELECT * FROM tenants.tenants WHERE slug = :slug LIMIT 1;
    stmt = select(Tenant).where(Tenant.slug == slug)
    result = await session.execute(stmt)
    return result.scalar_one_or_none()
```

#### Example 2.4: Querying a Filtered List with `scalars().all()`
Equivalent to EF Core's `Where(...).ToListAsync()`:
```python
async def get_active_tenant_documents(session: AsyncSession, tenant_id: uuid.UUID) -> list[Document]:
    stmt = select(Document).where(Document.tenant_id == tenant_id)
    result = await session.execute(stmt)
    return list(result.scalars().all())
```

#### Example 2.5: Inserting Records with `session.add()`
Equivalent to EF Core's `_context.Add(...)` and `SaveChangesAsync()`:
```python
async def create_new_tenant(session: AsyncSession, name: str, slug: str) -> Tenant:
    tenant = Tenant(name=name, slug=slug, plan="Pro")
    session.add(tenant)
    await session.flush()  # Flushes to DB to populate tenant.id without committing
    return tenant
```

---

## 3. Multi-Tenant Schema Search Path Isolation

In Normora, each module owns its own PostgreSQL schema (`tenants`, `documents`, `conversations`).  
Furthermore, we guarantee **tenant isolation** by ensuring that all queries against domain entities include an explicit tenant filter: `where(Entity.tenant_id == current_tenant_id)`.

### 5 Real-World Normora Examples:

#### Example 3.1: Enforcing Tenant Filter on Document Lookups (BOLA Defense)
```python
async def get_tenant_document_secure(
    session: AsyncSession,
    document_id: uuid.UUID,
    tenant_id: uuid.UUID
) -> Document | None:
    # Strictly scoped by both document ID AND tenant ID:
    stmt = (
        select(Document)
        .where(Document.id == document_id)
        .where(Document.tenant_id == tenant_id)
    )
    result = await session.execute(stmt)
    return result.scalar_one_or_none()
```

#### Example 3.2: Setting PostgreSQL `search_path` Dynamically
```python
from sqlalchemy import text

async def set_active_schema(session: AsyncSession, schema_name: str):
    await session.execute(text(f"SET search_path TO {schema_name}, public;"))
```

#### Example 3.3: Departmental Query Filter (RBAC Scoping)
```python
async def list_department_documents(
    session: AsyncSession,
    tenant_id: uuid.UUID,
    department_id: uuid.UUID
) -> list[Document]:
    stmt = (
        select(Document)
        .where(Document.tenant_id == tenant_id)
        .where(Document.department_id == department_id)
    )
    result = await session.execute(stmt)
    return list(result.scalars().all())
```

#### Example 3.4: Atomic Tenant Document Counter Update
```python
from sqlalchemy import update

async def increment_token_usage(session: AsyncSession, tenant_id: uuid.UUID, tokens: int):
    stmt = (
        update(Tenant)
        .where(Tenant.id == tenant_id)
        .values(active_seats=Tenant.active_seats + 1)
    )
    await session.execute(stmt)
```

#### Example 3.5: Deleting All Chunks for a Purged Document
```python
from sqlalchemy import delete

async def purge_document_chunks(session: AsyncSession, document_id: uuid.UUID, tenant_id: uuid.UUID):
    # Guarantees no cross-tenant accidental deletion:
    stmt = (
        delete(Document)
        .where(Document.id == document_id)
        .where(Document.tenant_id == tenant_id)
    )
    await session.execute(stmt)
```

---

## 4. Vector Storage with `pgvector` & HNSW Indexing

`pgvector` is a native PostgreSQL extension that adds vector support directly inside PostgreSQL.

### Why Google Gemini Uses 768 Dimensions:
* The Google Gemini `text-embedding-004` model generates dense embedding vectors of **exactly 768 floating-point numbers**.
* We define a `Vector(768)` column in our `DocumentChunk` table.
* We build an **HNSW (Hierarchical Navigable Small World)** index with cosine distance (`vector_cosine_ops`) for millisecond similarity lookups across millions of chunks!

```mermaid
flowchart TD
    Query["User Prompt: 'vacation policy'"] --> Gem["Gemini Embedding Model"]
    Gem --> Vec["Dense Vector (768 floats)"]
    Vec --> PG["PostgreSQL + pgvector Query<br>SELECT * FROM chunks ORDER BY embedding <=> :vec LIMIT 3"]
    PG --> Top["Top-3 Most Semantically Relevant Chunks!"]
```

### 5 Real-World Normora Examples:

#### Example 4.1: The `DocumentChunk` pgvector Model
```python
from pgvector.sqlalchemy import Vector
from sqlalchemy import Index

class DocumentChunk(Base):
    __tablename__ = "document_chunks"
    __table_args__ = (
        # HNSW Index for ultra-fast approximate nearest neighbor vector search:
        Index(
            "ix_document_chunks_embedding_hnsw",
            "embedding",
            postgresql_using="hnsw",
            postgresql_with={"m": 16, "ef_construction": 64},
            postgresql_ops={"embedding": "vector_cosine_ops"}
        ),
        {"schema": "documents"}
    )

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(nullable=False, index=True)
    document_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("documents.documents.id"), nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    page_number: Mapped[int] = mapped_column(Integer, nullable=False)
    # The 768-dimensional vector column for Google Gemini embeddings:
    embedding = mapped_column(Vector(768), nullable=False)
```

#### Example 4.2: Semantic Vector Similarity Query Using Cosine Distance (`<=>`)
In `pgvector`:
* `<=>` calculates **Cosine Distance** ($1 - \text{Cosine Similarity}$).
* A cosine distance of `0.0` means **100% identical semantics**.
* Order by distance ascending to get the most relevant chunks:
```python
async def search_similar_chunks(
    session: AsyncSession,
    tenant_id: uuid.UUID,
    query_vector: list[float],
    limit: int = 3
) -> list[tuple[DocumentChunk, float]]:
    # Distance expression:
    distance_expr = DocumentChunk.embedding.cosine_distance(query_vector).label("distance")

    stmt = (
        select(DocumentChunk, distance_expr)
        .where(DocumentChunk.tenant_id == tenant_id)
        .order_by(distance_expr)
        .limit(limit)
    )

    result = await session.execute(stmt)
    # Returns list of (chunk, distance) tuples:
    return [(row[0], 1.0 - row[1]) for row in result.all()]  # 1.0 - distance = similarity!
```

#### Example 4.3: Bulk Inserting Vector Chunks in a Batch
```python
async def insert_chunk_batch(
    session: AsyncSession,
    chunks_data: list[dict]
):
    chunk_entities = [DocumentChunk(**data) for data in chunks_data]
    session.add_all(chunk_entities)
    await session.flush()
    print(f"Batch inserted {len(chunk_entities)} vector chunks into pgvector.")
```

#### Example 4.4: Combining Metadata Filtering with Vector Distance
Filter strictly by department before running vector distance:
```python
async def search_departmental_chunks(
    session: AsyncSession,
    tenant_id: uuid.UUID,
    dept_id: uuid.UUID,
    query_vector: list[float]
):
    stmt = (
        select(DocumentChunk)
        .join(Document, DocumentChunk.document_id == Document.id)
        .where(DocumentChunk.tenant_id == tenant_id)
        .where(Document.department_id == dept_id)
        .order_by(DocumentChunk.embedding.cosine_distance(query_vector))
        .limit(5)
    )
    result = await session.execute(stmt)
    return list(result.scalars().all())
```

#### Example 4.5: Enabling the `vector` Extension in PostgreSQL
Before using `pgvector`, the extension must be created in your database:
```python
async def ensure_pgvector_extension(session: AsyncSession):
    await session.execute(text("CREATE EXTENSION IF NOT EXISTS vector;"))
    print("pgvector extension verified.")
```

---

## 5. Database Schema Migrations with Alembic

Just like `dotnet-ef migrations` in EF Core, **Alembic** is the official database migration tool for SQLAlchemy.

* `alembic init -t async migrations`: Initializes async Alembic configuration.
* `alembic revision --autogenerate -m "Add pgvector chunks"`: Inspects your SQLAlchemy models and generates the migration script.
* `alembic upgrade head`: Applies all pending migrations to the database.

### 5 Real-World Normora Examples:

#### Example 5.1: Async Alembic `env.py` Configuration
```python
# migrations/env.py
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context

# Import your models Base metadata:
target_metadata = Base.metadata

def run_migrations_offline():
    url = config.get_main_option("sqlalchemy.url")
    context.configure(url=url, target_metadata=target_metadata, literal_binds=True)
    with context.begin_transaction():
        context.run_migrations()
```

#### Example 5.2: Migration File Adding `pgvector` Extension and Schema
```python
# migrations/versions/001_initial_schemas.py
from alembic import op

def upgrade():
    # 1. Enable pgvector
    op.execute("CREATE EXTENSION IF NOT EXISTS vector;")
    # 2. Create isolated schemas
    op.execute("CREATE SCHEMA IF NOT EXISTS tenants;")
    op.execute("CREATE SCHEMA IF NOT EXISTS documents;")
    op.execute("CREATE SCHEMA IF NOT EXISTS conversations;")

def downgrade():
    op.execute("DROP SCHEMA IF EXISTS conversations CASCADE;")
    op.execute("DROP SCHEMA IF EXISTS documents CASCADE;")
    op.execute("DROP SCHEMA IF EXISTS tenants CASCADE;")
```

#### Example 5.3: Automated Migration CLI Command
```bash
# Generate migration from models:
alembic revision --autogenerate -m "Create tenants and document_chunks tables"

# Apply migration:
alembic upgrade head
```

#### Example 5.4: Checking for Pending Migrations at Application Startup
Fail-fast pattern: Ensure the database matches the code before opening the web port:
```python
async def verify_database_schema(session: AsyncSession):
    # Verify tables exist:
    res = await session.execute(text("SELECT to_regclass('documents.document_chunks');"))
    if res.scalar() is None:
        raise RuntimeError("CRITICAL: Database schema is out of sync. Please run 'alembic upgrade head'.")
```

#### Example 5.5: Rollback Command
```bash
# Rollback the last migration:
alembic downgrade -1
```

---

## Summary & Next Step

You have mastered the entire persistence and vector engine:
1. Writing asynchronous SQLAlchemy 2.0 declarative models (`Mapped[]`, `mapped_column()`).
2. Configuring production connection pooling with `asyncpg`.
3. Isolating multi-tenant data by schema and query filters.
4. Persisting 768-dimensional vector embeddings with `pgvector` and HNSW indexing.
5. Managing database versions with Alembic.

Now, we enter **Part IV: Real-Time Protocols & AI Application Systems**!

In **Module 16**, we master **Streaming Architectures, Server-Sent Events (SSE) & WebSockets**!
