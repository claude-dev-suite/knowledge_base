# SQLAlchemy ORM Guide

Comprehensive guide to SQLAlchemy 2.0 ORM patterns, relationships, and async support.

**Official Documentation:** https://docs.sqlalchemy.org/en/20/

---

## Table of Contents

1. [Engine and Session](#engine-and-session)
2. [Model Definition](#model-definition)
3. [Relationships](#relationships)
4. [Querying](#querying)
5. [Async SQLAlchemy](#async-sqlalchemy)
6. [Migrations with Alembic](#migrations-with-alembic)
7. [Advanced Patterns](#advanced-patterns)
8. [Testing](#testing)
9. [Performance](#performance)

---

## Engine and Session

### Creating Engine

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

# Sync engine
engine = create_engine(
    "postgresql://user:password@localhost/dbname",
    pool_size=5,
    max_overflow=10,
    pool_timeout=30,
    pool_recycle=1800,
    echo=False,  # Log SQL statements
)

# With connection arguments
engine = create_engine(
    "postgresql://user:password@localhost/dbname",
    connect_args={
        "connect_timeout": 10,
        "application_name": "myapp",
    },
)

# SQLite
engine = create_engine("sqlite:///./app.db", echo=True)

# In-memory SQLite (for testing)
engine = create_engine("sqlite:///:memory:")
```

### Session Configuration

```python
from sqlalchemy.orm import Session, sessionmaker

# Session factory
SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,
    autoflush=False,
    expire_on_commit=False,
)

# Using session
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


# Context manager pattern
from contextlib import contextmanager

@contextmanager
def get_session():
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()


# Usage
with get_session() as session:
    user = session.get(User, 1)
```

### Scoped Session

```python
from sqlalchemy.orm import scoped_session

# Thread-local session
db_session = scoped_session(SessionLocal)

# Usage in web app
@app.teardown_appcontext
def shutdown_session(exception=None):
    db_session.remove()
```

---

## Model Definition

### Base Model

```python
from datetime import datetime
from typing import Optional
from sqlalchemy import String, DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    """Base class for all models"""
    pass


class TimestampMixin:
    """Mixin for created_at and updated_at columns"""
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )


class User(TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(100))
    is_active: Mapped[bool] = mapped_column(default=True)
    bio: Mapped[Optional[str]] = mapped_column(String(500), nullable=True)

    def __repr__(self) -> str:
        return f"<User(id={self.id}, email={self.email})>"
```

### Column Types

```python
from sqlalchemy import (
    String, Integer, Float, Boolean, DateTime, Date, Time,
    Text, LargeBinary, JSON, ARRAY, Enum, Numeric, UUID
)
from sqlalchemy.dialects.postgresql import JSONB
import uuid
import enum


class Status(enum.Enum):
    PENDING = "pending"
    ACTIVE = "active"
    INACTIVE = "inactive"


class Product(Base):
    __tablename__ = "products"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
    )
    name: Mapped[str] = mapped_column(String(200))
    description: Mapped[Optional[str]] = mapped_column(Text)
    price: Mapped[float] = mapped_column(Numeric(10, 2))
    quantity: Mapped[int] = mapped_column(Integer, default=0)
    is_available: Mapped[bool] = mapped_column(Boolean, default=True)
    status: Mapped[Status] = mapped_column(Enum(Status), default=Status.PENDING)
    metadata_: Mapped[Optional[dict]] = mapped_column("metadata", JSONB)
    tags: Mapped[Optional[list[str]]] = mapped_column(ARRAY(String))
    image: Mapped[Optional[bytes]] = mapped_column(LargeBinary)
```

### Table Arguments

```python
class Article(Base):
    __tablename__ = "articles"
    __table_args__ = (
        # Indexes
        Index("ix_articles_slug", "slug"),
        Index("ix_articles_created", "created_at"),
        # Composite unique constraint
        UniqueConstraint("author_id", "slug", name="uq_author_slug"),
        # Check constraint
        CheckConstraint("views >= 0", name="ck_positive_views"),
        # Table options
        {"schema": "blog"},
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    slug: Mapped[str] = mapped_column(String(200))
    views: Mapped[int] = mapped_column(default=0)
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
```

---

## Relationships

### One-to-Many

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship, Mapped


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    # One-to-Many: User has many posts
    posts: Mapped[list["Post"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",
        lazy="selectin",  # Eager loading strategy
    )


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    # Many-to-One: Post belongs to User
    author: Mapped["User"] = relationship(back_populates="posts")
```

### Many-to-Many

```python
from sqlalchemy import Table, Column, ForeignKey

# Association table
post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", ForeignKey("tags.id"), primary_key=True),
)


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))

    tags: Mapped[list["Tag"]] = relationship(
        secondary=post_tags,
        back_populates="posts",
        lazy="selectin",
    )


class Tag(Base):
    __tablename__ = "tags"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    posts: Mapped[list["Post"]] = relationship(
        secondary=post_tags,
        back_populates="tags",
    )
```

### Many-to-Many with Association Object

```python
from datetime import datetime


class UserRole(Base):
    """Association object with extra data"""
    __tablename__ = "user_roles"

    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), primary_key=True)
    role_id: Mapped[int] = mapped_column(ForeignKey("roles.id"), primary_key=True)
    granted_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    granted_by: Mapped[Optional[int]] = mapped_column(ForeignKey("users.id"))

    user: Mapped["User"] = relationship(back_populates="user_roles", foreign_keys=[user_id])
    role: Mapped["Role"] = relationship(back_populates="user_roles")


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    user_roles: Mapped[list["UserRole"]] = relationship(
        back_populates="user",
        foreign_keys="UserRole.user_id",
    )

    @property
    def roles(self) -> list["Role"]:
        return [ur.role for ur in self.user_roles]


class Role(Base):
    __tablename__ = "roles"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    user_roles: Mapped[list["UserRole"]] = relationship(back_populates="role")
```

### Self-Referential Relationship

```python
class Employee(Base):
    __tablename__ = "employees"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    manager_id: Mapped[Optional[int]] = mapped_column(ForeignKey("employees.id"))

    # Self-referential
    manager: Mapped[Optional["Employee"]] = relationship(
        back_populates="subordinates",
        remote_side=[id],
    )
    subordinates: Mapped[list["Employee"]] = relationship(
        back_populates="manager",
    )


class Category(Base):
    """Adjacency list pattern for tree structure"""
    __tablename__ = "categories"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    parent_id: Mapped[Optional[int]] = mapped_column(ForeignKey("categories.id"))

    parent: Mapped[Optional["Category"]] = relationship(
        back_populates="children",
        remote_side=[id],
    )
    children: Mapped[list["Category"]] = relationship(back_populates="parent")
```

---

## Querying

### Basic Queries

```python
from sqlalchemy import select, and_, or_, not_
from sqlalchemy.orm import Session


def get_user_by_id(session: Session, user_id: int) -> User | None:
    return session.get(User, user_id)


def get_user_by_email(session: Session, email: str) -> User | None:
    stmt = select(User).where(User.email == email)
    return session.execute(stmt).scalar_one_or_none()


def get_active_users(session: Session) -> list[User]:
    stmt = select(User).where(User.is_active == True)
    return session.execute(stmt).scalars().all()


def get_users_with_filters(
    session: Session,
    name_contains: str | None = None,
    is_active: bool | None = None,
) -> list[User]:
    stmt = select(User)

    if name_contains:
        stmt = stmt.where(User.name.ilike(f"%{name_contains}%"))

    if is_active is not None:
        stmt = stmt.where(User.is_active == is_active)

    return session.execute(stmt).scalars().all()
```

### Filtering

```python
from sqlalchemy import select, and_, or_, not_, between, in_


# Multiple conditions
stmt = select(User).where(
    and_(
        User.is_active == True,
        User.created_at >= start_date,
    )
)

# OR conditions
stmt = select(User).where(
    or_(
        User.role == "admin",
        User.role == "moderator",
    )
)

# IN clause
stmt = select(User).where(User.id.in_([1, 2, 3]))

# NOT IN
stmt = select(User).where(User.status.not_in(["banned", "deleted"]))

# BETWEEN
stmt = select(Product).where(between(Product.price, 10, 100))

# LIKE / ILIKE
stmt = select(User).where(User.name.ilike("%john%"))

# IS NULL / IS NOT NULL
stmt = select(User).where(User.deleted_at.is_(None))
stmt = select(User).where(User.verified_at.is_not(None))

# Comparing columns
stmt = select(Product).where(Product.quantity < Product.min_quantity)
```

### Ordering and Pagination

```python
from sqlalchemy import desc, asc, nulls_last


# Order by
stmt = select(User).order_by(User.created_at.desc())

# Multiple order columns
stmt = select(User).order_by(
    desc(User.is_active),
    asc(User.name),
)

# Handle nulls
stmt = select(User).order_by(User.last_login.desc().nulls_last())

# Pagination
def get_users_paginated(
    session: Session,
    page: int = 1,
    page_size: int = 20,
) -> list[User]:
    stmt = (
        select(User)
        .order_by(User.id)
        .offset((page - 1) * page_size)
        .limit(page_size)
    )
    return session.execute(stmt).scalars().all()
```

### Aggregation

```python
from sqlalchemy import func, distinct


# Count
stmt = select(func.count(User.id))
total = session.execute(stmt).scalar()

# Count with filter
stmt = select(func.count(User.id)).where(User.is_active == True)

# Distinct count
stmt = select(func.count(distinct(User.email)))

# Sum, Avg, Min, Max
stmt = select(
    func.sum(Order.total),
    func.avg(Order.total),
    func.min(Order.total),
    func.max(Order.total),
).where(Order.user_id == user_id)

result = session.execute(stmt).one()
total_sum, average, minimum, maximum = result

# Group by
stmt = (
    select(
        User.status,
        func.count(User.id).label("count"),
    )
    .group_by(User.status)
)

results = session.execute(stmt).all()
for status, count in results:
    print(f"{status}: {count}")

# Having
stmt = (
    select(
        Product.category_id,
        func.count(Product.id).label("product_count"),
    )
    .group_by(Product.category_id)
    .having(func.count(Product.id) > 5)
)
```

### Joins

```python
from sqlalchemy import join, outerjoin
from sqlalchemy.orm import joinedload, selectinload, contains_eager


# Implicit join (relationship)
stmt = (
    select(User)
    .join(User.posts)
    .where(Post.published == True)
)

# Explicit join
stmt = (
    select(User, Post)
    .join(Post, User.id == Post.author_id)
)

# Left outer join
stmt = (
    select(User, Post)
    .outerjoin(Post, User.id == Post.author_id)
)

# Multiple joins
stmt = (
    select(User)
    .join(User.posts)
    .join(Post.tags)
    .where(Tag.name == "python")
)
```

### Eager Loading

```python
from sqlalchemy.orm import joinedload, selectinload, subqueryload


# Joined load (single query with JOIN)
stmt = (
    select(User)
    .options(joinedload(User.profile))
    .where(User.id == user_id)
)

# Select-in load (separate SELECT with IN clause)
stmt = (
    select(User)
    .options(selectinload(User.posts))
)

# Nested eager loading
stmt = (
    select(User)
    .options(
        selectinload(User.posts)
        .selectinload(Post.comments)
    )
)

# Combined strategies
stmt = (
    select(User)
    .options(
        joinedload(User.profile),
        selectinload(User.posts).joinedload(Post.author),
    )
)

# Load only specific columns
stmt = (
    select(User)
    .options(
        selectinload(User.posts).load_only(Post.id, Post.title)
    )
)
```

---

## Async SQLAlchemy

### Async Engine and Session

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    async_sessionmaker,
    AsyncSession,
)

# Async engine
async_engine = create_async_engine(
    "postgresql+asyncpg://user:password@localhost/dbname",
    pool_size=5,
    max_overflow=10,
    echo=False,
)

# Async session factory
AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


# Dependency injection pattern
async def get_async_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Async Queries

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession


async def get_user(session: AsyncSession, user_id: int) -> User | None:
    result = await session.get(User, user_id)
    return result


async def get_all_users(session: AsyncSession) -> list[User]:
    stmt = select(User).order_by(User.name)
    result = await session.execute(stmt)
    return result.scalars().all()


async def create_user(session: AsyncSession, data: dict) -> User:
    user = User(**data)
    session.add(user)
    await session.commit()
    await session.refresh(user)
    return user


async def update_user(
    session: AsyncSession,
    user_id: int,
    data: dict,
) -> User | None:
    user = await session.get(User, user_id)
    if user:
        for key, value in data.items():
            setattr(user, key, value)
        await session.commit()
        await session.refresh(user)
    return user


async def delete_user(session: AsyncSession, user_id: int) -> bool:
    user = await session.get(User, user_id)
    if user:
        await session.delete(user)
        await session.commit()
        return True
    return False
```

### Async Relationships (Lazy Loading Issue)

```python
from sqlalchemy.orm import selectinload


# BAD: Lazy loading doesn't work in async
async def get_user_posts_wrong(session: AsyncSession, user_id: int):
    user = await session.get(User, user_id)
    # This will fail - lazy loading in async context
    return user.posts  # Error!


# GOOD: Use eager loading
async def get_user_posts(session: AsyncSession, user_id: int):
    stmt = (
        select(User)
        .options(selectinload(User.posts))
        .where(User.id == user_id)
    )
    result = await session.execute(stmt)
    user = result.scalar_one_or_none()
    return user.posts if user else []


# Or use run_sync for lazy loading
async def get_user_posts_sync(session: AsyncSession, user_id: int):
    user = await session.get(User, user_id)
    if user:
        # Run lazy load in sync context
        posts = await session.run_sync(lambda _: user.posts)
        return posts
    return []
```

### Async with FastAPI

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()


async def get_db():
    async with AsyncSessionLocal() as session:
        yield session


@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
):
    stmt = select(User).where(User.id == user_id)
    result = await db.execute(stmt)
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return user


@app.post("/users")
async def create_user(
    data: UserCreate,
    db: AsyncSession = Depends(get_db),
):
    user = User(**data.model_dump())
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user
```

---

## Migrations with Alembic

### Setup

```bash
# Install
pip install alembic

# Initialize
alembic init alembic
```

### Configuration

```python
# alembic/env.py
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
from myapp.models import Base
from myapp.config import DATABASE_URL

config = context.config
config.set_main_option("sqlalchemy.url", DATABASE_URL)

target_metadata = Base.metadata


def run_migrations_online():
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
            compare_server_default=True,
        )

        with context.begin_transaction():
            context.run_migrations()
```

### Commands

```bash
# Generate migration
alembic revision --autogenerate -m "Add user table"

# Run migrations
alembic upgrade head

# Downgrade
alembic downgrade -1

# Show current revision
alembic current

# Show history
alembic history
```

### Migration Script

```python
# alembic/versions/xxx_add_user_table.py
"""Add user table

Revision ID: abc123
Create Date: 2024-01-01 12:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

revision = 'abc123'
down_revision = None
branch_labels = None
depends_on = None


def upgrade():
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('email', sa.String(255), nullable=False, unique=True),
        sa.Column('name', sa.String(100), nullable=False),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.func.now()),
    )
    op.create_index('ix_users_email', 'users', ['email'])


def downgrade():
    op.drop_index('ix_users_email')
    op.drop_table('users')
```

---

## Advanced Patterns

### Repository Pattern

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar
from sqlalchemy import select
from sqlalchemy.orm import Session

T = TypeVar("T", bound=Base)


class AbstractRepository(ABC, Generic[T]):
    @abstractmethod
    def get(self, id: int) -> T | None:
        pass

    @abstractmethod
    def get_all(self) -> list[T]:
        pass

    @abstractmethod
    def add(self, entity: T) -> T:
        pass

    @abstractmethod
    def delete(self, entity: T) -> None:
        pass


class SQLAlchemyRepository(AbstractRepository[T]):
    def __init__(self, session: Session, model: type[T]):
        self.session = session
        self.model = model

    def get(self, id: int) -> T | None:
        return self.session.get(self.model, id)

    def get_all(self) -> list[T]:
        stmt = select(self.model)
        return self.session.execute(stmt).scalars().all()

    def add(self, entity: T) -> T:
        self.session.add(entity)
        self.session.flush()
        return entity

    def delete(self, entity: T) -> None:
        self.session.delete(entity)


class UserRepository(SQLAlchemyRepository[User]):
    def __init__(self, session: Session):
        super().__init__(session, User)

    def get_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        return self.session.execute(stmt).scalar_one_or_none()

    def get_active_users(self) -> list[User]:
        stmt = select(User).where(User.is_active == True)
        return self.session.execute(stmt).scalars().all()
```

### Unit of Work Pattern

```python
from contextlib import contextmanager


class UnitOfWork:
    def __init__(self, session_factory):
        self.session_factory = session_factory

    def __enter__(self):
        self.session = self.session_factory()
        self.users = UserRepository(self.session)
        self.posts = PostRepository(self.session)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.rollback()
        self.session.close()

    def commit(self):
        self.session.commit()

    def rollback(self):
        self.session.rollback()


# Usage
def create_user_with_post(uow: UnitOfWork, user_data: dict, post_data: dict):
    with uow:
        user = User(**user_data)
        uow.users.add(user)

        post = Post(**post_data, author=user)
        uow.posts.add(post)

        uow.commit()
        return user
```

### Soft Delete

```python
from datetime import datetime
from sqlalchemy import event


class SoftDeleteMixin:
    deleted_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
        default=None,
    )

    def soft_delete(self):
        self.deleted_at = datetime.utcnow()

    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None


class User(SoftDeleteMixin, Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))


# Query helper
def not_deleted(query):
    return query.where(User.deleted_at.is_(None))


# Or use event listener
@event.listens_for(Session, "do_orm_execute")
def soft_delete_filter(execute_state):
    if execute_state.is_select:
        execute_state.statement = execute_state.statement.where(
            User.deleted_at.is_(None)
        )
```

### Hybrid Properties

```python
from sqlalchemy.ext.hybrid import hybrid_property


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    first_name: Mapped[str] = mapped_column(String(50))
    last_name: Mapped[str] = mapped_column(String(50))

    @hybrid_property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"

    @full_name.expression
    def full_name(cls):
        return cls.first_name + " " + cls.last_name


# Can query on hybrid property
stmt = select(User).where(User.full_name == "John Doe")
```

---

## Testing

### Test Fixtures

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker


@pytest.fixture(scope="session")
def engine():
    return create_engine("sqlite:///:memory:")


@pytest.fixture(scope="session")
def tables(engine):
    Base.metadata.create_all(engine)
    yield
    Base.metadata.drop_all(engine)


@pytest.fixture
def db_session(engine, tables):
    connection = engine.connect()
    transaction = connection.begin()
    Session = sessionmaker(bind=connection)
    session = Session()

    yield session

    session.close()
    transaction.rollback()
    connection.close()


def test_create_user(db_session):
    user = User(name="Test", email="test@example.com")
    db_session.add(user)
    db_session.commit()

    assert db_session.query(User).count() == 1
```

### Async Test Fixtures

```python
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession


@pytest.fixture(scope="session")
def event_loop():
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(scope="session")
async def async_engine():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest.fixture
async def async_session(async_engine):
    async with AsyncSession(async_engine) as session:
        yield session
        await session.rollback()


@pytest.mark.asyncio
async def test_async_create_user(async_session):
    user = User(name="Test", email="test@example.com")
    async_session.add(user)
    await async_session.commit()

    stmt = select(User)
    result = await async_session.execute(stmt)
    assert len(result.scalars().all()) == 1
```

---

## Performance

### Query Optimization

```python
# Use load_only for partial loading
stmt = select(User).options(load_only(User.id, User.email))

# Avoid N+1 with eager loading
stmt = select(User).options(selectinload(User.posts))

# Use batch operations
from sqlalchemy import update, delete

# Bulk update
stmt = (
    update(User)
    .where(User.is_active == False)
    .values(status="inactive")
)
session.execute(stmt)

# Bulk insert
session.bulk_insert_mappings(User, [
    {"name": "User 1", "email": "user1@example.com"},
    {"name": "User 2", "email": "user2@example.com"},
])

# Or with ORM objects
session.bulk_save_objects([
    User(name="User 1", email="user1@example.com"),
    User(name="User 2", email="user2@example.com"),
])
```

### Connection Pooling

```python
engine = create_engine(
    DATABASE_URL,
    pool_size=5,           # Maintain 5 connections
    max_overflow=10,       # Allow up to 15 total
    pool_timeout=30,       # Wait 30s for connection
    pool_recycle=1800,     # Recycle connections after 30min
    pool_pre_ping=True,    # Verify connection is alive
)
```

---

## Quick Reference

### Common Operations

| Operation | Code |
|-----------|------|
| Get by ID | `session.get(Model, id)` |
| Get one | `session.execute(stmt).scalar_one()` |
| Get one or none | `session.execute(stmt).scalar_one_or_none()` |
| Get all | `session.execute(stmt).scalars().all()` |
| Add | `session.add(obj)` |
| Delete | `session.delete(obj)` |
| Commit | `session.commit()` |
| Rollback | `session.rollback()` |

### Loading Strategies

| Strategy | Use Case |
|----------|----------|
| `joinedload` | One-to-one, many-to-one |
| `selectinload` | One-to-many, many-to-many |
| `subqueryload` | Large collections |
| `lazy="raise"` | Prevent N+1 (raise error) |
