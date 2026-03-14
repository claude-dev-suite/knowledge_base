# FastAPI Basics

A comprehensive guide to building APIs with FastAPI, the modern, fast (high-performance) web framework for building APIs with Python based on standard Python type hints.

---

## Table of Contents

1. [Application Setup](#1-application-setup)
2. [Path Operations](#2-path-operations-get-post-put-delete-patch)
3. [Path Parameters](#3-path-parameters)
4. [Query Parameters](#4-query-parameters)
5. [Request Body with Pydantic](#5-request-body-with-pydantic)
6. [Response Models](#6-response-models)
7. [Status Codes](#7-status-codes)
8. [Form Data and File Uploads](#8-form-data-and-file-uploads)
9. [Dependency Injection](#9-dependency-injection)
10. [Security (OAuth2, JWT)](#10-security-oauth2-jwt)
11. [Middleware](#11-middleware)
12. [Exception Handling](#12-exception-handling)
13. [Background Tasks](#13-background-tasks)
14. [CORS](#14-cors)
15. [Async/Await in FastAPI](#15-asyncawait-in-fastapi)
16. [APIRouter for Modular Apps](#16-apirouter-for-modular-apps)
17. [Testing FastAPI](#17-testing-fastapi)
18. [Best Practices](#18-best-practices)

---

## 1. Application Setup

### Basic Application

The simplest FastAPI application consists of creating an instance of the `FastAPI` class.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

### Application Configuration

FastAPI accepts various configuration parameters for customizing your API:

```python
from fastapi import FastAPI

app = FastAPI(
    title="My Awesome API",
    description="This is a very detailed API description with **markdown** support.",
    version="1.0.0",
    terms_of_service="https://example.com/terms/",
    contact={
        "name": "API Support",
        "url": "https://example.com/support",
        "email": "support@example.com",
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT",
    },
    openapi_url="/api/v1/openapi.json",
    docs_url="/docs",
    redoc_url="/redoc",
)
```

### Running with Uvicorn

FastAPI runs on ASGI servers. Uvicorn is the recommended server:

```bash
# Install uvicorn
pip install uvicorn[standard]

# Run the application
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Running Programmatically

```python
import uvicorn
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

if __name__ == "__main__":
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

### Application with Lifespan Events

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

# Simulated database connection
fake_db = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Load resources, establish connections
    fake_db["connection"] = "established"
    print("Application starting up...")
    yield
    # Shutdown: Clean up resources
    fake_db.clear()
    print("Application shutting down...")

app = FastAPI(lifespan=lifespan)

@app.get("/")
async def root():
    return {"db_status": fake_db.get("connection", "not connected")}
```

### OpenAPI Tags Metadata

```python
from fastapi import FastAPI

tags_metadata = [
    {
        "name": "users",
        "description": "Operations with users. The **login** logic is also here.",
    },
    {
        "name": "items",
        "description": "Manage items. So _fancy_ they have their own docs.",
        "externalDocs": {
            "description": "Items external docs",
            "url": "https://example.com/items/",
        },
    },
]

app = FastAPI(openapi_tags=tags_metadata)

@app.get("/users/", tags=["users"])
async def get_users():
    return [{"name": "Harry"}, {"name": "Ron"}]

@app.get("/items/", tags=["items"])
async def get_items():
    return [{"name": "wand"}, {"name": "flying broom"}]
```

---

## 2. Path Operations (GET, POST, PUT, DELETE, PATCH)

### GET - Retrieve Data

```python
from fastapi import FastAPI

app = FastAPI()

# Simple GET
@app.get("/")
async def root():
    return {"message": "Hello World"}

# GET with path parameter
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}

# GET with query parameters
@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}
```

### POST - Create Data

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.post("/items/")
async def create_item(item: Item):
    item_dict = item.model_dump()
    if item.tax:
        price_with_tax = item.price + item.tax
        item_dict.update({"price_with_tax": price_with_tax})
    return item_dict

# POST with path and body
@app.post("/items/{item_id}")
async def create_item_with_id(item_id: int, item: Item):
    return {"item_id": item_id, **item.model_dump()}
```

### PUT - Full Update

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float = 10.5

items = {}

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item):
    items[item_id] = item
    return {"item_id": item_id, **item.model_dump()}

# PUT with query parameter
@app.put("/items/{item_id}")
async def update_item_with_query(item_id: int, item: Item, q: str | None = None):
    result = {"item_id": item_id, **item.model_dump()}
    if q:
        result.update({"q": q})
    return result
```

### PATCH - Partial Update

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float = 10.5

class ItemUpdate(BaseModel):
    name: str | None = None
    description: str | None = None
    price: float | None = None
    tax: float | None = None

items = {
    1: Item(name="Foo", price=50.2),
    2: Item(name="Bar", description="The Bar fighters", price=62.0, tax=20.2),
}

@app.patch("/items/{item_id}")
async def partial_update_item(item_id: int, item: ItemUpdate):
    stored_item = items.get(item_id)
    if stored_item is None:
        return {"error": "Item not found"}

    # Get stored data
    stored_item_data = stored_item.model_dump()

    # Update only provided fields
    update_data = item.model_dump(exclude_unset=True)
    updated_item = stored_item.model_copy(update=update_data)

    items[item_id] = updated_item
    return updated_item
```

### DELETE - Remove Data

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()

items = {1: {"name": "Foo"}, 2: {"name": "Bar"}}

@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
    del items[item_id]
    return None

# DELETE with response
@app.delete("/items/{item_id}/with-response")
async def delete_item_with_response(item_id: int):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
    deleted_item = items.pop(item_id)
    return {"deleted": True, "item": deleted_item}
```

### Multiple HTTP Methods on Same Path

```python
from fastapi import FastAPI

app = FastAPI()

@app.api_route("/items/", methods=["GET", "POST"])
async def handle_items():
    return {"message": "Handles both GET and POST"}
```

### Path Operation Configuration

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.post(
    "/items/",
    response_model=Item,
    status_code=status.HTTP_201_CREATED,
    tags=["items"],
    summary="Create an item",
    description="Create an item with all the information: name, description, price, tax.",
    response_description="The created item",
    deprecated=False,
)
async def create_item(item: Item):
    return item
```

---

## 3. Path Parameters

### Basic Path Parameters

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}

# String path parameter
@app.get("/users/{user_name}")
async def read_user(user_name: str):
    return {"user_name": user_name}
```

### Path Parameters with Validation

```python
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(
    item_id: int = Path(
        ...,
        title="The ID of the item to get",
        description="A unique identifier for the item",
        ge=1,
        le=1000,
        example=42,
    )
):
    return {"item_id": item_id}

# String validation
@app.get("/users/{username}")
async def read_user(
    username: str = Path(
        ...,
        min_length=3,
        max_length=50,
        pattern="^[a-zA-Z0-9_]+$",
    )
):
    return {"username": username}
```

### Predefined Path Parameter Values (Enum)

```python
from enum import Enum
from fastapi import FastAPI

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

app = FastAPI()

@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    if model_name is ModelName.alexnet:
        return {"model_name": model_name, "message": "Deep Learning FTW!"}
    if model_name.value == "lenet":
        return {"model_name": model_name, "message": "LeCNN all the images"}
    return {"model_name": model_name, "message": "Have some residuals"}
```

### Path Parameters with File Paths

```python
from fastapi import FastAPI

app = FastAPI()

# Use :path converter for file paths
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    return {"file_path": file_path}

# Example: /files/home/user/myfile.txt
# file_path will be "home/user/myfile.txt"
```

### Order of Path Operations

```python
from fastapi import FastAPI

app = FastAPI()

# Fixed path must come before parameterized path
@app.get("/users/me")
async def read_user_me():
    return {"user_id": "the current user"}

@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}
```

### Multiple Path Parameters

```python
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/users/{user_id}/items/{item_id}")
async def read_user_item(
    user_id: int = Path(..., ge=1),
    item_id: int = Path(..., ge=1),
    q: str | None = None,
):
    result = {"user_id": user_id, "item_id": item_id}
    if q:
        result.update({"q": q})
    return result
```

---

## 4. Query Parameters

### Basic Query Parameters

```python
from fastapi import FastAPI

app = FastAPI()

fake_items_db = [{"item_name": "Foo"}, {"item_name": "Bar"}, {"item_name": "Baz"}]

@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10):
    return fake_items_db[skip : skip + limit]
```

### Optional Query Parameters

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: str, q: str | None = None):
    if q:
        return {"item_id": item_id, "q": q}
    return {"item_id": item_id}

# Multiple optional parameters
@app.get("/items/{item_id}")
async def read_item_detailed(
    item_id: str,
    q: str | None = None,
    short: bool = False,
):
    item = {"item_id": item_id}
    if q:
        item.update({"q": q})
    if not short:
        item.update({"description": "This is an amazing item that has a long description"})
    return item
```

### Required Query Parameters

```python
from fastapi import FastAPI

app = FastAPI()

# Required query parameter (no default value)
@app.get("/items/{item_id}")
async def read_item(item_id: str, needy: str):
    return {"item_id": item_id, "needy": needy}
```

### Query Parameters with Validation

```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
async def read_items(
    q: str | None = Query(
        default=None,
        min_length=3,
        max_length=50,
        pattern="^[a-zA-Z]+$",
        title="Query string",
        description="Query string for the items to search in the database",
        alias="item-query",
        deprecated=False,
        include_in_schema=True,
    )
):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```

### Required Query Parameters with Validation

```python
from fastapi import FastAPI, Query

app = FastAPI()

# Using ellipsis to mark as required
@app.get("/items/")
async def read_items(q: str = Query(..., min_length=3)):
    return {"q": q}

# Using Required sentinel (equivalent to ...)
from pydantic import Required

@app.get("/items/required/")
async def read_items_required(q: str = Query(default=Required, min_length=3)):
    return {"q": q}
```

### Query Parameter List / Multiple Values

```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
async def read_items(q: list[str] | None = Query(default=None)):
    return {"q": q}

# With default values
@app.get("/items/default/")
async def read_items_default(q: list[str] = Query(default=["foo", "bar"])):
    return {"q": q}

# Example URL: /items/?q=foo&q=bar
```

### Numeric Validations

```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
async def read_items(
    size: int = Query(default=10, ge=1, le=100),
    price: float = Query(default=0.0, gt=0, lt=10000.0),
):
    return {"size": size, "price": price}
```

---

## 5. Request Body with Pydantic

### Basic Request Body

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.post("/items/")
async def create_item(item: Item):
    return item
```

### Field Validation

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI()

class Item(BaseModel):
    name: str = Field(
        ...,
        min_length=1,
        max_length=100,
        title="Item Name",
        description="The name of the item",
        examples=["Widget"],
    )
    description: str | None = Field(
        default=None,
        max_length=500,
        title="Description",
        description="Optional description of the item",
    )
    price: float = Field(
        ...,
        gt=0,
        description="Price must be greater than zero",
        examples=[29.99],
    )
    tax: float | None = Field(
        default=None,
        ge=0,
        le=100,
        description="Tax percentage",
    )

@app.post("/items/")
async def create_item(item: Item):
    return item
```

### Nested Models

```python
from fastapi import FastAPI
from pydantic import BaseModel, HttpUrl

app = FastAPI()

class Image(BaseModel):
    url: HttpUrl
    name: str

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None
    tags: set[str] = set()
    images: list[Image] | None = None

class Offer(BaseModel):
    name: str
    description: str | None = None
    price: float
    items: list[Item]

@app.post("/offers/")
async def create_offer(offer: Offer):
    return offer
```

### Model Configuration

```python
from fastapi import FastAPI
from pydantic import BaseModel, ConfigDict

app = FastAPI()

class Item(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,
        str_min_length=1,
        json_schema_extra={
            "examples": [
                {
                    "name": "Foo",
                    "description": "A very nice Item",
                    "price": 35.4,
                    "tax": 3.2,
                }
            ]
        },
    )

    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.post("/items/")
async def create_item(item: Item):
    return item
```

### Multiple Body Parameters

```python
from fastapi import FastAPI, Body
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

class User(BaseModel):
    username: str
    full_name: str | None = None

@app.put("/items/{item_id}")
async def update_item(
    item_id: int,
    item: Item,
    user: User,
    importance: int = Body(..., ge=1, le=10),
):
    return {"item_id": item_id, "item": item, "user": user, "importance": importance}
```

### Embedding Single Body Parameter

```python
from fastapi import FastAPI, Body
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

# Without embed: {"name": "...", "price": ...}
# With embed: {"item": {"name": "...", "price": ...}}
@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item = Body(..., embed=True)):
    return {"item_id": item_id, "item": item}
```

### Custom Validators

```python
from fastapi import FastAPI
from pydantic import BaseModel, field_validator, model_validator

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float
    discount: float | None = None

    @field_validator("name")
    @classmethod
    def name_must_not_be_empty(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Name cannot be empty or whitespace")
        return v.strip().title()

    @field_validator("price")
    @classmethod
    def price_must_be_positive(cls, v: float) -> float:
        if v <= 0:
            raise ValueError("Price must be positive")
        return round(v, 2)

    @model_validator(mode="after")
    def check_discount_less_than_price(self):
        if self.discount is not None and self.discount >= self.price:
            raise ValueError("Discount must be less than price")
        return self

@app.post("/items/")
async def create_item(item: Item):
    return item
```

---

## 6. Response Models

### Basic Response Model

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ItemIn(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

class ItemOut(BaseModel):
    name: str
    price: float
    description: str | None = None

@app.post("/items/", response_model=ItemOut)
async def create_item(item: ItemIn):
    return item
```

### Response Model with Password Exclusion

```python
from fastapi import FastAPI
from pydantic import BaseModel, EmailStr

app = FastAPI()

class UserIn(BaseModel):
    username: str
    password: str
    email: EmailStr
    full_name: str | None = None

class UserOut(BaseModel):
    username: str
    email: EmailStr
    full_name: str | None = None

@app.post("/user/", response_model=UserOut)
async def create_user(user: UserIn):
    return user  # Password will be excluded automatically
```

### Response Model Options

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float = 10.5

items = {
    "foo": {"name": "Foo", "price": 50.2},
    "bar": {"name": "Bar", "description": "The Bar fighters", "price": 62.0, "tax": 20.2},
}

# Exclude unset values
@app.get("/items/{item_id}", response_model=Item, response_model_exclude_unset=True)
async def read_item(item_id: str):
    return items[item_id]

# Exclude defaults
@app.get("/items/{item_id}/defaults", response_model=Item, response_model_exclude_defaults=True)
async def read_item_defaults(item_id: str):
    return items[item_id]

# Exclude None values
@app.get("/items/{item_id}/none", response_model=Item, response_model_exclude_none=True)
async def read_item_none(item_id: str):
    return items[item_id]
```

### Include and Exclude Specific Fields

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float = 10.5

# Include only specific fields
@app.get("/items/{item_id}/include", response_model=Item, response_model_include={"name", "price"})
async def read_item_include(item_id: str):
    return {"name": "Foo", "description": "Some description", "price": 50.2, "tax": 10.5}

# Exclude specific fields
@app.get("/items/{item_id}/exclude", response_model=Item, response_model_exclude={"tax"})
async def read_item_exclude(item_id: str):
    return {"name": "Foo", "description": "Some description", "price": 50.2, "tax": 10.5}
```

### Union Response Types

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class PlaneItem(BaseModel):
    name: str
    type: str = "plane"
    size: int

class CarItem(BaseModel):
    name: str
    type: str = "car"
    color: str

items = {
    "plane1": PlaneItem(name="Boeing", size=200),
    "car1": CarItem(name="Tesla", color="red"),
}

@app.get("/items/{item_id}", response_model=PlaneItem | CarItem)
async def read_item(item_id: str):
    return items[item_id]
```

### List Response

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.get("/items/", response_model=list[Item])
async def read_items():
    return [
        {"name": "Foo", "price": 50.2},
        {"name": "Bar", "price": 62.0},
    ]
```

---

## 7. Status Codes

### Using Status Code Constants

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item(name: str):
    return {"name": name}

@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int):
    return None

@app.get("/items/{item_id}", status_code=status.HTTP_200_OK)
async def read_item(item_id: int):
    return {"item_id": item_id}
```

### Common Status Codes

```python
from fastapi import FastAPI, status

app = FastAPI()

# 200 OK - Default for GET
@app.get("/")
async def root():
    return {"message": "Hello"}

# 201 Created - For POST creating resources
@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item():
    return {"created": True}

# 202 Accepted - For async processing
@app.post("/tasks/", status_code=status.HTTP_202_ACCEPTED)
async def create_task():
    return {"task": "queued"}

# 204 No Content - For DELETE operations
@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int):
    return None

# 301 Moved Permanently
from fastapi.responses import RedirectResponse

@app.get("/old-path")
async def redirect():
    return RedirectResponse(url="/new-path", status_code=status.HTTP_301_MOVED_PERMANENTLY)
```

### Dynamic Status Codes with Response

```python
from fastapi import FastAPI, Response, status

app = FastAPI()

tasks = {}

@app.put("/tasks/{task_id}")
async def update_task(task_id: str, response: Response):
    if task_id not in tasks:
        tasks[task_id] = {"id": task_id}
        response.status_code = status.HTTP_201_CREATED
        return tasks[task_id]

    # Update existing task
    response.status_code = status.HTTP_200_OK
    return tasks[task_id]
```

---

## 8. Form Data and File Uploads

### Form Data

```python
from fastapi import FastAPI, Form

app = FastAPI()

@app.post("/login/")
async def login(username: str = Form(...), password: str = Form(...)):
    return {"username": username}

# Form with validation
@app.post("/login/validated/")
async def login_validated(
    username: str = Form(..., min_length=3, max_length=50),
    password: str = Form(..., min_length=8),
):
    return {"username": username}
```

### File Upload

```python
from fastapi import FastAPI, File, UploadFile

app = FastAPI()

# Simple file upload (bytes)
@app.post("/files/")
async def create_file(file: bytes = File(...)):
    return {"file_size": len(file)}

# UploadFile (recommended for large files)
@app.post("/uploadfile/")
async def create_upload_file(file: UploadFile):
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": file.size,
    }
```

### File Upload with Validation

```python
from fastapi import FastAPI, File, UploadFile, HTTPException

app = FastAPI()

ALLOWED_EXTENSIONS = {"png", "jpg", "jpeg", "gif"}
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5MB

@app.post("/upload/")
async def upload_file(file: UploadFile = File(...)):
    # Check file extension
    file_ext = file.filename.split(".")[-1].lower()
    if file_ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(
            status_code=400,
            detail=f"File extension not allowed. Allowed: {ALLOWED_EXTENSIONS}"
        )

    # Check file size
    contents = await file.read()
    if len(contents) > MAX_FILE_SIZE:
        raise HTTPException(status_code=400, detail="File too large")

    # Reset file position for further processing
    await file.seek(0)

    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
    }
```

### Multiple File Uploads

```python
from fastapi import FastAPI, File, UploadFile

app = FastAPI()

@app.post("/files/")
async def create_files(files: list[bytes] = File(...)):
    return {"file_sizes": [len(file) for file in files]}

@app.post("/uploadfiles/")
async def create_upload_files(files: list[UploadFile]):
    return {"filenames": [file.filename for file in files]}
```

### Form Data with File Upload

```python
from fastapi import FastAPI, File, Form, UploadFile

app = FastAPI()

@app.post("/files/")
async def create_file(
    file: UploadFile = File(...),
    fileb: UploadFile = File(...),
    token: str = Form(...),
    notes: str = Form(default=None),
):
    return {
        "file_size": file.size,
        "fileb_content_type": fileb.content_type,
        "token": token,
        "notes": notes,
    }
```

### Saving Uploaded Files

```python
from fastapi import FastAPI, File, UploadFile
import shutil
from pathlib import Path

app = FastAPI()

UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)

@app.post("/upload/")
async def upload_file(file: UploadFile = File(...)):
    file_path = UPLOAD_DIR / file.filename

    with file_path.open("wb") as buffer:
        shutil.copyfileobj(file.file, buffer)

    return {"filename": file.filename, "path": str(file_path)}

# Async version for large files
@app.post("/upload/async/")
async def upload_file_async(file: UploadFile = File(...)):
    file_path = UPLOAD_DIR / file.filename

    contents = await file.read()
    with file_path.open("wb") as f:
        f.write(contents)

    return {"filename": file.filename, "path": str(file_path)}
```

---

## 9. Dependency Injection

### Basic Dependencies

```python
from fastapi import FastAPI, Depends

app = FastAPI()

async def common_parameters(q: str | None = None, skip: int = 0, limit: int = 100):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items/")
async def read_items(commons: dict = Depends(common_parameters)):
    return commons

@app.get("/users/")
async def read_users(commons: dict = Depends(common_parameters)):
    return commons
```

### Class-based Dependencies

```python
from fastapi import FastAPI, Depends

app = FastAPI()

class CommonQueryParams:
    def __init__(self, q: str | None = None, skip: int = 0, limit: int = 100):
        self.q = q
        self.skip = skip
        self.limit = limit

@app.get("/items/")
async def read_items(commons: CommonQueryParams = Depends()):
    return {"q": commons.q, "skip": commons.skip, "limit": commons.limit}
```

### Sub-dependencies

```python
from fastapi import FastAPI, Depends, Cookie

app = FastAPI()

def query_extractor(q: str | None = None):
    return q

def query_or_cookie_extractor(
    q: str = Depends(query_extractor),
    last_query: str | None = Cookie(default=None),
):
    if not q:
        return last_query
    return q

@app.get("/items/")
async def read_query(query_or_default: str = Depends(query_or_cookie_extractor)):
    return {"q_or_cookie": query_or_default}
```

### Dependencies in Path Operation Decorators

```python
from fastapi import FastAPI, Depends, Header, HTTPException

app = FastAPI()

async def verify_token(x_token: str = Header(...)):
    if x_token != "fake-super-secret-token":
        raise HTTPException(status_code=400, detail="X-Token header invalid")

async def verify_key(x_key: str = Header(...)):
    if x_key != "fake-super-secret-key":
        raise HTTPException(status_code=400, detail="X-Key header invalid")
    return x_key

@app.get("/items/", dependencies=[Depends(verify_token), Depends(verify_key)])
async def read_items():
    return [{"item": "Foo"}, {"item": "Bar"}]
```

### Global Dependencies

```python
from fastapi import FastAPI, Depends, Header, HTTPException

async def verify_token(x_token: str = Header(...)):
    if x_token != "fake-super-secret-token":
        raise HTTPException(status_code=400, detail="X-Token header invalid")

app = FastAPI(dependencies=[Depends(verify_token)])

@app.get("/items/")
async def read_items():
    return [{"item": "Portal Gun"}, {"item": "Plumbus"}]

@app.get("/users/")
async def read_users():
    return [{"username": "Rick"}, {"username": "Morty"}]
```

### Dependencies with Yield (Context Manager)

```python
from fastapi import FastAPI, Depends

app = FastAPI()

class DatabaseSession:
    def __init__(self):
        self.connected = False

    def connect(self):
        self.connected = True

    def disconnect(self):
        self.connected = False

async def get_db():
    db = DatabaseSession()
    db.connect()
    try:
        yield db
    finally:
        db.disconnect()

@app.get("/items/")
async def read_items(db: DatabaseSession = Depends(get_db)):
    return {"db_connected": db.connected}
```

### Dependency with Exception Handling

```python
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

class DBSession:
    def __init__(self):
        self.data = {}

async def get_db_session():
    db = DBSession()
    try:
        yield db
    except Exception:
        # Handle any exceptions during request
        raise
    finally:
        # Cleanup always runs
        pass

@app.get("/items/{item_id}")
async def read_item(item_id: str, db: DBSession = Depends(get_db_session)):
    if item_id not in db.data:
        raise HTTPException(status_code=404, detail="Item not found")
    return db.data[item_id]
```

---

## 10. Security (OAuth2, JWT)

### Basic OAuth2 with Password Flow

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel

app = FastAPI()

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

fake_users_db = {
    "johndoe": {
        "username": "johndoe",
        "full_name": "John Doe",
        "email": "johndoe@example.com",
        "hashed_password": "fakehashedsecret",
        "disabled": False,
    }
}

class User(BaseModel):
    username: str
    email: str | None = None
    full_name: str | None = None
    disabled: bool | None = None

def fake_hash_password(password: str):
    return "fakehashed" + password

def get_user(db, username: str):
    if username in db:
        user_dict = db[username]
        return User(**user_dict)

def fake_decode_token(token):
    return get_user(fake_users_db, token)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    user = fake_decode_token(token)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return user

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user_dict = fake_users_db.get(form_data.username)
    if not user_dict:
        raise HTTPException(status_code=400, detail="Incorrect username or password")

    hashed_password = fake_hash_password(form_data.password)
    if not hashed_password == user_dict["hashed_password"]:
        raise HTTPException(status_code=400, detail="Incorrect username or password")

    return {"access_token": form_data.username, "token_type": "bearer"}

@app.get("/users/me")
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user
```

### JWT Token Authentication

```python
from datetime import datetime, timedelta
from typing import Annotated

from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel

# Configuration
SECRET_KEY = "your-secret-key-keep-it-secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

app = FastAPI()

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Models
class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: str | None = None

class User(BaseModel):
    username: str
    email: str | None = None
    full_name: str | None = None
    disabled: bool | None = None

class UserInDB(User):
    hashed_password: str

# Fake database
fake_users_db = {
    "johndoe": {
        "username": "johndoe",
        "full_name": "John Doe",
        "email": "johndoe@example.com",
        "hashed_password": "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW",
        "disabled": False,
    }
}

# Utility functions
def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

def get_user(db: dict, username: str) -> UserInDB | None:
    if username in db:
        user_dict = db[username]
        return UserInDB(**user_dict)

def authenticate_user(fake_db: dict, username: str, password: str) -> UserInDB | bool:
    user = get_user(fake_db, username)
    if not user:
        return False
    if not verify_password(password, user.hashed_password):
        return False
    return user

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username)
    except JWTError:
        raise credentials_exception
    user = get_user(fake_users_db, username=token_data.username)
    if user is None:
        raise credentials_exception
    return user

async def get_current_active_user(
    current_user: Annotated[User, Depends(get_current_user)]
) -> User:
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

# Endpoints
@app.post("/token", response_model=Token)
async def login_for_access_token(
    form_data: Annotated[OAuth2PasswordRequestForm, Depends()]
) -> Token:
    user = authenticate_user(fake_users_db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username}, expires_delta=access_token_expires
    )
    return Token(access_token=access_token, token_type="bearer")

@app.get("/users/me", response_model=User)
async def read_users_me(
    current_user: Annotated[User, Depends(get_current_active_user)]
) -> User:
    return current_user
```

### API Key Authentication

```python
from fastapi import FastAPI, Security, HTTPException, status
from fastapi.security import APIKeyHeader, APIKeyQuery

app = FastAPI()

API_KEY = "your-api-key"
API_KEY_NAME = "X-API-Key"

api_key_header = APIKeyHeader(name=API_KEY_NAME, auto_error=False)
api_key_query = APIKeyQuery(name="api_key", auto_error=False)

async def get_api_key(
    api_key_header: str = Security(api_key_header),
    api_key_query: str = Security(api_key_query),
):
    if api_key_header == API_KEY:
        return api_key_header
    elif api_key_query == API_KEY:
        return api_key_query
    else:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Could not validate credentials",
        )

@app.get("/protected/")
async def protected_route(api_key: str = Security(get_api_key)):
    return {"message": "Access granted", "api_key": api_key}
```

---

## 11. Middleware

### Basic Middleware

```python
import time
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

### Class-based Middleware

```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response

app = FastAPI()

class CustomMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        # Before request
        print(f"Request: {request.method} {request.url}")

        response = await call_next(request)

        # After request
        response.headers["X-Custom-Header"] = "Custom Value"
        return response

app.add_middleware(CustomMiddleware)

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

### Logging Middleware

```python
import logging
import time
from fastapi import FastAPI, Request

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI()

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()

    # Log request
    logger.info(f"Request: {request.method} {request.url}")
    logger.info(f"Headers: {dict(request.headers)}")

    response = await call_next(request)

    # Log response
    process_time = time.time() - start_time
    logger.info(f"Response status: {response.status_code}")
    logger.info(f"Process time: {process_time:.4f}s")

    return response
```

### Authentication Middleware

```python
from fastapi import FastAPI, Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

class AuthMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, excluded_paths: list[str] = None):
        super().__init__(app)
        self.excluded_paths = excluded_paths or []

    async def dispatch(self, request: Request, call_next):
        # Skip authentication for excluded paths
        if request.url.path in self.excluded_paths:
            return await call_next(request)

        # Check for authorization header
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            raise HTTPException(status_code=401, detail="Missing or invalid token")

        # Validate token (simplified)
        token = auth_header.split(" ")[1]
        if token != "valid-token":
            raise HTTPException(status_code=401, detail="Invalid token")

        return await call_next(request)

app.add_middleware(AuthMiddleware, excluded_paths=["/", "/health", "/docs", "/openapi.json"])
```

### Trusted Host Middleware

```python
from fastapi import FastAPI
from starlette.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["example.com", "*.example.com", "localhost"],
)
```

### GZip Middleware

```python
from fastapi import FastAPI
from starlette.middleware.gzip import GZipMiddleware

app = FastAPI()

app.add_middleware(GZipMiddleware, minimum_size=1000)

@app.get("/large-response")
async def large_response():
    return {"data": "x" * 10000}
```

---

## 12. Exception Handling

### HTTPException

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

items = {"foo": "The Foo Wrestlers"}

@app.get("/items/{item_id}")
async def read_item(item_id: str):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"item": items[item_id]}

# With custom headers
@app.get("/items-header/{item_id}")
async def read_item_header(item_id: str):
    if item_id not in items:
        raise HTTPException(
            status_code=404,
            detail="Item not found",
            headers={"X-Error": "There goes my error"},
        )
    return {"item": items[item_id]}
```

### Custom Exception Classes

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

class ItemNotFoundException(Exception):
    def __init__(self, item_id: str):
        self.item_id = item_id

class InsufficientPermissionsException(Exception):
    def __init__(self, required_permission: str):
        self.required_permission = required_permission

@app.exception_handler(ItemNotFoundException)
async def item_not_found_exception_handler(request: Request, exc: ItemNotFoundException):
    return JSONResponse(
        status_code=404,
        content={
            "error": "item_not_found",
            "message": f"Item with ID '{exc.item_id}' was not found",
            "item_id": exc.item_id,
        },
    )

@app.exception_handler(InsufficientPermissionsException)
async def insufficient_permissions_exception_handler(
    request: Request, exc: InsufficientPermissionsException
):
    return JSONResponse(
        status_code=403,
        content={
            "error": "insufficient_permissions",
            "message": f"You need '{exc.required_permission}' permission to perform this action",
            "required_permission": exc.required_permission,
        },
    )

items = {}

@app.get("/items/{item_id}")
async def read_item(item_id: str):
    if item_id not in items:
        raise ItemNotFoundException(item_id=item_id)
    return {"item": items[item_id]}
```

### Override Default Exception Handlers

```python
from fastapi import FastAPI, Request, status
from fastapi.encoders import jsonable_encoder
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

app = FastAPI()

@app.exception_handler(StarletteHTTPException)
async def http_exception_handler(request: Request, exc: StarletteHTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": True,
            "message": exc.detail,
            "status_code": exc.status_code,
        },
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "error": True,
            "message": "Validation error",
            "details": jsonable_encoder(exc.errors()),
            "body": exc.body,
        },
    )
```

### Global Exception Handler

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import traceback
import logging

logging.basicConfig(level=logging.ERROR)
logger = logging.getLogger(__name__)

app = FastAPI()

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    # Log the error
    logger.error(f"Unhandled exception: {exc}")
    logger.error(traceback.format_exc())

    return JSONResponse(
        status_code=500,
        content={
            "error": True,
            "message": "An internal server error occurred",
            "type": type(exc).__name__,
        },
    )
```

---

## 13. Background Tasks

### Basic Background Tasks

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def write_notification(email: str, message: str = ""):
    with open("log.txt", mode="a") as log_file:
        content = f"notification for {email}: {message}\n"
        log_file.write(content)

@app.post("/send-notification/{email}")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(write_notification, email, message="some notification")
    return {"message": "Notification sent in the background"}
```

### Background Tasks with Dependencies

```python
from fastapi import FastAPI, BackgroundTasks, Depends

app = FastAPI()

def write_log(message: str):
    with open("log.txt", mode="a") as log:
        log.write(message + "\n")

def get_query(background_tasks: BackgroundTasks, q: str | None = None):
    if q:
        message = f"found query: {q}"
        background_tasks.add_task(write_log, message)
    return q

@app.post("/send-notification/{email}")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks,
    q: str = Depends(get_query),
):
    message = f"message to {email}"
    background_tasks.add_task(write_log, message)
    return {"message": "Message sent", "query": q}
```

### Async Background Tasks

```python
import asyncio
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

async def send_email_async(email: str, subject: str, body: str):
    # Simulate async email sending
    await asyncio.sleep(2)
    print(f"Email sent to {email}: {subject}")

def send_email_sync(email: str, subject: str, body: str):
    # Sync function also works
    import time
    time.sleep(2)
    print(f"Email sent to {email}: {subject}")

@app.post("/send-email/")
async def send_email(
    email: str,
    subject: str,
    body: str,
    background_tasks: BackgroundTasks,
):
    # Both async and sync functions work
    background_tasks.add_task(send_email_async, email, subject, body)
    return {"message": "Email will be sent in background"}
```

### Multiple Background Tasks

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def task_one(data: str):
    print(f"Task one processing: {data}")

def task_two(data: str):
    print(f"Task two processing: {data}")

def task_three(data: str):
    print(f"Task three processing: {data}")

@app.post("/process/")
async def process_data(data: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(task_one, data)
    background_tasks.add_task(task_two, data)
    background_tasks.add_task(task_three, data)
    return {"message": "Processing started in background"}
```

---

## 14. CORS

### Basic CORS Configuration

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

origins = [
    "http://localhost",
    "http://localhost:3000",
    "http://localhost:8080",
    "https://myapp.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/")
async def main():
    return {"message": "Hello World"}
```

### CORS with Specific Methods and Headers

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://frontend.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Custom-Header"],
    expose_headers=["X-Custom-Response-Header"],
    max_age=600,  # Cache preflight response for 10 minutes
)
```

### CORS Allow All (Development Only)

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# WARNING: Only use this in development!
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,  # Must be False when allow_origins=["*"]
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### CORS with Regex Pattern

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origin_regex=r"https://.*\.example\.com",
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Environment-based CORS

```python
import os
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Get allowed origins from environment
ALLOWED_ORIGINS = os.getenv("ALLOWED_ORIGINS", "http://localhost:3000").split(",")
ENVIRONMENT = os.getenv("ENVIRONMENT", "development")

if ENVIRONMENT == "development":
    origins = ["*"]
    allow_credentials = False
else:
    origins = ALLOWED_ORIGINS
    allow_credentials = True

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=allow_credentials,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["*"],
)
```

---

## 15. Async/Await in FastAPI

### Async Path Operations

```python
from fastapi import FastAPI

app = FastAPI()

# Async function
@app.get("/async")
async def read_async():
    return {"message": "This is async"}

# Sync function (also valid)
@app.get("/sync")
def read_sync():
    return {"message": "This is sync"}
```

### When to Use Async

```python
import asyncio
import httpx
from fastapi import FastAPI

app = FastAPI()

# Use async for I/O bound operations
@app.get("/fetch-data")
async def fetch_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()

# Use async for database operations with async drivers
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # Example with async database
    # user = await database.fetch_one(query, values={"id": user_id})
    await asyncio.sleep(0.1)  # Simulating async database call
    return {"user_id": user_id}
```

### Concurrent Requests

```python
import asyncio
import httpx
from fastapi import FastAPI

app = FastAPI()

async def fetch_url(client: httpx.AsyncClient, url: str):
    response = await client.get(url)
    return response.json()

@app.get("/fetch-multiple")
async def fetch_multiple():
    urls = [
        "https://api.example.com/data1",
        "https://api.example.com/data2",
        "https://api.example.com/data3",
    ]

    async with httpx.AsyncClient() as client:
        tasks = [fetch_url(client, url) for url in urls]
        results = await asyncio.gather(*tasks)

    return {"results": results}
```

### Async Context Managers

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends

# Simulated async database
class AsyncDatabase:
    async def connect(self):
        print("Connecting to database...")

    async def disconnect(self):
        print("Disconnecting from database...")

    async def execute(self, query: str):
        return {"query": query, "result": "success"}

db = AsyncDatabase()

@asynccontextmanager
async def lifespan(app: FastAPI):
    await db.connect()
    yield
    await db.disconnect()

app = FastAPI(lifespan=lifespan)

async def get_db():
    return db

@app.get("/query")
async def execute_query(database: AsyncDatabase = Depends(get_db)):
    result = await database.execute("SELECT * FROM users")
    return result
```

### Mixing Sync and Async

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
from fastapi import FastAPI

app = FastAPI()

# CPU-bound sync function
def cpu_intensive_task(n: int) -> int:
    result = 0
    for i in range(n):
        result += i * i
    return result

# Run sync function in thread pool
@app.get("/compute/{n}")
async def compute(n: int):
    loop = asyncio.get_event_loop()
    with ThreadPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_intensive_task, n)
    return {"result": result}
```

---

## 16. APIRouter for Modular Apps

### Basic Router Setup

```python
# routers/items.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/")
async def read_items():
    return [{"item_id": "Foo"}, {"item_id": "Bar"}]

@router.get("/{item_id}")
async def read_item(item_id: str):
    return {"item_id": item_id}

@router.post("/")
async def create_item(item: dict):
    return item
```

```python
# routers/users.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/")
async def read_users():
    return [{"username": "Rick"}, {"username": "Morty"}]

@router.get("/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}
```

```python
# main.py
from fastapi import FastAPI
from routers import items, users

app = FastAPI()

app.include_router(items.router, prefix="/items", tags=["items"])
app.include_router(users.router, prefix="/users", tags=["users"])

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

### Router with Dependencies

```python
# routers/items.py
from fastapi import APIRouter, Depends, Header, HTTPException

async def verify_token(x_token: str = Header(...)):
    if x_token != "fake-super-secret-token":
        raise HTTPException(status_code=400, detail="X-Token header invalid")

async def verify_key(x_key: str = Header(...)):
    if x_key != "fake-super-secret-key":
        raise HTTPException(status_code=400, detail="X-Key header invalid")

router = APIRouter(
    prefix="/items",
    tags=["items"],
    dependencies=[Depends(verify_token)],
    responses={404: {"description": "Not found"}},
)

@router.get("/")
async def read_items():
    return [{"item_id": "Foo"}, {"item_id": "Bar"}]

@router.get("/{item_id}")
async def read_item(item_id: str):
    return {"item_id": item_id}

# Additional dependency for specific route
@router.put("/{item_id}", dependencies=[Depends(verify_key)])
async def update_item(item_id: str):
    return {"item_id": item_id, "status": "updated"}
```

### Nested Routers

```python
# routers/api_v1/endpoints/users.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/")
async def get_users():
    return [{"id": 1, "name": "User 1"}]
```

```python
# routers/api_v1/endpoints/items.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/")
async def get_items():
    return [{"id": 1, "name": "Item 1"}]
```

```python
# routers/api_v1/api.py
from fastapi import APIRouter
from routers.api_v1.endpoints import users, items

api_router = APIRouter()
api_router.include_router(users.router, prefix="/users", tags=["users"])
api_router.include_router(items.router, prefix="/items", tags=["items"])
```

```python
# main.py
from fastapi import FastAPI
from routers.api_v1.api import api_router

app = FastAPI()
app.include_router(api_router, prefix="/api/v1")
```

### Project Structure Example

```
project/
├── main.py
├── core/
│   ├── __init__.py
│   ├── config.py
│   └── security.py
├── models/
│   ├── __init__.py
│   ├── user.py
│   └── item.py
├── schemas/
│   ├── __init__.py
│   ├── user.py
│   └── item.py
├── routers/
│   ├── __init__.py
│   ├── users.py
│   └── items.py
├── dependencies/
│   ├── __init__.py
│   └── auth.py
└── services/
    ├── __init__.py
    ├── user_service.py
    └── item_service.py
```

---

## 17. Testing FastAPI

### Basic Testing Setup

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

def test_read_item():
    response = client.get("/items/foo")
    assert response.status_code == 200
    assert response.json() == {"item_id": "foo"}

def test_create_item():
    response = client.post(
        "/items/",
        json={"name": "Foo", "price": 50.5},
    )
    assert response.status_code == 201
    assert response.json()["name"] == "Foo"
```

### Testing with Pytest

```python
# conftest.py
import pytest
from fastapi.testclient import TestClient
from main import app

@pytest.fixture
def client():
    return TestClient(app)

@pytest.fixture
def auth_headers():
    return {"Authorization": "Bearer fake-token"}
```

```python
# test_items.py
import pytest

def test_read_items(client):
    response = client.get("/items/")
    assert response.status_code == 200
    assert isinstance(response.json(), list)

def test_create_item(client):
    response = client.post(
        "/items/",
        json={"name": "Test Item", "price": 25.50},
    )
    assert response.status_code == 201

def test_read_nonexistent_item(client):
    response = client.get("/items/nonexistent")
    assert response.status_code == 404

@pytest.mark.parametrize("item_id,expected_status", [
    ("foo", 200),
    ("bar", 200),
    ("nonexistent", 404),
])
def test_read_item_parametrized(client, item_id, expected_status):
    response = client.get(f"/items/{item_id}")
    assert response.status_code == expected_status
```

### Async Testing

```python
# test_async.py
import pytest
from httpx import AsyncClient
from main import app

@pytest.mark.anyio
async def test_read_root():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/")
        assert response.status_code == 200
        assert response.json() == {"message": "Hello World"}

@pytest.mark.anyio
async def test_create_item():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/items/",
            json={"name": "Test", "price": 10.0},
        )
        assert response.status_code == 201
```

### Testing with Dependency Overrides

```python
# main.py
from fastapi import FastAPI, Depends

app = FastAPI()

async def get_db():
    # Real database connection
    return {"connected": True}

@app.get("/items/")
async def read_items(db=Depends(get_db)):
    return {"db": db}
```

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app, get_db

def override_get_db():
    return {"connected": False, "mock": True}

app.dependency_overrides[get_db] = override_get_db

client = TestClient(app)

def test_read_items():
    response = client.get("/items/")
    assert response.status_code == 200
    assert response.json()["db"]["mock"] == True

# Clean up
app.dependency_overrides = {}
```

### Testing File Uploads

```python
# test_upload.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_upload_file():
    # Create test file content
    file_content = b"test file content"

    response = client.post(
        "/upload/",
        files={"file": ("test.txt", file_content, "text/plain")},
    )
    assert response.status_code == 200
    assert response.json()["filename"] == "test.txt"

def test_upload_multiple_files():
    files = [
        ("files", ("file1.txt", b"content1", "text/plain")),
        ("files", ("file2.txt", b"content2", "text/plain")),
    ]
    response = client.post("/uploadfiles/", files=files)
    assert response.status_code == 200
    assert len(response.json()["filenames"]) == 2
```

### Testing Authentication

```python
# test_auth.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_protected_route_without_token():
    response = client.get("/protected/")
    assert response.status_code == 401

def test_protected_route_with_invalid_token():
    response = client.get(
        "/protected/",
        headers={"Authorization": "Bearer invalid-token"},
    )
    assert response.status_code == 401

def test_protected_route_with_valid_token():
    # First, get a token
    login_response = client.post(
        "/token",
        data={"username": "testuser", "password": "testpass"},
    )
    token = login_response.json()["access_token"]

    # Then use the token
    response = client.get(
        "/protected/",
        headers={"Authorization": f"Bearer {token}"},
    )
    assert response.status_code == 200
```

---

## 18. Best Practices

### Project Structure

```
my_project/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI application entry point
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py        # Application configuration
│   │   ├── security.py      # Authentication & authorization
│   │   └── logging.py       # Logging configuration
│   ├── api/
│   │   ├── __init__.py
│   │   ├── deps.py          # Shared dependencies
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── api.py       # API router aggregation
│   │       └── endpoints/
│   │           ├── __init__.py
│   │           ├── users.py
│   │           └── items.py
│   ├── models/              # SQLAlchemy/database models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── item.py
│   ├── schemas/             # Pydantic models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── item.py
│   ├── crud/                # Database operations
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── user.py
│   │   └── item.py
│   ├── services/            # Business logic
│   │   ├── __init__.py
│   │   ├── user_service.py
│   │   └── item_service.py
│   └── db/
│       ├── __init__.py
│       ├── base.py
│       └── session.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── api/
│       └── test_users.py
├── alembic/                 # Database migrations
├── requirements.txt
├── pyproject.toml
└── Dockerfile
```

### Configuration Management

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # Application
    app_name: str = "My FastAPI App"
    debug: bool = False
    environment: str = "production"

    # API
    api_v1_prefix: str = "/api/v1"

    # Database
    database_url: str

    # Security
    secret_key: str
    access_token_expire_minutes: int = 30

    # CORS
    allowed_origins: list[str] = ["http://localhost:3000"]

    class Config:
        env_file = ".env"
        case_sensitive = False

@lru_cache
def get_settings() -> Settings:
    return Settings()

settings = get_settings()
```

### Input Validation Best Practices

```python
from pydantic import BaseModel, Field, field_validator, EmailStr
from typing import Annotated

class UserCreate(BaseModel):
    email: EmailStr
    username: Annotated[str, Field(min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_]+$")]
    password: Annotated[str, Field(min_length=8)]

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain at least one uppercase letter")
        if not any(c.isdigit() for c in v):
            raise ValueError("Password must contain at least one digit")
        return v
```

### Error Handling Best Practices

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel
import logging

logger = logging.getLogger(__name__)

class ErrorResponse(BaseModel):
    error: bool = True
    message: str
    code: str
    details: dict | None = None

class AppException(Exception):
    def __init__(self, status_code: int, code: str, message: str, details: dict | None = None):
        self.status_code = status_code
        self.code = code
        self.message = message
        self.details = details

app = FastAPI()

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    logger.error(f"AppException: {exc.code} - {exc.message}")
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            message=exc.message,
            code=exc.code,
            details=exc.details,
        ).model_dump(),
    )
```

### Dependency Injection Best Practices

```python
from fastapi import Depends
from typing import Annotated

# Define reusable dependencies with type aliases
async def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

async def get_current_user(token: str = Depends(oauth2_scheme)):
    # ... authentication logic
    return user

# Use Annotated for cleaner code
DB = Annotated[Session, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]

@app.get("/items/")
async def read_items(db: DB, user: CurrentUser):
    return crud.get_user_items(db, user.id)
```

### Async Database Best Practices

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql+asyncpg://user:password@localhost/dbname"

engine = create_async_engine(DATABASE_URL, echo=True)
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_async_db():
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Logging Best Practices

```python
import logging
import sys
from fastapi import FastAPI, Request
import time

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler("app.log"),
    ],
)
logger = logging.getLogger(__name__)

app = FastAPI()

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()

    response = await call_next(request)

    process_time = time.time() - start_time
    logger.info(
        f"{request.method} {request.url.path} "
        f"status={response.status_code} "
        f"duration={process_time:.3f}s"
    )

    return response
```

### Security Best Practices

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from passlib.context import CryptContext
import secrets

# Use strong password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Generate secure secret key
SECRET_KEY = secrets.token_urlsafe(32)

# Always validate tokens
async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        username: str = payload.get("sub")
        if username is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid authentication credentials",
            )
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials",
        )
    return username

# Rate limiting (use with a library like slowapi)
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()

@app.get("/limited/")
@limiter.limit("5/minute")
async def limited_endpoint(request: Request):
    return {"message": "This endpoint is rate limited"}
```

### Performance Best Practices

```python
from fastapi import FastAPI
from fastapi.responses import ORJSONResponse
import orjson

# Use faster JSON library
app = FastAPI(default_response_class=ORJSONResponse)

# Cache expensive operations
from functools import lru_cache

@lru_cache(maxsize=100)
def get_expensive_data(key: str):
    # Expensive computation
    return {"key": key, "data": "expensive"}

# Use connection pooling for databases
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=5,
    max_overflow=10,
    pool_timeout=30,
    pool_recycle=1800,
)

# Async operations for I/O bound tasks
import httpx

async def fetch_external_data(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()
```

### Documentation Best Practices

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel, Field

app = FastAPI(
    title="My API",
    description="""
## My API Description

This API allows you to:
* Create and manage items
* Handle user authentication
* Process orders

## Authentication

All endpoints require authentication via Bearer token.
    """,
    version="1.0.0",
    contact={
        "name": "API Support",
        "email": "support@example.com",
    },
)

class Item(BaseModel):
    """Represents an item in the system."""

    name: str = Field(..., description="The name of the item", examples=["Widget"])
    price: float = Field(..., gt=0, description="The price in USD", examples=[29.99])

    model_config = {
        "json_schema_extra": {
            "examples": [
                {"name": "Widget", "price": 29.99},
                {"name": "Gadget", "price": 49.99},
            ]
        }
    }

@app.post(
    "/items/",
    summary="Create a new item",
    description="Create a new item with the provided name and price.",
    response_description="The created item",
    responses={
        201: {"description": "Item created successfully"},
        400: {"description": "Invalid input"},
        422: {"description": "Validation error"},
    },
)
async def create_item(item: Item):
    """
    Create an item with all the information:

    - **name**: the name of the item (required)
    - **price**: the price of the item in USD (required, must be > 0)
    """
    return item
```

---

## Quick Reference

### Common Imports

```python
from fastapi import (
    FastAPI,
    APIRouter,
    Depends,
    HTTPException,
    status,
    Request,
    Response,
    Path,
    Query,
    Body,
    Header,
    Cookie,
    Form,
    File,
    UploadFile,
    BackgroundTasks,
)
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from fastapi.responses import JSONResponse, HTMLResponse, RedirectResponse
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field, field_validator, EmailStr
```

### Status Code Reference

| Code | Constant | Usage |
|------|----------|-------|
| 200 | `HTTP_200_OK` | Successful GET, PUT, PATCH |
| 201 | `HTTP_201_CREATED` | Successful POST (created) |
| 202 | `HTTP_202_ACCEPTED` | Async processing accepted |
| 204 | `HTTP_204_NO_CONTENT` | Successful DELETE |
| 400 | `HTTP_400_BAD_REQUEST` | Invalid request |
| 401 | `HTTP_401_UNAUTHORIZED` | Authentication required |
| 403 | `HTTP_403_FORBIDDEN` | Permission denied |
| 404 | `HTTP_404_NOT_FOUND` | Resource not found |
| 422 | `HTTP_422_UNPROCESSABLE_ENTITY` | Validation error |
| 500 | `HTTP_500_INTERNAL_SERVER_ERROR` | Server error |

---

## Additional Resources

- Official Documentation: https://fastapi.tiangolo.com/
- GitHub Repository: https://github.com/tiangolo/fastapi
- Starlette Documentation: https://www.starlette.io/
- Pydantic Documentation: https://docs.pydantic.dev/
- Uvicorn Documentation: https://www.uvicorn.org/
