# FastAPI Database Integration - Comprehensive Guide

This guide covers database integration with FastAPI, including SQLAlchemy ORM, async operations,
MongoDB, migrations, and best practices for production applications.

---

## Table of Contents

1. [SQLAlchemy Setup with FastAPI](#1-sqlalchemy-setup-with-fastapi)
2. [Database Connection and Session Management](#2-database-connection-and-session-management)
3. [Model Definition (SQLAlchemy ORM)](#3-model-definition-sqlalchemy-orm)
4. [Pydantic Schemas (Create, Read, Update)](#4-pydantic-schemas-create-read-update)
5. [CRUD Operations](#5-crud-operations)
6. [Dependency Injection for DB Sessions](#6-dependency-injection-for-db-sessions)
7. [Async Database Operations](#7-async-database-operations)
8. [Relationships (One-to-Many, Many-to-Many)](#8-relationships-one-to-many-many-to-many)
9. [Query Optimization](#9-query-optimization)
10. [Database Migrations with Alembic](#10-database-migrations-with-alembic)
11. [Transaction Management](#11-transaction-management)
12. [PostgreSQL with SQLAlchemy](#12-postgresql-with-sqlalchemy)
13. [MongoDB with Motor](#13-mongodb-with-motor)
14. [Testing with Test Database](#14-testing-with-test-database)
15. [Connection Pooling](#15-connection-pooling)
16. [Best Practices](#16-best-practices)

---

## 1. SQLAlchemy Setup with FastAPI

### Installation

```bash
# Core dependencies
pip install sqlalchemy[asyncio]
pip install asyncpg          # PostgreSQL async driver
pip install psycopg2-binary  # PostgreSQL sync driver (optional)
pip install alembic          # Database migrations

# For MySQL
pip install aiomysql

# For SQLite async
pip install aiosqlite
```

### Project Structure

```
project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── database.py          # Database configuration
│   ├── models/
│   │   ├── __init__.py
│   │   ├── base.py          # Base model class
│   │   ├── user.py
│   │   └── post.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── post.py
│   ├── crud/
│   │   ├── __init__.py
│   │   ├── base.py          # Generic CRUD class
│   │   ├── user.py
│   │   └── post.py
│   └── api/
│       ├── __init__.py
│       ├── deps.py          # Dependencies
│       └── routes/
│           ├── users.py
│           └── posts.py
├── alembic/
│   ├── versions/
│   └── env.py
├── alembic.ini
└── requirements.txt
```

### Synchronous Setup

```python
# database.py - Synchronous configuration
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

DATABASE_URL = "postgresql://user:password@localhost:5432/dbname"

# Create engine with connection pool settings
engine = create_engine(
    DATABASE_URL,
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,  # Enable connection health checks
    echo=False,          # Set True for SQL logging
)

# Session factory
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine,
)

# Base class for models
Base = declarative_base()

# Dependency for getting database session
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Asynchronous Setup

```python
# database.py - Async configuration
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import declarative_base

DATABASE_URL = "postgresql+asyncpg://user:password@localhost:5432/dbname"

# Create async engine
engine = create_async_engine(
    DATABASE_URL,
    echo=True,           # SQL logging
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,
)

# Async session factory (SQLAlchemy 2.0 style)
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autocommit=False,
    autoflush=False,
)

Base = declarative_base()

# Async dependency
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

---

## 2. Database Connection and Session Management

### Engine Configuration Options

```python
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    DATABASE_URL,
    # Logging
    echo=True,                    # Log all SQL statements
    echo_pool=True,               # Log connection pool events

    # Connection pool settings
    pool_size=5,                  # Number of permanent connections
    max_overflow=10,              # Additional connections when pool is full
    pool_timeout=30,              # Seconds to wait for available connection
    pool_recycle=1800,            # Recycle connections after 30 minutes
    pool_pre_ping=True,           # Test connections before use

    # Execution options
    execution_options={
        "isolation_level": "REPEATABLE READ"
    },
)
```

### Session Configuration

```python
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

# Synchronous session
SyncSessionLocal = sessionmaker(
    bind=engine,
    class_=Session,
    autocommit=False,           # Require explicit commits
    autoflush=False,            # Don't auto-flush before queries
    expire_on_commit=True,      # Expire objects after commit (default)
)

# Asynchronous session
AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    autocommit=False,
    autoflush=False,
    expire_on_commit=False,     # Keep objects usable after commit
)
```

### Context Manager Pattern

```python
# Sync context manager
from contextlib import contextmanager

@contextmanager
def get_db_session():
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Async context manager
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_async_db_session():
    session = AsyncSessionLocal()
    try:
        yield session
        await session.commit()
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()

# Usage
async def example_usage():
    async with get_async_db_session() as db:
        user = await db.get(User, 1)
        user.name = "Updated Name"
        # Auto-commits on successful exit
```

### Lifespan Events for Connection Management

```python
# main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from database import engine, Base

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Create tables (development only)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield

    # Shutdown: Dispose engine connections
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

---

## 3. Model Definition (SQLAlchemy ORM)

### Base Model with Common Fields

```python
# models/base.py
from datetime import datetime
from sqlalchemy import Column, Integer, DateTime
from sqlalchemy.sql import func
from sqlalchemy.orm import declarative_base, declared_attr

Base = declarative_base()

class TimestampMixin:
    """Mixin that adds created_at and updated_at columns."""

    created_at = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    updated_at = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )

class TableNameMixin:
    """Mixin that automatically generates table name."""

    @declared_attr
    def __tablename__(cls) -> str:
        # Convert CamelCase to snake_case
        import re
        name = re.sub('(.)([A-Z][a-z]+)', r'\1_\2', cls.__name__)
        return re.sub('([a-z0-9])([A-Z])', r'\1_\2', name).lower() + 's'

class BaseModel(Base, TimestampMixin, TableNameMixin):
    """Abstract base model with id, timestamps, and auto table name."""

    __abstract__ = True

    id = Column(Integer, primary_key=True, index=True, autoincrement=True)

    def __repr__(self):
        return f"<{self.__class__.__name__}(id={self.id})>"
```

### User Model with Full Features

```python
# models/user.py
from sqlalchemy import Column, Integer, String, Boolean, Text, Enum as SQLEnum
from sqlalchemy.orm import relationship, validates
from enum import Enum
import re

from .base import BaseModel

class UserRole(str, Enum):
    ADMIN = "admin"
    MODERATOR = "moderator"
    USER = "user"

class UserStatus(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    BANNED = "banned"

class User(BaseModel):
    __tablename__ = "users"

    # Authentication fields
    email = Column(String(255), unique=True, index=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)

    # Profile fields
    username = Column(String(50), unique=True, index=True, nullable=False)
    full_name = Column(String(100), nullable=True)
    bio = Column(Text, nullable=True)
    avatar_url = Column(String(500), nullable=True)

    # Status and role
    role = Column(
        SQLEnum(UserRole),
        default=UserRole.USER,
        nullable=False,
    )
    status = Column(
        SQLEnum(UserStatus),
        default=UserStatus.ACTIVE,
        nullable=False,
    )

    # Flags
    is_verified = Column(Boolean, default=False, nullable=False)
    is_superuser = Column(Boolean, default=False, nullable=False)

    # Relationships
    posts = relationship(
        "Post",
        back_populates="author",
        cascade="all, delete-orphan",
        lazy="selectin",
    )
    comments = relationship(
        "Comment",
        back_populates="author",
        cascade="all, delete-orphan",
    )

    # Validation
    @validates('email')
    def validate_email(self, key, email):
        if not re.match(r"[^@]+@[^@]+\.[^@]+", email):
            raise ValueError("Invalid email address")
        return email.lower()

    @validates('username')
    def validate_username(self, key, username):
        if not re.match(r"^[a-zA-Z0-9_]{3,50}$", username):
            raise ValueError("Username must be 3-50 alphanumeric characters")
        return username.lower()

    # Properties
    @property
    def is_admin(self) -> bool:
        return self.role == UserRole.ADMIN or self.is_superuser

    # Table configuration
    __table_args__ = (
        # Composite index for common queries
        # Index('ix_users_role_status', 'role', 'status'),
    )
```

### Post Model with Foreign Keys

```python
# models/post.py
from sqlalchemy import Column, Integer, String, Text, ForeignKey, Boolean
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import ARRAY

from .base import BaseModel

class Post(BaseModel):
    __tablename__ = "posts"

    # Content
    title = Column(String(200), nullable=False)
    slug = Column(String(250), unique=True, index=True, nullable=False)
    content = Column(Text, nullable=True)
    excerpt = Column(String(500), nullable=True)

    # Metadata
    is_published = Column(Boolean, default=False, nullable=False)
    view_count = Column(Integer, default=0, nullable=False)

    # PostgreSQL-specific: Array type
    tags = Column(ARRAY(String(50)), default=list, nullable=False)

    # Foreign key
    author_id = Column(
        Integer,
        ForeignKey("users.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )

    # Relationships
    author = relationship("User", back_populates="posts")
    comments = relationship(
        "Comment",
        back_populates="post",
        cascade="all, delete-orphan",
        order_by="Comment.created_at.desc()",
    )

    def __repr__(self):
        return f"<Post(id={self.id}, title='{self.title[:30]}...')>"
```

### Comment Model (Nested/Self-Referential)

```python
# models/comment.py
from sqlalchemy import Column, Integer, Text, ForeignKey
from sqlalchemy.orm import relationship

from .base import BaseModel

class Comment(BaseModel):
    __tablename__ = "comments"

    content = Column(Text, nullable=False)

    # Foreign keys
    author_id = Column(
        Integer,
        ForeignKey("users.id", ondelete="CASCADE"),
        nullable=False,
    )
    post_id = Column(
        Integer,
        ForeignKey("posts.id", ondelete="CASCADE"),
        nullable=False,
    )
    # Self-referential for nested comments
    parent_id = Column(
        Integer,
        ForeignKey("comments.id", ondelete="CASCADE"),
        nullable=True,
    )

    # Relationships
    author = relationship("User", back_populates="comments")
    post = relationship("Post", back_populates="comments")
    parent = relationship(
        "Comment",
        remote_side="Comment.id",
        backref="replies",
    )
```

---

## 4. Pydantic Schemas (Create, Read, Update)

### Base Schema Configuration

```python
# schemas/base.py
from pydantic import BaseModel, ConfigDict
from datetime import datetime
from typing import Optional

class BaseSchema(BaseModel):
    """Base schema with common configuration."""
    model_config = ConfigDict(
        from_attributes=True,      # Enable ORM mode
        populate_by_name=True,     # Allow alias population
        str_strip_whitespace=True, # Strip whitespace from strings
        validate_assignment=True,  # Validate on attribute assignment
    )

class TimestampSchema(BaseSchema):
    """Schema with timestamp fields."""
    created_at: datetime
    updated_at: datetime
```

### User Schemas

```python
# schemas/user.py
from pydantic import BaseModel, EmailStr, Field, field_validator, ConfigDict
from typing import Optional, List
from datetime import datetime
from enum import Enum
import re

class UserRole(str, Enum):
    ADMIN = "admin"
    MODERATOR = "moderator"
    USER = "user"

class UserStatus(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    BANNED = "banned"

# Base schema - shared fields
class UserBase(BaseModel):
    email: EmailStr
    username: str = Field(..., min_length=3, max_length=50)
    full_name: Optional[str] = Field(None, max_length=100)
    bio: Optional[str] = Field(None, max_length=500)

    @field_validator('username')
    @classmethod
    def validate_username(cls, v: str) -> str:
        if not re.match(r"^[a-zA-Z0-9_]+$", v):
            raise ValueError('Username must be alphanumeric with underscores only')
        return v.lower()

# Create schema - for creating new users
class UserCreate(UserBase):
    password: str = Field(..., min_length=8, max_length=100)

    @field_validator('password')
    @classmethod
    def validate_password(cls, v: str) -> str:
        if not re.search(r"[A-Z]", v):
            raise ValueError('Password must contain at least one uppercase letter')
        if not re.search(r"[a-z]", v):
            raise ValueError('Password must contain at least one lowercase letter')
        if not re.search(r"\d", v):
            raise ValueError('Password must contain at least one digit')
        return v

# Update schema - all fields optional
class UserUpdate(BaseModel):
    email: Optional[EmailStr] = None
    username: Optional[str] = Field(None, min_length=3, max_length=50)
    full_name: Optional[str] = Field(None, max_length=100)
    bio: Optional[str] = Field(None, max_length=500)
    avatar_url: Optional[str] = None

    model_config = ConfigDict(extra='forbid')  # Forbid extra fields

# Partial update schema - using exclude_unset
class UserPatch(BaseModel):
    full_name: Optional[str] = None
    bio: Optional[str] = None
    avatar_url: Optional[str] = None

# Response schema - what's returned from API
class UserResponse(UserBase):
    id: int
    role: UserRole
    status: UserStatus
    is_verified: bool
    created_at: datetime
    updated_at: datetime

    model_config = ConfigDict(from_attributes=True)

# Response with nested relationships
class UserWithPosts(UserResponse):
    posts: List["PostResponse"] = []

# Internal schema - includes sensitive data
class UserInDB(UserResponse):
    hashed_password: str

# List response with pagination
class UserListResponse(BaseModel):
    items: List[UserResponse]
    total: int
    page: int
    per_page: int
    pages: int
```

### Post Schemas

```python
# schemas/post.py
from pydantic import BaseModel, Field, field_validator, ConfigDict
from typing import Optional, List
from datetime import datetime
import re

class PostBase(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    content: Optional[str] = None
    excerpt: Optional[str] = Field(None, max_length=500)
    tags: List[str] = Field(default_factory=list)

    @field_validator('tags')
    @classmethod
    def validate_tags(cls, v: List[str]) -> List[str]:
        if len(v) > 10:
            raise ValueError('Maximum 10 tags allowed')
        return [tag.lower().strip() for tag in v if tag.strip()]

class PostCreate(PostBase):
    slug: Optional[str] = None

    @field_validator('slug', mode='before')
    @classmethod
    def generate_slug(cls, v: Optional[str], info) -> str:
        if v:
            return re.sub(r'[^a-z0-9]+', '-', v.lower()).strip('-')
        # Generate from title if not provided
        title = info.data.get('title', '')
        return re.sub(r'[^a-z0-9]+', '-', title.lower()).strip('-')

class PostUpdate(BaseModel):
    title: Optional[str] = Field(None, min_length=1, max_length=200)
    content: Optional[str] = None
    excerpt: Optional[str] = Field(None, max_length=500)
    tags: Optional[List[str]] = None
    is_published: Optional[bool] = None

    model_config = ConfigDict(extra='forbid')

class PostResponse(PostBase):
    id: int
    slug: str
    is_published: bool
    view_count: int
    author_id: int
    created_at: datetime
    updated_at: datetime

    model_config = ConfigDict(from_attributes=True)

class PostWithAuthor(PostResponse):
    author: "UserResponse"

class PostListResponse(BaseModel):
    items: List[PostResponse]
    total: int
    page: int
    per_page: int

# Forward reference resolution
from .user import UserResponse
PostWithAuthor.model_rebuild()
UserWithPosts.model_rebuild()
```

---

## 5. CRUD Operations

### Generic CRUD Base Class

```python
# crud/base.py
from typing import TypeVar, Generic, Type, Optional, List, Any
from sqlalchemy import select, func, update, delete
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel

from models.base import BaseModel as DBBaseModel

ModelType = TypeVar("ModelType", bound=DBBaseModel)
CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)

class CRUDBase(Generic[ModelType, CreateSchemaType, UpdateSchemaType]):
    """Generic CRUD operations for SQLAlchemy models."""

    def __init__(self, model: Type[ModelType]):
        self.model = model

    async def get(self, db: AsyncSession, id: int) -> Optional[ModelType]:
        """Get a single record by ID."""
        result = await db.execute(
            select(self.model).filter(self.model.id == id)
        )
        return result.scalar_one_or_none()

    async def get_multi(
        self,
        db: AsyncSession,
        *,
        skip: int = 0,
        limit: int = 100,
        order_by: Optional[str] = None,
    ) -> List[ModelType]:
        """Get multiple records with pagination."""
        query = select(self.model).offset(skip).limit(limit)

        if order_by:
            column = getattr(self.model, order_by, None)
            if column:
                query = query.order_by(column.desc())

        result = await db.execute(query)
        return list(result.scalars().all())

    async def get_count(self, db: AsyncSession) -> int:
        """Get total count of records."""
        result = await db.execute(
            select(func.count()).select_from(self.model)
        )
        return result.scalar() or 0

    async def create(
        self,
        db: AsyncSession,
        *,
        obj_in: CreateSchemaType,
        **extra_data: Any,
    ) -> ModelType:
        """Create a new record."""
        obj_data = obj_in.model_dump()
        obj_data.update(extra_data)
        db_obj = self.model(**obj_data)
        db.add(db_obj)
        await db.commit()
        await db.refresh(db_obj)
        return db_obj

    async def update(
        self,
        db: AsyncSession,
        *,
        db_obj: ModelType,
        obj_in: UpdateSchemaType | dict[str, Any],
    ) -> ModelType:
        """Update an existing record."""
        if isinstance(obj_in, dict):
            update_data = obj_in
        else:
            update_data = obj_in.model_dump(exclude_unset=True)

        for field, value in update_data.items():
            if hasattr(db_obj, field):
                setattr(db_obj, field, value)

        db.add(db_obj)
        await db.commit()
        await db.refresh(db_obj)
        return db_obj

    async def delete(self, db: AsyncSession, *, id: int) -> bool:
        """Delete a record by ID."""
        result = await db.execute(
            delete(self.model).where(self.model.id == id)
        )
        await db.commit()
        return result.rowcount > 0

    async def exists(self, db: AsyncSession, id: int) -> bool:
        """Check if a record exists."""
        result = await db.execute(
            select(func.count()).select_from(self.model).filter(self.model.id == id)
        )
        return (result.scalar() or 0) > 0
```

### User CRUD with Custom Methods

```python
# crud/user.py
from typing import Optional, List
from sqlalchemy import select, or_
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from .base import CRUDBase
from models.user import User
from schemas.user import UserCreate, UserUpdate
from core.security import get_password_hash, verify_password

class CRUDUser(CRUDBase[User, UserCreate, UserUpdate]):

    async def get_by_email(self, db: AsyncSession, email: str) -> Optional[User]:
        """Get user by email address."""
        result = await db.execute(
            select(User).filter(User.email == email.lower())
        )
        return result.scalar_one_or_none()

    async def get_by_username(self, db: AsyncSession, username: str) -> Optional[User]:
        """Get user by username."""
        result = await db.execute(
            select(User).filter(User.username == username.lower())
        )
        return result.scalar_one_or_none()

    async def get_by_email_or_username(
        self,
        db: AsyncSession,
        identifier: str,
    ) -> Optional[User]:
        """Get user by email or username."""
        result = await db.execute(
            select(User).filter(
                or_(
                    User.email == identifier.lower(),
                    User.username == identifier.lower(),
                )
            )
        )
        return result.scalar_one_or_none()

    async def get_with_posts(self, db: AsyncSession, user_id: int) -> Optional[User]:
        """Get user with their posts eagerly loaded."""
        result = await db.execute(
            select(User)
            .options(selectinload(User.posts))
            .filter(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def create(self, db: AsyncSession, *, obj_in: UserCreate) -> User:
        """Create user with hashed password."""
        db_obj = User(
            email=obj_in.email.lower(),
            username=obj_in.username.lower(),
            full_name=obj_in.full_name,
            bio=obj_in.bio,
            hashed_password=get_password_hash(obj_in.password),
        )
        db.add(db_obj)
        await db.commit()
        await db.refresh(db_obj)
        return db_obj

    async def authenticate(
        self,
        db: AsyncSession,
        *,
        email: str,
        password: str,
    ) -> Optional[User]:
        """Authenticate user by email and password."""
        user = await self.get_by_email(db, email=email)
        if not user:
            return None
        if not verify_password(password, user.hashed_password):
            return None
        return user

    async def search(
        self,
        db: AsyncSession,
        *,
        query: str,
        skip: int = 0,
        limit: int = 20,
    ) -> List[User]:
        """Search users by username or full name."""
        search_term = f"%{query}%"
        result = await db.execute(
            select(User)
            .filter(
                or_(
                    User.username.ilike(search_term),
                    User.full_name.ilike(search_term),
                )
            )
            .offset(skip)
            .limit(limit)
        )
        return list(result.scalars().all())

    async def is_active(self, user: User) -> bool:
        """Check if user is active."""
        return user.status == "active"

# Singleton instance
user_crud = CRUDUser(User)
```

### Post CRUD with Filtering

```python
# crud/post.py
from typing import Optional, List
from sqlalchemy import select, func, and_
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from .base import CRUDBase
from models.post import Post
from schemas.post import PostCreate, PostUpdate

class CRUDPost(CRUDBase[Post, PostCreate, PostUpdate]):

    async def get_by_slug(self, db: AsyncSession, slug: str) -> Optional[Post]:
        """Get post by slug."""
        result = await db.execute(
            select(Post).filter(Post.slug == slug)
        )
        return result.scalar_one_or_none()

    async def get_with_author(self, db: AsyncSession, post_id: int) -> Optional[Post]:
        """Get post with author eagerly loaded."""
        result = await db.execute(
            select(Post)
            .options(selectinload(Post.author))
            .filter(Post.id == post_id)
        )
        return result.scalar_one_or_none()

    async def get_published(
        self,
        db: AsyncSession,
        *,
        skip: int = 0,
        limit: int = 20,
    ) -> List[Post]:
        """Get published posts."""
        result = await db.execute(
            select(Post)
            .filter(Post.is_published == True)
            .order_by(Post.created_at.desc())
            .offset(skip)
            .limit(limit)
        )
        return list(result.scalars().all())

    async def get_by_author(
        self,
        db: AsyncSession,
        *,
        author_id: int,
        include_unpublished: bool = False,
        skip: int = 0,
        limit: int = 20,
    ) -> List[Post]:
        """Get posts by author."""
        query = select(Post).filter(Post.author_id == author_id)

        if not include_unpublished:
            query = query.filter(Post.is_published == True)

        query = query.order_by(Post.created_at.desc()).offset(skip).limit(limit)

        result = await db.execute(query)
        return list(result.scalars().all())

    async def get_by_tag(
        self,
        db: AsyncSession,
        *,
        tag: str,
        skip: int = 0,
        limit: int = 20,
    ) -> List[Post]:
        """Get posts containing a specific tag."""
        result = await db.execute(
            select(Post)
            .filter(Post.tags.contains([tag.lower()]))
            .filter(Post.is_published == True)
            .order_by(Post.created_at.desc())
            .offset(skip)
            .limit(limit)
        )
        return list(result.scalars().all())

    async def increment_view_count(self, db: AsyncSession, post_id: int) -> None:
        """Increment the view count of a post."""
        await db.execute(
            select(Post).filter(Post.id == post_id).with_for_update()
        )
        result = await db.execute(
            select(Post).filter(Post.id == post_id)
        )
        post = result.scalar_one_or_none()
        if post:
            post.view_count += 1
            await db.commit()

    async def create(
        self,
        db: AsyncSession,
        *,
        obj_in: PostCreate,
        author_id: int,
    ) -> Post:
        """Create post with author."""
        return await super().create(db, obj_in=obj_in, author_id=author_id)

# Singleton instance
post_crud = CRUDPost(Post)
```

---

## 6. Dependency Injection for DB Sessions

### Basic Database Dependency

```python
# api/deps.py
from typing import AsyncGenerator
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from database import AsyncSessionLocal

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """Dependency that provides a database session."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

### Database Dependency with Transaction

```python
# api/deps.py
from typing import AsyncGenerator
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db_with_transaction() -> AsyncGenerator[AsyncSession, None]:
    """Dependency that provides a database session with automatic transaction."""
    async with AsyncSessionLocal() as session:
        async with session.begin():
            try:
                yield session
            except Exception:
                await session.rollback()
                raise
            # Auto-commits on successful exit
```

### Combined Dependencies

```python
# api/deps.py
from typing import Optional
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
from jose import JWTError, jwt

from database import AsyncSessionLocal
from models.user import User
from crud.user import user_crud
from core.config import settings

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()

async def get_current_user(
    db: AsyncSession = Depends(get_db),
    token: str = Depends(oauth2_scheme),
) -> User:
    """Get current authenticated user from JWT token."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        user_id: int = int(payload.get("sub"))
        if user_id is None:
            raise credentials_exception
    except (JWTError, ValueError):
        raise credentials_exception

    user = await user_crud.get(db, id=user_id)
    if user is None:
        raise credentials_exception

    return user

async def get_current_active_user(
    current_user: User = Depends(get_current_user),
) -> User:
    """Ensure the current user is active."""
    if not user_crud.is_active(current_user):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Inactive user"
        )
    return current_user

async def get_current_admin_user(
    current_user: User = Depends(get_current_active_user),
) -> User:
    """Ensure the current user is an admin."""
    if not current_user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin privileges required"
        )
    return current_user

def get_optional_current_user(
    db: AsyncSession = Depends(get_db),
    token: Optional[str] = Depends(oauth2_scheme),
) -> Optional[User]:
    """Optionally get current user (for public endpoints)."""
    if token is None:
        return None
    try:
        return get_current_user(db, token)
    except HTTPException:
        return None
```

### Using Dependencies in Routes

```python
# api/routes/users.py
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List

from api.deps import get_db, get_current_user, get_current_admin_user
from models.user import User
from schemas.user import UserResponse, UserUpdate, UserListResponse
from crud.user import user_crud

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/me", response_model=UserResponse)
async def get_current_user_info(
    current_user: User = Depends(get_current_user),
):
    """Get current user's information."""
    return current_user

@router.patch("/me", response_model=UserResponse)
async def update_current_user(
    user_update: UserUpdate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Update current user's information."""
    return await user_crud.update(db, db_obj=current_user, obj_in=user_update)

@router.get("/", response_model=UserListResponse)
async def list_users(
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_admin_user),  # Admin only
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
):
    """List all users (admin only)."""
    skip = (page - 1) * per_page
    users = await user_crud.get_multi(db, skip=skip, limit=per_page)
    total = await user_crud.get_count(db)

    return UserListResponse(
        items=users,
        total=total,
        page=page,
        per_page=per_page,
        pages=(total + per_page - 1) // per_page,
    )
```

---

## 7. Async Database Operations

### asyncpg with SQLAlchemy

```python
# database.py - Full async setup with asyncpg
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
    AsyncEngine,
)
from sqlalchemy.orm import declarative_base
from sqlalchemy.pool import NullPool, AsyncAdaptedQueuePool

DATABASE_URL = "postgresql+asyncpg://user:password@localhost:5432/dbname"

# Create async engine
async_engine: AsyncEngine = create_async_engine(
    DATABASE_URL,
    echo=True,
    poolclass=AsyncAdaptedQueuePool,  # Connection pooling
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,
    # asyncpg-specific options
    connect_args={
        "server_settings": {
            "jit": "off",  # Disable JIT for faster connection
        },
        "command_timeout": 60,
    },
)

# Session factory
AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autocommit=False,
    autoflush=False,
)

Base = declarative_base()
```

### Using the `databases` Library

```python
# database_raw.py - Using encode/databases for raw queries
from databases import Database
from sqlalchemy import MetaData, Table, Column, Integer, String, create_engine

DATABASE_URL = "postgresql://user:password@localhost:5432/dbname"
ASYNC_DATABASE_URL = "postgresql+asyncpg://user:password@localhost:5432/dbname"

# For raw async queries
database = Database(ASYNC_DATABASE_URL)

# For SQLAlchemy table definitions
metadata = MetaData()

users = Table(
    "users",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("email", String(255), unique=True),
    Column("name", String(100)),
)

# Connect/disconnect in lifespan
async def connect_db():
    await database.connect()

async def disconnect_db():
    await database.disconnect()

# Raw query examples
async def get_user_raw(user_id: int):
    query = users.select().where(users.c.id == user_id)
    return await database.fetch_one(query)

async def get_users_raw(skip: int = 0, limit: int = 100):
    query = users.select().offset(skip).limit(limit)
    return await database.fetch_all(query)

async def create_user_raw(email: str, name: str):
    query = users.insert()
    values = {"email": email, "name": name}
    last_id = await database.execute(query, values)
    return last_id

async def execute_raw_sql():
    query = "SELECT * FROM users WHERE email LIKE :pattern"
    rows = await database.fetch_all(query, values={"pattern": "%@example.com"})
    return rows
```

### Async Bulk Operations

```python
# crud/bulk.py
from typing import List, TypeVar, Type
from sqlalchemy import insert, update, delete
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.dialects.postgresql import insert as pg_insert

from models.base import BaseModel

T = TypeVar("T", bound=BaseModel)

async def bulk_create(
    db: AsyncSession,
    model: Type[T],
    items: List[dict],
) -> List[T]:
    """Bulk insert multiple records."""
    stmt = insert(model).values(items).returning(model)
    result = await db.execute(stmt)
    await db.commit()
    return list(result.scalars().all())

async def bulk_upsert(
    db: AsyncSession,
    model: Type[T],
    items: List[dict],
    index_elements: List[str],
    update_fields: List[str],
) -> None:
    """Bulk upsert (insert or update on conflict)."""
    stmt = pg_insert(model).values(items)

    update_dict = {field: stmt.excluded[field] for field in update_fields}

    stmt = stmt.on_conflict_do_update(
        index_elements=index_elements,
        set_=update_dict,
    )

    await db.execute(stmt)
    await db.commit()

async def bulk_update(
    db: AsyncSession,
    model: Type[T],
    items: List[dict],  # Must include 'id' field
) -> int:
    """Bulk update multiple records."""
    updated_count = 0
    for item in items:
        item_id = item.pop('id')
        stmt = (
            update(model)
            .where(model.id == item_id)
            .values(**item)
        )
        result = await db.execute(stmt)
        updated_count += result.rowcount

    await db.commit()
    return updated_count

async def bulk_delete(
    db: AsyncSession,
    model: Type[T],
    ids: List[int],
) -> int:
    """Bulk delete multiple records by ID."""
    stmt = delete(model).where(model.id.in_(ids))
    result = await db.execute(stmt)
    await db.commit()
    return result.rowcount

# Usage example
async def example_bulk_operations(db: AsyncSession):
    from models.user import User

    # Bulk create
    users_data = [
        {"email": "user1@example.com", "username": "user1", "hashed_password": "..."},
        {"email": "user2@example.com", "username": "user2", "hashed_password": "..."},
    ]
    new_users = await bulk_create(db, User, users_data)

    # Bulk upsert
    await bulk_upsert(
        db,
        User,
        users_data,
        index_elements=["email"],
        update_fields=["username"],
    )
```

---

## 8. Relationships (One-to-Many, Many-to-Many)

### One-to-Many Relationship

```python
# models/relationships.py
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

from .base import BaseModel

class Author(BaseModel):
    __tablename__ = "authors"

    name = Column(String(100), nullable=False)

    # One-to-Many: An author has many books
    books = relationship(
        "Book",
        back_populates="author",
        cascade="all, delete-orphan",  # Delete books when author is deleted
        lazy="selectin",               # Eager loading strategy
        order_by="Book.title",         # Default ordering
    )

class Book(BaseModel):
    __tablename__ = "books"

    title = Column(String(200), nullable=False)
    isbn = Column(String(20), unique=True)

    # Foreign key to Author
    author_id = Column(
        Integer,
        ForeignKey("authors.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )

    # Many-to-One: A book belongs to one author
    author = relationship("Author", back_populates="books")
```

### Many-to-Many Relationship

```python
# models/many_to_many.py
from sqlalchemy import Column, Integer, String, ForeignKey, Table
from sqlalchemy.orm import relationship

from .base import Base, BaseModel

# Association table for many-to-many
book_tags = Table(
    "book_tags",
    Base.metadata,
    Column("book_id", Integer, ForeignKey("books.id", ondelete="CASCADE"), primary_key=True),
    Column("tag_id", Integer, ForeignKey("tags.id", ondelete="CASCADE"), primary_key=True),
)

class Book(BaseModel):
    __tablename__ = "books"

    title = Column(String(200), nullable=False)

    # Many-to-Many with Tag
    tags = relationship(
        "Tag",
        secondary=book_tags,
        back_populates="books",
        lazy="selectin",
    )

class Tag(BaseModel):
    __tablename__ = "tags"

    name = Column(String(50), unique=True, nullable=False)

    # Many-to-Many with Book
    books = relationship(
        "Book",
        secondary=book_tags,
        back_populates="tags",
        lazy="selectin",
    )
```

### Many-to-Many with Extra Fields (Association Object)

```python
# models/association.py
from sqlalchemy import Column, Integer, String, ForeignKey, DateTime
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func

from .base import BaseModel

class Student(BaseModel):
    __tablename__ = "students"

    name = Column(String(100), nullable=False)

    # Relationship through association object
    enrollments = relationship(
        "Enrollment",
        back_populates="student",
        cascade="all, delete-orphan",
    )

    # Convenience property to get courses
    @property
    def courses(self):
        return [enrollment.course for enrollment in self.enrollments]

class Course(BaseModel):
    __tablename__ = "courses"

    name = Column(String(200), nullable=False)
    code = Column(String(20), unique=True, nullable=False)

    enrollments = relationship(
        "Enrollment",
        back_populates="course",
        cascade="all, delete-orphan",
    )

    @property
    def students(self):
        return [enrollment.student for enrollment in self.enrollments]

class Enrollment(BaseModel):
    """Association object with extra fields."""
    __tablename__ = "enrollments"

    student_id = Column(
        Integer,
        ForeignKey("students.id", ondelete="CASCADE"),
        nullable=False,
    )
    course_id = Column(
        Integer,
        ForeignKey("courses.id", ondelete="CASCADE"),
        nullable=False,
    )

    # Extra fields
    grade = Column(String(2), nullable=True)
    enrolled_at = Column(DateTime(timezone=True), server_default=func.now())

    # Relationships
    student = relationship("Student", back_populates="enrollments")
    course = relationship("Course", back_populates="enrollments")
```

### Self-Referential Relationship

```python
# models/self_referential.py
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

from .base import BaseModel

class Category(BaseModel):
    """Category with parent-child hierarchy."""
    __tablename__ = "categories"

    name = Column(String(100), nullable=False)

    # Self-referential foreign key
    parent_id = Column(
        Integer,
        ForeignKey("categories.id", ondelete="CASCADE"),
        nullable=True,
        index=True,
    )

    # Parent relationship
    parent = relationship(
        "Category",
        remote_side="Category.id",
        backref="children",
    )

    @property
    def ancestors(self):
        """Get all ancestor categories."""
        ancestors = []
        current = self.parent
        while current:
            ancestors.append(current)
            current = current.parent
        return ancestors

    @property
    def path(self) -> str:
        """Get full path from root to this category."""
        ancestors = self.ancestors[::-1]
        return " > ".join([a.name for a in ancestors] + [self.name])
```

### Querying Relationships

```python
# crud/relationships.py
from sqlalchemy import select
from sqlalchemy.orm import selectinload, joinedload, contains_eager
from sqlalchemy.ext.asyncio import AsyncSession

from models.relationships import Author, Book
from models.many_to_many import Tag

async def get_author_with_books(db: AsyncSession, author_id: int):
    """Get author with all their books (eager loading)."""
    result = await db.execute(
        select(Author)
        .options(selectinload(Author.books))
        .filter(Author.id == author_id)
    )
    return result.scalar_one_or_none()

async def get_book_with_tags(db: AsyncSession, book_id: int):
    """Get book with all its tags."""
    result = await db.execute(
        select(Book)
        .options(selectinload(Book.tags))
        .filter(Book.id == book_id)
    )
    return result.scalar_one_or_none()

async def get_books_by_tag(db: AsyncSession, tag_name: str):
    """Get all books with a specific tag."""
    result = await db.execute(
        select(Book)
        .join(Book.tags)
        .filter(Tag.name == tag_name)
        .options(selectinload(Book.tags))
    )
    return list(result.scalars().unique().all())

async def add_tag_to_book(db: AsyncSession, book_id: int, tag_name: str):
    """Add a tag to a book."""
    # Get or create tag
    result = await db.execute(select(Tag).filter(Tag.name == tag_name))
    tag = result.scalar_one_or_none()

    if not tag:
        tag = Tag(name=tag_name)
        db.add(tag)

    # Get book and add tag
    result = await db.execute(
        select(Book)
        .options(selectinload(Book.tags))
        .filter(Book.id == book_id)
    )
    book = result.scalar_one_or_none()

    if book and tag not in book.tags:
        book.tags.append(tag)
        await db.commit()

    return book
```

---

## 9. Query Optimization

### Eager Loading Strategies

```python
# Avoid N+1 queries with proper loading strategies
from sqlalchemy.orm import selectinload, joinedload, subqueryload, lazyload

# selectinload: Best for collections, uses IN clause
async def get_users_with_posts_selectin(db: AsyncSession):
    result = await db.execute(
        select(User)
        .options(selectinload(User.posts))
    )
    return result.scalars().all()

# joinedload: Best for single objects, uses LEFT JOIN
async def get_posts_with_author_joined(db: AsyncSession):
    result = await db.execute(
        select(Post)
        .options(joinedload(Post.author))
    )
    return result.scalars().unique().all()

# subqueryload: Alternative to selectinload
async def get_authors_with_books_subquery(db: AsyncSession):
    result = await db.execute(
        select(Author)
        .options(subqueryload(Author.books))
    )
    return result.scalars().all()

# Nested eager loading
async def get_users_with_posts_and_comments(db: AsyncSession):
    result = await db.execute(
        select(User)
        .options(
            selectinload(User.posts).selectinload(Post.comments)
        )
    )
    return result.scalars().all()

# Conditional loading
async def get_users_with_published_posts(db: AsyncSession):
    from sqlalchemy.orm import contains_eager

    result = await db.execute(
        select(User)
        .outerjoin(User.posts)
        .filter(Post.is_published == True)
        .options(contains_eager(User.posts))
    )
    return result.scalars().unique().all()
```

### Query Performance Techniques

```python
# crud/optimized.py
from sqlalchemy import select, func, and_, or_, text
from sqlalchemy.ext.asyncio import AsyncSession

# Only select needed columns
async def get_user_emails(db: AsyncSession):
    """Select only specific columns."""
    result = await db.execute(
        select(User.id, User.email)
    )
    return result.all()

# Use exists for checking
async def email_exists(db: AsyncSession, email: str) -> bool:
    """Efficient existence check."""
    result = await db.execute(
        select(func.count()).select_from(
            select(User.id).filter(User.email == email).exists().select()
        )
    )
    return result.scalar() > 0

# Pagination with count
async def get_paginated_posts(
    db: AsyncSession,
    page: int = 1,
    per_page: int = 20,
):
    """Efficient pagination with total count."""
    # Count query
    count_result = await db.execute(
        select(func.count()).select_from(Post).filter(Post.is_published == True)
    )
    total = count_result.scalar()

    # Data query
    offset = (page - 1) * per_page
    result = await db.execute(
        select(Post)
        .filter(Post.is_published == True)
        .order_by(Post.created_at.desc())
        .offset(offset)
        .limit(per_page)
    )
    items = result.scalars().all()

    return {
        "items": items,
        "total": total,
        "page": page,
        "per_page": per_page,
        "pages": (total + per_page - 1) // per_page,
    }

# Cursor-based pagination (more efficient for large datasets)
async def get_posts_cursor(
    db: AsyncSession,
    cursor: int = None,
    limit: int = 20,
):
    """Cursor-based pagination using ID."""
    query = select(Post).filter(Post.is_published == True)

    if cursor:
        query = query.filter(Post.id < cursor)

    query = query.order_by(Post.id.desc()).limit(limit + 1)

    result = await db.execute(query)
    items = list(result.scalars().all())

    has_more = len(items) > limit
    if has_more:
        items = items[:-1]

    next_cursor = items[-1].id if items and has_more else None

    return {
        "items": items,
        "next_cursor": next_cursor,
        "has_more": has_more,
    }

# Full-text search (PostgreSQL)
async def search_posts_fulltext(
    db: AsyncSession,
    search_query: str,
    limit: int = 20,
):
    """PostgreSQL full-text search."""
    result = await db.execute(
        select(Post)
        .filter(
            func.to_tsvector('english', Post.title + ' ' + Post.content)
            .match(search_query)
        )
        .limit(limit)
    )
    return result.scalars().all()
```

### Database Indexes

```python
# models/indexed.py
from sqlalchemy import Column, Integer, String, Index, UniqueConstraint
from sqlalchemy.dialects.postgresql import GIN
from sqlalchemy.sql import func

from .base import BaseModel

class OptimizedPost(BaseModel):
    __tablename__ = "optimized_posts"

    title = Column(String(200), nullable=False)
    content = Column(String, nullable=True)
    slug = Column(String(250), nullable=False)
    author_id = Column(Integer, nullable=False)
    status = Column(String(20), nullable=False, default="draft")

    __table_args__ = (
        # Single column indexes
        Index('ix_posts_slug', 'slug', unique=True),
        Index('ix_posts_author', 'author_id'),

        # Composite index for common queries
        Index('ix_posts_author_status', 'author_id', 'status'),

        # Partial index (PostgreSQL)
        Index(
            'ix_posts_published',
            'created_at',
            postgresql_where=(status == 'published'),
        ),

        # GIN index for full-text search (PostgreSQL)
        Index(
            'ix_posts_search',
            func.to_tsvector('english', title + ' ' + content),
            postgresql_using='gin',
        ),

        # Unique constraint
        UniqueConstraint('author_id', 'slug', name='uq_author_slug'),
    )
```

---

## 10. Database Migrations with Alembic

### Initial Setup

```bash
# Initialize Alembic
pip install alembic
alembic init alembic

# Directory structure created:
# alembic/
#   ├── versions/          # Migration files
#   ├── env.py             # Environment configuration
#   ├── script.py.mako     # Migration template
#   └── README
# alembic.ini              # Alembic configuration
```

### Configuration for Async (alembic.ini)

```ini
# alembic.ini
[alembic]
script_location = alembic
prepend_sys_path = .
version_path_separator = os

# Use async driver
sqlalchemy.url = postgresql+asyncpg://user:password@localhost/dbname

[post_write_hooks]
hooks = black
black.type = console_scripts
black.entrypoint = black
black.options = -l 100
```

### Async Environment Configuration

```python
# alembic/env.py
import asyncio
from logging.config import fileConfig

from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config

from alembic import context

# Import your models' Base
from app.models.base import Base
from app.core.config import settings

# This is the Alembic Config object
config = context.config

# Set the database URL from settings
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)

# Interpret the config file for Python logging
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Add your model's MetaData object for 'autogenerate' support
target_metadata = Base.metadata

def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def do_run_migrations(connection: Connection) -> None:
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,        # Detect column type changes
        compare_server_default=True,  # Detect default value changes
    )

    with context.begin_transaction():
        context.run_migrations()

async def run_async_migrations() -> None:
    """Run migrations in 'online' mode with async engine."""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()

def run_migrations_online() -> None:
    """Run migrations in 'online' mode."""
    asyncio.run(run_async_migrations())

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Migration Commands

```bash
# Generate migration from model changes
alembic revision --autogenerate -m "Add user table"

# Apply all pending migrations
alembic upgrade head

# Apply specific revision
alembic upgrade <revision_id>

# Rollback one migration
alembic downgrade -1

# Rollback to specific revision
alembic downgrade <revision_id>

# Show current revision
alembic current

# Show migration history
alembic history

# Show pending migrations
alembic history --indicate-current
```

### Example Migration File

```python
# alembic/versions/001_add_user_table.py
"""Add user table

Revision ID: 001
Revises:
Create Date: 2024-01-15 10:00:00.000000
"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# revision identifiers, used by Alembic.
revision: str = '001'
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

def upgrade() -> None:
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('email', sa.String(255), nullable=False),
        sa.Column('username', sa.String(50), nullable=False),
        sa.Column('hashed_password', sa.String(255), nullable=False),
        sa.Column('full_name', sa.String(100), nullable=True),
        sa.Column('role', sa.Enum('admin', 'moderator', 'user', name='userrole'), nullable=False),
        sa.Column('is_verified', sa.Boolean(), nullable=False, default=False),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
    )

    # Create indexes
    op.create_index('ix_users_email', 'users', ['email'], unique=True)
    op.create_index('ix_users_username', 'users', ['username'], unique=True)

def downgrade() -> None:
    op.drop_index('ix_users_username', table_name='users')
    op.drop_index('ix_users_email', table_name='users')
    op.drop_table('users')

    # Drop enum type
    op.execute("DROP TYPE IF EXISTS userrole")
```

### Data Migration Example

```python
# alembic/versions/002_seed_admin_user.py
"""Seed admin user

Revision ID: 002
Revises: 001
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.sql import table, column

revision = '002'
down_revision = '001'

def upgrade() -> None:
    # Define table structure for data operation
    users = table(
        'users',
        column('id', sa.Integer),
        column('email', sa.String),
        column('username', sa.String),
        column('hashed_password', sa.String),
        column('role', sa.String),
        column('is_verified', sa.Boolean),
    )

    # Insert seed data
    op.bulk_insert(users, [
        {
            'email': 'admin@example.com',
            'username': 'admin',
            'hashed_password': '$2b$12$...',  # Pre-hashed password
            'role': 'admin',
            'is_verified': True,
        }
    ])

def downgrade() -> None:
    op.execute("DELETE FROM users WHERE email = 'admin@example.com'")
```

---

## 11. Transaction Management

### Basic Transactions

```python
# Automatic transaction with context manager
async def transfer_funds(
    db: AsyncSession,
    from_account_id: int,
    to_account_id: int,
    amount: float,
):
    """Transfer funds between accounts atomically."""
    async with db.begin():
        # Debit from source account
        from_account = await db.get(Account, from_account_id, with_for_update=True)
        if from_account.balance < amount:
            raise ValueError("Insufficient funds")
        from_account.balance -= amount

        # Credit to destination account
        to_account = await db.get(Account, to_account_id, with_for_update=True)
        to_account.balance += amount

        # Create transaction record
        transaction = Transaction(
            from_account_id=from_account_id,
            to_account_id=to_account_id,
            amount=amount,
        )
        db.add(transaction)

        # Commits automatically on success, rolls back on exception
```

### Nested Transactions (Savepoints)

```python
async def complex_operation(db: AsyncSession):
    """Operation with nested savepoints."""
    async with db.begin():  # Outer transaction
        user = User(email="user@example.com", username="user")
        db.add(user)
        await db.flush()  # Get the ID without committing

        try:
            async with db.begin_nested():  # Savepoint
                post = Post(title="Post", author_id=user.id)
                db.add(post)
                await db.flush()

                # This might fail
                comment = Comment(content="", post_id=post.id)  # Invalid
                db.add(comment)
                await db.flush()
        except Exception:
            # Savepoint is rolled back, but user is still created
            pass

        # User still exists, post and comment are rolled back
        # Outer transaction commits
```

### Manual Transaction Control

```python
async def manual_transaction(db: AsyncSession):
    """Manual transaction control."""
    try:
        user = User(email="user@example.com", username="user")
        db.add(user)

        # Explicitly commit
        await db.commit()

        # Refresh to get database-generated values
        await db.refresh(user)

        return user
    except Exception as e:
        # Explicitly rollback on error
        await db.rollback()
        raise e
```

### Transaction with Isolation Level

```python
from sqlalchemy import text

async def serializable_transaction(db: AsyncSession):
    """Transaction with serializable isolation level."""
    # Set isolation level for this transaction
    await db.execute(text("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE"))

    async with db.begin():
        # Critical operations that require strict consistency
        result = await db.execute(
            select(Account)
            .filter(Account.id == 1)
            .with_for_update()
        )
        account = result.scalar_one()

        # Perform operations
        account.balance += 100
```

### Transaction Decorator

```python
from functools import wraps
from typing import Callable, TypeVar, ParamSpec

P = ParamSpec('P')
R = TypeVar('R')

def transactional(func: Callable[P, R]) -> Callable[P, R]:
    """Decorator to wrap function in a transaction."""
    @wraps(func)
    async def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        db: AsyncSession = kwargs.get('db') or args[0]

        async with db.begin():
            return await func(*args, **kwargs)

    return wrapper

# Usage
@transactional
async def create_user_with_profile(db: AsyncSession, user_data: dict):
    user = User(**user_data)
    db.add(user)
    await db.flush()

    profile = Profile(user_id=user.id)
    db.add(profile)

    return user
```

---

## 12. PostgreSQL with SQLAlchemy

### PostgreSQL-Specific Types

```python
# models/postgres_types.py
from sqlalchemy import Column, Integer, String, Text
from sqlalchemy.dialects.postgresql import (
    ARRAY,
    JSON,
    JSONB,
    UUID,
    INET,
    CIDR,
    MACADDR,
    TSVECTOR,
    DATERANGE,
    INT4RANGE,
    NUMRANGE,
    TSRANGE,
    TSTZRANGE,
)
from sqlalchemy.sql import func
import uuid

from .base import BaseModel

class PostgresModel(BaseModel):
    __tablename__ = "postgres_features"

    # UUID primary key
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)

    # Array types
    tags = Column(ARRAY(String(50)), default=list)
    scores = Column(ARRAY(Integer), default=list)

    # JSON types
    metadata = Column(JSON, default=dict)           # Standard JSON
    settings = Column(JSONB, default=dict)          # Binary JSON (faster queries)

    # Network types
    ip_address = Column(INET, nullable=True)
    network = Column(CIDR, nullable=True)
    mac = Column(MACADDR, nullable=True)

    # Full-text search
    search_vector = Column(TSVECTOR, nullable=True)

    # Range types
    date_range = Column(DATERANGE, nullable=True)
    int_range = Column(INT4RANGE, nullable=True)
```

### JSONB Operations

```python
from sqlalchemy import select, cast
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.ext.asyncio import AsyncSession

async def query_jsonb_field(db: AsyncSession):
    """Query JSONB fields."""

    # Query by nested JSON path
    result = await db.execute(
        select(PostgresModel)
        .filter(PostgresModel.settings['theme'].astext == 'dark')
    )

    # Query with containment operator @>
    result = await db.execute(
        select(PostgresModel)
        .filter(PostgresModel.settings.contains({'notifications': True}))
    )

    # Query array in JSONB
    result = await db.execute(
        select(PostgresModel)
        .filter(PostgresModel.settings['roles'].contains(['admin']))
    )

    # Update JSONB field
    from sqlalchemy.dialects.postgresql import insert

    stmt = (
        insert(PostgresModel)
        .values(id=uuid.uuid4(), settings={'theme': 'light'})
        .on_conflict_do_update(
            index_elements=['id'],
            set_={'settings': PostgresModel.settings.concat({'theme': 'dark'})}
        )
    )
    await db.execute(stmt)

async def update_jsonb_nested(db: AsyncSession, model_id: uuid.UUID):
    """Update nested JSONB field."""
    from sqlalchemy import func

    await db.execute(
        select(PostgresModel)
        .filter(PostgresModel.id == model_id)
        .with_for_update()
    )

    # Use jsonb_set function for nested updates
    await db.execute(
        PostgresModel.__table__.update()
        .where(PostgresModel.id == model_id)
        .values(
            settings=func.jsonb_set(
                PostgresModel.settings,
                ['nested', 'key'],
                '"new_value"',
                True  # Create missing keys
            )
        )
    )
    await db.commit()
```

### Full-Text Search with PostgreSQL

```python
from sqlalchemy import Column, String, Text, Index, func, event
from sqlalchemy.dialects.postgresql import TSVECTOR

class SearchablePost(BaseModel):
    __tablename__ = "searchable_posts"

    title = Column(String(200), nullable=False)
    content = Column(Text, nullable=True)

    # Full-text search vector
    search_vector = Column(
        TSVECTOR,
        Computed(
            "to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''))",
            persisted=True,
        ),
    )

    __table_args__ = (
        Index('ix_searchable_posts_fts', 'search_vector', postgresql_using='gin'),
    )

# Alternative: Update search vector via trigger
@event.listens_for(SearchablePost, 'before_insert')
@event.listens_for(SearchablePost, 'before_update')
def update_search_vector(mapper, connection, target):
    target.search_vector = func.to_tsvector(
        'english',
        f"{target.title or ''} {target.content or ''}"
    )

async def full_text_search(db: AsyncSession, query: str):
    """Perform full-text search with ranking."""
    search_query = func.plainto_tsquery('english', query)

    result = await db.execute(
        select(
            SearchablePost,
            func.ts_rank(SearchablePost.search_vector, search_query).label('rank')
        )
        .filter(SearchablePost.search_vector.match(query))
        .order_by(func.ts_rank(SearchablePost.search_vector, search_query).desc())
        .limit(20)
    )

    return result.all()

async def full_text_search_with_highlights(db: AsyncSession, query: str):
    """Search with highlighted snippets."""
    search_query = func.plainto_tsquery('english', query)

    result = await db.execute(
        select(
            SearchablePost.id,
            SearchablePost.title,
            func.ts_headline(
                'english',
                SearchablePost.content,
                search_query,
                'StartSel=<mark>, StopSel=</mark>, MaxWords=50'
            ).label('snippet'),
        )
        .filter(SearchablePost.search_vector.match(query))
        .limit(20)
    )

    return result.all()
```

---

## 13. MongoDB with Motor

### Motor Setup

```python
# database_mongo.py
from motor.motor_asyncio import AsyncIOMotorClient, AsyncIOMotorDatabase
from pymongo import IndexModel, ASCENDING, DESCENDING
from pydantic import BaseModel
from typing import Optional
import os

MONGODB_URL = os.getenv("MONGODB_URL", "mongodb://localhost:27017")
DATABASE_NAME = os.getenv("MONGODB_DATABASE", "myapp")

class MongoDB:
    client: Optional[AsyncIOMotorClient] = None
    db: Optional[AsyncIOMotorDatabase] = None

mongo = MongoDB()

async def connect_to_mongo():
    """Connect to MongoDB."""
    mongo.client = AsyncIOMotorClient(
        MONGODB_URL,
        maxPoolSize=10,
        minPoolSize=1,
    )
    mongo.db = mongo.client[DATABASE_NAME]

    # Create indexes
    await create_indexes()

async def close_mongo_connection():
    """Close MongoDB connection."""
    if mongo.client:
        mongo.client.close()

async def create_indexes():
    """Create database indexes."""
    # Users collection indexes
    await mongo.db.users.create_indexes([
        IndexModel([("email", ASCENDING)], unique=True),
        IndexModel([("username", ASCENDING)], unique=True),
        IndexModel([("created_at", DESCENDING)]),
    ])

    # Posts collection indexes
    await mongo.db.posts.create_indexes([
        IndexModel([("author_id", ASCENDING)]),
        IndexModel([("slug", ASCENDING)], unique=True),
        IndexModel([("tags", ASCENDING)]),
        IndexModel([("created_at", DESCENDING)]),
        # Text index for search
        IndexModel([("title", "text"), ("content", "text")]),
    ])

def get_database() -> AsyncIOMotorDatabase:
    """Get database instance."""
    return mongo.db
```

### MongoDB Models with Pydantic

```python
# schemas/mongo_models.py
from pydantic import BaseModel, Field, EmailStr, ConfigDict
from typing import Optional, List
from datetime import datetime
from bson import ObjectId

class PyObjectId(str):
    """Custom type for MongoDB ObjectId."""

    @classmethod
    def __get_validators__(cls):
        yield cls.validate

    @classmethod
    def validate(cls, v):
        if not ObjectId.is_valid(v):
            raise ValueError("Invalid ObjectId")
        return str(v)

class MongoBaseModel(BaseModel):
    """Base model for MongoDB documents."""
    model_config = ConfigDict(
        populate_by_name=True,
        arbitrary_types_allowed=True,
        json_encoders={ObjectId: str},
    )

class UserDocument(MongoBaseModel):
    id: Optional[PyObjectId] = Field(default=None, alias="_id")
    email: EmailStr
    username: str
    hashed_password: str
    full_name: Optional[str] = None
    role: str = "user"
    is_active: bool = True
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)

class UserCreate(BaseModel):
    email: EmailStr
    username: str
    password: str
    full_name: Optional[str] = None

class UserResponse(MongoBaseModel):
    id: PyObjectId = Field(alias="_id")
    email: EmailStr
    username: str
    full_name: Optional[str] = None
    role: str
    is_active: bool
    created_at: datetime

class PostDocument(MongoBaseModel):
    id: Optional[PyObjectId] = Field(default=None, alias="_id")
    title: str
    content: Optional[str] = None
    slug: str
    author_id: PyObjectId
    tags: List[str] = []
    is_published: bool = False
    view_count: int = 0
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

### MongoDB CRUD Operations

```python
# crud/mongo_crud.py
from typing import Optional, List
from motor.motor_asyncio import AsyncIOMotorDatabase
from bson import ObjectId
from datetime import datetime

from schemas.mongo_models import UserDocument, UserCreate, PostDocument

class MongoUserCRUD:
    def __init__(self, db: AsyncIOMotorDatabase):
        self.collection = db.users

    async def get(self, user_id: str) -> Optional[dict]:
        """Get user by ID."""
        return await self.collection.find_one({"_id": ObjectId(user_id)})

    async def get_by_email(self, email: str) -> Optional[dict]:
        """Get user by email."""
        return await self.collection.find_one({"email": email.lower()})

    async def get_multi(
        self,
        skip: int = 0,
        limit: int = 100,
        filters: dict = None,
    ) -> List[dict]:
        """Get multiple users with filtering."""
        query = filters or {}
        cursor = self.collection.find(query).skip(skip).limit(limit)
        return await cursor.to_list(length=limit)

    async def create(self, user: UserCreate, hashed_password: str) -> dict:
        """Create a new user."""
        user_doc = {
            "email": user.email.lower(),
            "username": user.username.lower(),
            "hashed_password": hashed_password,
            "full_name": user.full_name,
            "role": "user",
            "is_active": True,
            "created_at": datetime.utcnow(),
            "updated_at": datetime.utcnow(),
        }
        result = await self.collection.insert_one(user_doc)
        user_doc["_id"] = result.inserted_id
        return user_doc

    async def update(self, user_id: str, update_data: dict) -> Optional[dict]:
        """Update a user."""
        update_data["updated_at"] = datetime.utcnow()

        result = await self.collection.find_one_and_update(
            {"_id": ObjectId(user_id)},
            {"$set": update_data},
            return_document=True,
        )
        return result

    async def delete(self, user_id: str) -> bool:
        """Delete a user."""
        result = await self.collection.delete_one({"_id": ObjectId(user_id)})
        return result.deleted_count > 0

class MongoPostCRUD:
    def __init__(self, db: AsyncIOMotorDatabase):
        self.collection = db.posts

    async def get(self, post_id: str) -> Optional[dict]:
        return await self.collection.find_one({"_id": ObjectId(post_id)})

    async def get_by_slug(self, slug: str) -> Optional[dict]:
        return await self.collection.find_one({"slug": slug})

    async def get_by_author(
        self,
        author_id: str,
        include_unpublished: bool = False,
    ) -> List[dict]:
        query = {"author_id": ObjectId(author_id)}
        if not include_unpublished:
            query["is_published"] = True

        cursor = self.collection.find(query).sort("created_at", -1)
        return await cursor.to_list(length=100)

    async def search(self, query: str, limit: int = 20) -> List[dict]:
        """Full-text search."""
        cursor = self.collection.find(
            {"$text": {"$search": query}},
            {"score": {"$meta": "textScore"}},
        ).sort([("score", {"$meta": "textScore"})]).limit(limit)

        return await cursor.to_list(length=limit)

    async def get_by_tags(self, tags: List[str], limit: int = 20) -> List[dict]:
        """Get posts containing any of the tags."""
        cursor = self.collection.find(
            {"tags": {"$in": tags}, "is_published": True}
        ).sort("created_at", -1).limit(limit)

        return await cursor.to_list(length=limit)

    async def aggregate_by_author(self) -> List[dict]:
        """Aggregate post counts by author."""
        pipeline = [
            {"$match": {"is_published": True}},
            {"$group": {
                "_id": "$author_id",
                "post_count": {"$sum": 1},
                "total_views": {"$sum": "$view_count"},
            }},
            {"$sort": {"post_count": -1}},
        ]
        cursor = self.collection.aggregate(pipeline)
        return await cursor.to_list(length=100)
```

### MongoDB with FastAPI Integration

```python
# main.py
from fastapi import FastAPI, Depends, HTTPException
from contextlib import asynccontextmanager

from database_mongo import connect_to_mongo, close_mongo_connection, get_database

@asynccontextmanager
async def lifespan(app: FastAPI):
    await connect_to_mongo()
    yield
    await close_mongo_connection()

app = FastAPI(lifespan=lifespan)

# Dependency
def get_db():
    return get_database()

# Routes
@app.get("/users/{user_id}")
async def get_user(user_id: str, db = Depends(get_db)):
    from crud.mongo_crud import MongoUserCRUD

    crud = MongoUserCRUD(db)
    user = await crud.get(user_id)

    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return user
```

---

## 14. Testing with Test Database

### Test Configuration

```python
# tests/conftest.py
import asyncio
from typing import AsyncGenerator, Generator
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)
from sqlalchemy.pool import NullPool

from app.main import app
from app.database import get_db, Base
from app.core.config import settings

# Test database URL
TEST_DATABASE_URL = settings.DATABASE_URL.replace("/dbname", "/test_dbname")

# Create test engine
test_engine = create_async_engine(
    TEST_DATABASE_URL,
    poolclass=NullPool,  # Disable pooling for tests
    echo=True,
)

# Test session factory
TestAsyncSessionLocal = async_sessionmaker(
    bind=test_engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autocommit=False,
    autoflush=False,
)

@pytest.fixture(scope="session")
def event_loop() -> Generator:
    """Create event loop for async tests."""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="session")
async def setup_database():
    """Create test database tables."""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

@pytest.fixture
async def db_session(setup_database) -> AsyncGenerator[AsyncSession, None]:
    """Get test database session with transaction rollback."""
    async with TestAsyncSessionLocal() as session:
        async with session.begin():
            yield session
            await session.rollback()

@pytest.fixture
async def client(db_session: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    """Get test client with overridden database dependency."""

    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac

    app.dependency_overrides.clear()
```

### Test Fixtures for Data

```python
# tests/fixtures.py
import pytest
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.user import User
from app.models.post import Post
from app.core.security import get_password_hash

@pytest.fixture
async def test_user(db_session: AsyncSession) -> User:
    """Create a test user."""
    user = User(
        email="test@example.com",
        username="testuser",
        hashed_password=get_password_hash("testpassword"),
        full_name="Test User",
        is_verified=True,
    )
    db_session.add(user)
    await db_session.flush()
    await db_session.refresh(user)
    return user

@pytest.fixture
async def test_admin(db_session: AsyncSession) -> User:
    """Create a test admin user."""
    admin = User(
        email="admin@example.com",
        username="admin",
        hashed_password=get_password_hash("adminpassword"),
        full_name="Admin User",
        role="admin",
        is_verified=True,
        is_superuser=True,
    )
    db_session.add(admin)
    await db_session.flush()
    await db_session.refresh(admin)
    return admin

@pytest.fixture
async def test_posts(db_session: AsyncSession, test_user: User) -> list[Post]:
    """Create test posts."""
    posts = [
        Post(
            title=f"Test Post {i}",
            slug=f"test-post-{i}",
            content=f"Content for post {i}",
            author_id=test_user.id,
            is_published=True,
        )
        for i in range(5)
    ]
    db_session.add_all(posts)
    await db_session.flush()
    for post in posts:
        await db_session.refresh(post)
    return posts

@pytest.fixture
async def auth_headers(test_user: User) -> dict:
    """Get authentication headers for test user."""
    from app.core.security import create_access_token

    token = create_access_token(subject=str(test_user.id))
    return {"Authorization": f"Bearer {token}"}
```

### Test Examples

```python
# tests/test_users.py
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.user import User
from app.crud.user import user_crud

class TestUserEndpoints:

    @pytest.mark.asyncio
    async def test_create_user(self, client: AsyncClient):
        """Test user creation endpoint."""
        response = await client.post(
            "/api/users/",
            json={
                "email": "newuser@example.com",
                "username": "newuser",
                "password": "SecurePass123",
                "full_name": "New User",
            },
        )

        assert response.status_code == 201
        data = response.json()
        assert data["email"] == "newuser@example.com"
        assert data["username"] == "newuser"
        assert "password" not in data
        assert "hashed_password" not in data

    @pytest.mark.asyncio
    async def test_create_user_duplicate_email(
        self,
        client: AsyncClient,
        test_user: User,
    ):
        """Test that duplicate email is rejected."""
        response = await client.post(
            "/api/users/",
            json={
                "email": test_user.email,
                "username": "differentuser",
                "password": "SecurePass123",
            },
        )

        assert response.status_code == 400
        assert "already registered" in response.json()["detail"].lower()

    @pytest.mark.asyncio
    async def test_get_current_user(
        self,
        client: AsyncClient,
        test_user: User,
        auth_headers: dict,
    ):
        """Test getting current user info."""
        response = await client.get("/api/users/me", headers=auth_headers)

        assert response.status_code == 200
        data = response.json()
        assert data["id"] == test_user.id
        assert data["email"] == test_user.email

    @pytest.mark.asyncio
    async def test_get_current_user_unauthorized(self, client: AsyncClient):
        """Test unauthorized access to current user."""
        response = await client.get("/api/users/me")
        assert response.status_code == 401

class TestUserCRUD:

    @pytest.mark.asyncio
    async def test_get_user_by_email(
        self,
        db_session: AsyncSession,
        test_user: User,
    ):
        """Test getting user by email."""
        user = await user_crud.get_by_email(db_session, email=test_user.email)

        assert user is not None
        assert user.id == test_user.id
        assert user.email == test_user.email

    @pytest.mark.asyncio
    async def test_authenticate_user(
        self,
        db_session: AsyncSession,
        test_user: User,
    ):
        """Test user authentication."""
        user = await user_crud.authenticate(
            db_session,
            email=test_user.email,
            password="testpassword",
        )

        assert user is not None
        assert user.id == test_user.id

    @pytest.mark.asyncio
    async def test_authenticate_user_wrong_password(
        self,
        db_session: AsyncSession,
        test_user: User,
    ):
        """Test authentication with wrong password."""
        user = await user_crud.authenticate(
            db_session,
            email=test_user.email,
            password="wrongpassword",
        )

        assert user is None
```

### Factory Boy for Test Data

```python
# tests/factories.py
import factory
from factory.alchemy import SQLAlchemyModelFactory
from faker import Faker

from app.models.user import User
from app.models.post import Post
from app.core.security import get_password_hash

fake = Faker()

class BaseFactory(SQLAlchemyModelFactory):
    class Meta:
        abstract = True

class UserFactory(BaseFactory):
    class Meta:
        model = User

    email = factory.LazyAttribute(lambda _: fake.unique.email())
    username = factory.LazyAttribute(lambda _: fake.unique.user_name()[:50])
    hashed_password = factory.LazyFunction(lambda: get_password_hash("password123"))
    full_name = factory.LazyAttribute(lambda _: fake.name())
    role = "user"
    is_verified = True

class PostFactory(BaseFactory):
    class Meta:
        model = Post

    title = factory.LazyAttribute(lambda _: fake.sentence())
    slug = factory.LazyAttribute(lambda _: fake.slug())
    content = factory.LazyAttribute(lambda _: fake.paragraphs(3))
    is_published = True
    author_id = None  # Must be set explicitly
```

---

## 15. Connection Pooling

### SQLAlchemy Connection Pool Configuration

```python
# database.py - Advanced pool configuration
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.pool import (
    QueuePool,
    NullPool,
    AsyncAdaptedQueuePool,
    StaticPool,
)

# Production configuration with QueuePool
production_engine = create_async_engine(
    DATABASE_URL,
    poolclass=AsyncAdaptedQueuePool,

    # Pool size settings
    pool_size=5,           # Number of permanent connections
    max_overflow=10,       # Additional connections when pool is full

    # Timeout settings
    pool_timeout=30,       # Seconds to wait for connection
    pool_recycle=1800,     # Recycle connections after 30 minutes

    # Health checks
    pool_pre_ping=True,    # Test connection before use

    # Logging
    echo_pool=True,        # Log pool checkout/checkin
)

# Development configuration
development_engine = create_async_engine(
    DATABASE_URL,
    pool_size=2,
    max_overflow=3,
    pool_pre_ping=True,
    echo=True,
)

# Testing configuration (no pooling)
test_engine = create_async_engine(
    TEST_DATABASE_URL,
    poolclass=NullPool,    # Disable pooling for tests
)

# Single connection (SQLite in-memory)
sqlite_engine = create_async_engine(
    "sqlite+aiosqlite:///:memory:",
    poolclass=StaticPool,
    connect_args={"check_same_thread": False},
)
```

### Pool Events and Monitoring

```python
from sqlalchemy import event
from sqlalchemy.pool import Pool
import logging

logger = logging.getLogger(__name__)

@event.listens_for(Pool, "checkout")
def receive_checkout(dbapi_connection, connection_record, connection_proxy):
    """Called when a connection is retrieved from the pool."""
    logger.debug(f"Connection checked out: {connection_record}")

@event.listens_for(Pool, "checkin")
def receive_checkin(dbapi_connection, connection_record):
    """Called when a connection is returned to the pool."""
    logger.debug(f"Connection checked in: {connection_record}")

@event.listens_for(Pool, "connect")
def receive_connect(dbapi_connection, connection_record):
    """Called when a new connection is created."""
    logger.info(f"New connection created: {connection_record}")

@event.listens_for(Pool, "invalidate")
def receive_invalidate(dbapi_connection, connection_record, exception):
    """Called when a connection is invalidated."""
    logger.warning(f"Connection invalidated: {connection_record}, exception: {exception}")

# Pool statistics
async def get_pool_status(engine):
    """Get current pool status."""
    pool = engine.pool
    return {
        "pool_size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "invalid": pool.invalidatedcount() if hasattr(pool, 'invalidatedcount') else 0,
    }
```

### PgBouncer Integration

```python
# For use with PgBouncer connection pooler
# Use NullPool since PgBouncer handles pooling

from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.pool import NullPool

# PgBouncer URL (typically port 6432)
PGBOUNCER_URL = "postgresql+asyncpg://user:pass@localhost:6432/dbname"

engine = create_async_engine(
    PGBOUNCER_URL,
    poolclass=NullPool,  # Let PgBouncer handle pooling

    # Important for PgBouncer transaction mode
    connect_args={
        "prepared_statement_cache_size": 0,  # Disable prepared statements
    },
)
```

### Health Check Endpoint

```python
# api/routes/health.py
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text

from database import get_db, engine

router = APIRouter(tags=["health"])

@router.get("/health")
async def health_check():
    """Basic health check."""
    return {"status": "healthy"}

@router.get("/health/db")
async def database_health(db: AsyncSession = Depends(get_db)):
    """Database health check."""
    try:
        await db.execute(text("SELECT 1"))

        # Get pool stats
        pool = engine.pool
        pool_stats = {
            "pool_size": pool.size(),
            "checked_in": pool.checkedin(),
            "checked_out": pool.checkedout(),
        }

        return {
            "status": "healthy",
            "database": "connected",
            "pool": pool_stats,
        }
    except Exception as e:
        return {
            "status": "unhealthy",
            "database": "disconnected",
            "error": str(e),
        }
```

---

## 16. Best Practices

### Project Structure

```
app/
├── __init__.py
├── main.py                 # FastAPI app initialization
├── database.py             # Database configuration
├── core/
│   ├── config.py           # Settings with pydantic-settings
│   ├── security.py         # Password hashing, JWT
│   └── exceptions.py       # Custom exceptions
├── models/
│   ├── __init__.py         # Import all models
│   ├── base.py             # Base model class
│   └── *.py                # Domain models
├── schemas/
│   ├── __init__.py
│   └── *.py                # Pydantic schemas
├── crud/
│   ├── __init__.py
│   ├── base.py             # Generic CRUD
│   └── *.py                # Specific CRUD classes
├── api/
│   ├── __init__.py
│   ├── deps.py             # Dependencies
│   └── routes/
│       └── *.py            # Route handlers
└── tests/
    ├── conftest.py
    └── test_*.py
```

### Configuration Management

```python
# core/config.py
from pydantic_settings import BaseSettings
from pydantic import Field, PostgresDsn
from functools import lru_cache

class Settings(BaseSettings):
    # Application
    APP_NAME: str = "My FastAPI App"
    DEBUG: bool = False

    # Database
    DATABASE_URL: PostgresDsn
    DB_POOL_SIZE: int = Field(default=5, ge=1, le=20)
    DB_MAX_OVERFLOW: int = Field(default=10, ge=0, le=50)
    DB_POOL_TIMEOUT: int = Field(default=30, ge=1)
    DB_POOL_RECYCLE: int = Field(default=1800, ge=300)
    DB_ECHO: bool = False

    # Security
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    class Config:
        env_file = ".env"
        case_sensitive = True

@lru_cache
def get_settings() -> Settings:
    return Settings()

settings = get_settings()
```

### Error Handling

```python
# core/exceptions.py
from fastapi import HTTPException, status

class DatabaseException(Exception):
    """Base database exception."""
    pass

class RecordNotFoundError(DatabaseException):
    """Record not found in database."""
    pass

class DuplicateRecordError(DatabaseException):
    """Duplicate record constraint violation."""
    pass

# Exception handlers
from fastapi import Request
from fastapi.responses import JSONResponse

async def database_exception_handler(request: Request, exc: DatabaseException):
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={"detail": "Database error occurred"},
    )

async def record_not_found_handler(request: Request, exc: RecordNotFoundError):
    return JSONResponse(
        status_code=status.HTTP_404_NOT_FOUND,
        content={"detail": str(exc)},
    )

# Register handlers in main.py
app.add_exception_handler(DatabaseException, database_exception_handler)
app.add_exception_handler(RecordNotFoundError, record_not_found_handler)
```

### Security Best Practices

```python
# core/security.py
from passlib.context import CryptContext
from jose import jwt, JWTError
from datetime import datetime, timedelta
from typing import Optional

from core.config import settings

# Password hashing
pwd_context = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",
    bcrypt__rounds=12,
)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

# JWT tokens
def create_access_token(
    subject: str,
    expires_delta: Optional[timedelta] = None,
) -> str:
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        )

    to_encode = {"exp": expire, "sub": str(subject)}
    return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

def decode_token(token: str) -> Optional[str]:
    try:
        payload = jwt.decode(
            token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM]
        )
        return payload.get("sub")
    except JWTError:
        return None
```

### Performance Tips

```python
# 1. Use selectinload for collections to avoid N+1 queries
async def get_users_with_posts(db: AsyncSession):
    return await db.execute(
        select(User).options(selectinload(User.posts))
    )

# 2. Use only() to fetch specific columns
async def get_user_emails(db: AsyncSession):
    return await db.execute(
        select(User.id, User.email)
    )

# 3. Use pagination for large result sets
async def get_paginated_results(db: AsyncSession, page: int, per_page: int):
    offset = (page - 1) * per_page
    return await db.execute(
        select(Post)
        .order_by(Post.created_at.desc())
        .offset(offset)
        .limit(per_page)
    )

# 4. Use bulk operations for multiple inserts/updates
async def bulk_create_posts(db: AsyncSession, posts: list[dict]):
    await db.execute(insert(Post).values(posts))
    await db.commit()

# 5. Use indexes on frequently queried columns
class OptimizedModel(BaseModel):
    __table_args__ = (
        Index('ix_status_created', 'status', 'created_at'),
    )

# 6. Use connection pooling appropriately
# - Production: pool_size=5-10, max_overflow=10-20
# - Testing: NullPool
# - With PgBouncer: NullPool

# 7. Use read replicas for read-heavy workloads
read_engine = create_async_engine(READ_REPLICA_URL)
write_engine = create_async_engine(PRIMARY_URL)
```

### Logging Configuration

```python
# core/logging.py
import logging
import sys
from logging.handlers import RotatingFileHandler

def setup_logging(debug: bool = False):
    """Configure application logging."""

    # Root logger
    root_logger = logging.getLogger()
    root_logger.setLevel(logging.DEBUG if debug else logging.INFO)

    # Console handler
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(logging.DEBUG if debug else logging.INFO)
    console_formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    console_handler.setFormatter(console_formatter)
    root_logger.addHandler(console_handler)

    # SQLAlchemy logging
    sqlalchemy_logger = logging.getLogger('sqlalchemy.engine')
    sqlalchemy_logger.setLevel(logging.INFO if debug else logging.WARNING)

    # Alembic logging
    alembic_logger = logging.getLogger('alembic')
    alembic_logger.setLevel(logging.INFO)
```

### Docker Compose for Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/myapp
      - SECRET_KEY=your-secret-key
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

## Summary

This guide covered comprehensive database integration with FastAPI:

1. **SQLAlchemy Setup**: Sync and async configurations with proper engine settings
2. **Session Management**: Context managers, lifespan events, and dependency injection
3. **Model Definition**: Base models, mixins, validation, and relationships
4. **Pydantic Schemas**: Create, read, update patterns with validation
5. **CRUD Operations**: Generic base class and specific implementations
6. **Dependencies**: Database sessions combined with authentication
7. **Async Operations**: asyncpg, databases library, and bulk operations
8. **Relationships**: One-to-many, many-to-many, and self-referential
9. **Query Optimization**: Eager loading, indexing, and pagination strategies
10. **Alembic Migrations**: Async configuration and migration patterns
11. **Transactions**: Manual control, savepoints, and isolation levels
12. **PostgreSQL**: JSONB, arrays, full-text search, and specific types
13. **MongoDB**: Motor integration with Pydantic models
14. **Testing**: Test database setup, fixtures, and example tests
15. **Connection Pooling**: Configuration, monitoring, and PgBouncer
16. **Best Practices**: Project structure, security, performance, and DevOps

For more information, refer to:
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy Documentation](https://docs.sqlalchemy.org/)
- [Alembic Documentation](https://alembic.sqlalchemy.org/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
