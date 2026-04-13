# Tutorial 06: Building the HTTP API

## Goal

By the end of this tutorial you will have written all the FastAPI routers and wired them together in `app/main.py`, giving the project a fully functional HTTP API you can test in the browser. **Routing** is the mechanism that makes this possible — it maps each URL in the API to a Python function that handles it.

---

## Prerequisites

- Tutorial 05 (CRUD) is complete
- `app/crud.py` exists with all CRUD functions

---

## Concepts Introduced

- What a web API is and how frontend and backend communicate
- HTTP methods and status codes
- JSON as the data format
- How data enters a route: path parameters, query parameters, and request body
- `APIRouter` and how routes are defined
- `response_model` and how FastAPI serializes responses
- Dependency injection with `Depends()`

---

## Step 1: What Is a Web API?

Before writing a single line of code, it helps to understand what this whole layer is for.

A web application is split into two parts: the **frontend** (what runs in the browser or on a phone — buttons, forms, pages) and the **backend** (what runs on the server — business logic, the database). These two parts live on different machines and are written in different languages. They need a way to talk to each other.

That's what a **web API** is — a defined set of URLs that the frontend can call to ask the backend to do something. The frontend says *"give me the user with ID 5"* or *"create a new maintenance request"*, and the backend responds with data.

The key point is that the frontend never touches the database directly. It can only interact with whatever the backend exposes through the API. This separation means the frontend and backend can be built and deployed independently — a mobile app and a web app can both use the same API without the backend caring about the difference.

Here is the flow for a typical request:

```
Browser / Mobile App
        │
        │  GET http://localhost:8000/api/users/5
        ↓
   FastAPI Backend  ←── finds the matching route, runs its function
        │
        │  SELECT * FROM users WHERE user_id = 5
        ↓
   PostgreSQL Database
        │
        ↑  returns the row
        │
   FastAPI Backend  ←── converts the result to JSON
        │
        │  { "user_id": 5, "name": "..." }
        ↑
Browser / Mobile App
```

---

## Step 2: JSON

The frontend and backend speak to each other using **JSON** (JavaScript Object Notation) — a simple text format for representing structured data. Every language can read and write it, which is why it became the standard for web APIs.

A JSON object maps to a Python dictionary:

```json
{
  "user_id": 5,
  "name": "Mohannad",
  "email": "m@example.com",
  "role": 0
}
```

A JSON array maps to a Python list:

```json
[
  { "user_id": 1, "name": "Alice" },
  { "user_id": 2, "name": "Bob" }
]
```

Objects can be nested — for example, a room that includes its building:

```json
{
  "room_id": 3,
  "room_number": "101",
  "capacity": 30,
  "building": {
    "building_id": 1,
    "building_name": "Main Building"
  }
}
```

In this project, **Pydantic schemas define the shape of the JSON** for every resource. When the frontend sends a request body, Pydantic validates it. When the backend sends a response, Pydantic serializes the SQLAlchemy object into JSON. You already wrote these schemas in Tutorial 04.

---

## Step 3: HTTP Methods and Status Codes

Every request has two key parts that together describe what the client wants: a **method** and a **path**.

The **method** signals the intent:

| Method | Meaning | Example |
|--------|---------|---------|
| `GET` | Read data — retrieve a resource | `GET http://localhost:8000/api/users/5` — fetch user 5 |
| `POST` | Create a new resource | `POST http://localhost:8000/api/users/` — create a new user |
| `PUT` | Update an existing resource | `PUT http://localhost:8000/api/users/5` — update user 5 |
| `DELETE` | Delete a resource | `DELETE http://localhost:8000/api/users/5` — delete user 5 |

The same path can handle different methods — `GET /api/users/5` reads, `PUT /api/users/5` updates, `DELETE /api/users/5` deletes. The path identifies *what*, the method identifies *what to do with it*.

The **status code** in the response tells the client whether it worked:

| Code | Meaning | When it appears |
|------|---------|-----------------|
| `200 OK` | Success | Successful `GET` or `PUT` |
| `201 Created` | Resource was created | Successful `POST` |
| `204 No Content` | Success, nothing to return | Successful `DELETE` |
| `400 Bad Request` | The request was invalid | Duplicate email, business rule violated |
| `404 Not Found` | The resource doesn't exist | Querying a user ID that isn't in the database |
| `422 Unprocessable Entity` | Validation failed | Sending `"capacity": "lots"` when an integer is expected |

Status codes are how the backend communicates outcome without the frontend having to parse the response body. A `404` means "not found" regardless of what the body says.

---

## Step 4: How Data Enters a Route

A route can receive input from three distinct places. Understanding which to use is fundamental to designing clean endpoints.

### Path parameters

A path parameter is part of the URL itself, written as `{name}` in the route decorator:

```python
@router.get("/{user_id}")
def get_user(user_id: int):
    ...
```

When the client calls `GET http://localhost:8000/api/users/5`, FastAPI extracts `5` from the URL and passes it to the function as `user_id = 5`. Path parameters identify a specific resource — use them when you're operating on one thing by its ID.

### Query parameters

Query parameters are appended after a `?` in the URL. In code, they are function arguments with default values:

```python
@router.get("/")
def list_users(skip: int = 0, limit: int = 100):
    ...
```

`GET http://localhost:8000/api/users/?skip=20&limit=10` passes `skip=20` and `limit=10` to the function. If the client doesn't include them, the defaults apply. Query parameters are used for optional filtering, sorting, and pagination — things that refine a list, not identify a single resource.

### Request body

A request body is a JSON object sent alongside the request, used when you need to pass structured data. In code, it is declared as a Pydantic schema argument:

```python
@router.post("/")
def create_user(user: schemas.UserCreate):
    ...
```

The client sends:
```
POST http://localhost:8000/api/users/
Content-Type: application/json

{
  "email": "alice@example.com",
  "name": "Alice",
  "password": "secret123",
  "role": 1
}
```

FastAPI reads the body, validates it against `UserCreate`, and passes a populated schema object to the function. If a required field is missing or has the wrong type, FastAPI automatically returns a `422` error before the function even runs.

**How FastAPI tells them apart:** if the argument's type is a Pydantic model, it comes from the body. If it's a plain Python type (`int`, `str`, etc.) and its name matches something in the path (`{user_id}`), it's a path parameter. Otherwise, it's a query parameter.

---

## Step 5: What Is a Router?

A **route** is a mapping from an HTTP method + path (e.g. `GET /users/5`) to a Python function that handles it. When a request comes in, FastAPI finds the route whose method and path match, and calls its function.

A **router** is a collection of related routes grouped together in one file. FastAPI provides `APIRouter` for this purpose. You define routes on the router, then mount it on the main app with a URL prefix.

```python
# routers/users.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/{user_id}")
def get_user(user_id: int):
    ...

@router.post("/")
def create_user(...):
    ...
```

```python
# main.py
from .routers import users

app.include_router(users.router, prefix="/users")
```

The full endpoint is the **prefix + the decorator path**:

| Decorator | Prefix | Full endpoint |
|-----------|--------|---------------|
| `@router.get("/")` | `/api/users` | `GET /api/users/` |
| `@router.post("/")` | `/api/users` | `POST /api/users/` |
| `@router.get("/{user_id}")` | `/api/users` | `GET /api/users/{user_id}` |
| `@router.delete("/{user_id}")` | `/api/users` | `DELETE /api/users/{user_id}` |

Splitting routes across files keeps things organized and lets you reason about one resource at a time. `routers/users.py` only handles users, `routers/rooms.py` only handles rooms, and so on.

---

## Step 6: `response_model` — Controlling What Gets Sent Back

When you write a route, you return a Python object — usually a SQLAlchemy model instance. But the client expects JSON. FastAPI bridges this gap with `response_model`.

```python
@router.get("/{user_id}", response_model=schemas.UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db)):
    return crud.get_user(db, user_id)   # returns a SQLAlchemy User object
```

FastAPI takes the SQLAlchemy object, reads the fields defined in `UserResponse`, and serializes them as JSON. The client never sees the raw Python object — it sees this:

```json
{
  "user_id": 5,
  "email": "alice@example.com",
  "name": "Alice",
  "role": 1,
  "created_at": "2026-04-12T10:00:00Z"
}
```

Notice that `hashed_password` is not in the response — because it's not a field in `UserResponse`. This is an important security property: `response_model` acts as a filter. Any field on the model that isn't in the schema is silently excluded from the response.

This works because `UserResponse` has `from_attributes = True` in its `Config` class (from Tutorial 04). Without it, Pydantic would expect a dictionary and fail when given a SQLAlchemy object. With it, Pydantic reads attribute values directly off the object.

For list endpoints, wrap the schema in `List[]`:

```python
@router.get("/", response_model=List[schemas.UserResponse])
def list_users(...):
    return crud.get_users(db)   # returns a list of SQLAlchemy objects
```

---

## Step 7: Dependency Injection with `Depends()`

Every route that queries the database needs a database session. You could create one at the top of each function — but then you'd be responsible for closing it, handling errors, and repeating that setup in dozens of places.

FastAPI solves this with **dependency injection**. Instead of creating the session yourself, you declare that you need one and FastAPI provides it:

```python
from fastapi import Depends
from sqlalchemy.orm import Session
from ..database import get_db

@router.get("/")
def list_users(db: Session = Depends(get_db)):
    return crud.get_users(db)
```

`Depends(get_db)` tells FastAPI: *"before calling `list_users`, call `get_db()` and pass the result in as `db`."* FastAPI handles the full lifecycle — it calls `get_db`, injects the session, runs the route function, and closes the session when the request finishes (via the `finally` block in `get_db`).

This is the dependency injection pattern: functions declare what they need, and the framework provides it. The benefits are:

- **No repetition** — session management lives in one place (`database.py`)
- **Automatic cleanup** — sessions are always closed, even if an error occurs
- **Testability** — in tests, you can swap `get_db` for a test database without changing the route

`Depends()` is not limited to database sessions. Later, you will write a `get_current_user` dependency that reads a JWT token and returns the authenticated user. Any route that needs the current user just adds `current_user: User = Depends(get_current_user)` — the authentication logic runs automatically before the route function.

---

## Step 8: Create `app/routers/users.py`

With the concepts in place, here is the full users router. Read through it carefully — every other router follows the same structure.

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas
from ..database import get_db

router = APIRouter()


@router.get("/", response_model=List[schemas.UserResponse])
def list_users(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_users(db, skip=skip, limit=limit)


@router.post("/", response_model=schemas.UserResponse, status_code=201)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    try:
        return crud.create_user(db, user)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/{user_id}", response_model=schemas.UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db)):
    db_user = crud.get_user(db, user_id)
    if not db_user:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user


@router.put("/{user_id}", response_model=schemas.UserResponse)
def update_user(user_id: int, user: schemas.UserUpdate, db: Session = Depends(get_db)):
    try:
        db_user = crud.update_user(db, user_id, user)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    if not db_user:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user


@router.delete("/{user_id}", status_code=204)
def delete_user(user_id: int, db: Session = Depends(get_db)):
    if not crud.delete_user(db, user_id):
        raise HTTPException(status_code=404, detail="User not found")
```

Walk through each route:

**`list_users`** receives two query parameters — `skip` and `limit` — with defaults so they're optional. `GET http://localhost:8000/api/users/` returns the first 100 users. `GET http://localhost:8000/api/users/?skip=100&limit=50` returns the next 50. This is standard API pagination.

**`create_user`** receives the user data in the request body as a `UserCreate` schema. FastAPI validates it before calling the function — if `email` is missing or `password` is too short, the client gets a `422` automatically. Inside the function, if the CRUD layer raises a `ValueError` (e.g. email already exists), the router catches it and converts it to an HTTP `400`. The client never sees a Python exception — only an HTTP response.

**`get_user`** receives the user's ID as a path parameter. If `crud.get_user` returns `None`, the router raises a `404`. This is the standard pattern across all routes: CRUD returns `None` for missing resources, the router converts it to the appropriate HTTP error.

**`update_user`** needs two error checks: a `ValueError` from the CRUD layer (e.g. trying to update to an email that already belongs to another user), and a `None` return (the user being updated doesn't exist). Both map to different status codes — `400` and `404` respectively.

**`delete_user`** has no `response_model` and no `return` statement. The `204 No Content` status code means the operation succeeded but there is nothing to send back. The route simply checks whether the deletion found anything, and returns a `404` if not.

---

## Step 9: Create `app/routers/buildings.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas
from ..database import get_db

router = APIRouter()


@router.get("/", response_model=List[schemas.BuildingResponse])
def list_buildings(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_buildings(db, skip=skip, limit=limit)


@router.post("/", response_model=schemas.BuildingResponse, status_code=201)
def create_building(building: schemas.BuildingCreate, db: Session = Depends(get_db)):
    try:
        return crud.create_building(db, building)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/{building_id}", response_model=schemas.BuildingWithRooms)
def get_building(building_id: int, db: Session = Depends(get_db)):
    db_building = crud.get_building(db, building_id)
    if not db_building:
        raise HTTPException(status_code=404, detail="Building not found")
    return db_building


@router.put("/{building_id}", response_model=schemas.BuildingResponse)
def update_building(building_id: int, building: schemas.BuildingUpdate, db: Session = Depends(get_db)):
    try:
        db_building = crud.update_building(db, building_id, building)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    if not db_building:
        raise HTTPException(status_code=404, detail="Building not found")
    return db_building


@router.delete("/{building_id}", status_code=204)
def delete_building(building_id: int, db: Session = Depends(get_db)):
    if not crud.delete_building(db, building_id):
        raise HTTPException(status_code=404, detail="Building not found")


@router.get("/{building_id}/rooms", response_model=List[schemas.RoomResponse])
def list_rooms_by_building(building_id: int, skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    if not crud.get_building(db, building_id):
        raise HTTPException(status_code=404, detail="Building not found")
    return crud.get_rooms_by_building(db, building_id, skip=skip, limit=limit)
```

Two things here differ from the users router:

**Richer response schema.** `GET http://localhost:8000/api/buildings/{building_id}` uses `BuildingWithRooms` instead of `BuildingResponse`. The route function is identical — only the schema changes. SQLAlchemy loads the related rooms automatically via the `relationship` defined in `models.py`, and Pydantic serializes them as a nested list. The client sees:

```json
{
  "building_id": 1,
  "building_name": "Main Building",
  "location": "North Campus",
  "rooms": [
    { "room_id": 1, "room_number": "101", "capacity": 30 },
    { "room_id": 2, "room_number": "102", "capacity": 25 }
  ]
}
```

**Nested route.** `GET http://localhost:8000/api/buildings/{building_id}/rooms` is a sub-resource route — it returns the rooms that belong to a specific building. This is a common REST convention: `/parent/{id}/children`. Before querying, the route verifies the building exists and returns a `404` if not.

---

## Step 10: Generate the Remaining Routers

Rooms, Requests, Schedules, and Reservations follow the exact same structure. Use Claude to generate them one at a time, using the buildings router as the reference.

Open a Claude Code conversation and send:

```
Look at app/routers/buildings.py. Using the exact same structure, generate
app/routers/rooms.py for the Room resource.

Specifics:
- Prefix will be /api/rooms
- Use RoomCreate, RoomUpdate, RoomResponse schemas
- GET /{room_id} should use RoomWithBuilding as response_model
- No nested sub-resource routes needed
```

Then for requests:

```
Generate app/routers/requests.py for the Request resource.

Specifics:
- Prefix will be /api/requests
- Use RequestCreate, RequestUpdate, RequestResponse, RequestWithDetails schemas
- GET /{request_id} should use RequestWithDetails as response_model
- Add GET /user/{user_id} — returns all requests by a user (use get_requests_by_user)
- Add GET /room/{room_id} — returns all requests for a room (use get_requests_by_room)
- POST / takes an extra user_id: int query parameter for now (auth is not implemented yet).
  Pass it to crud.create_request(db, request, user_id).
  Add a comment: # TODO: replace with auth dependency
```

Then for schedules and reservations, describe their filter routes and any extra parameters the same way.

> **Note:** Review each generated file carefully before accepting it. Check that schema names match `schemas.py` exactly, that filter routes verify the parent resource exists before querying, and that `status_code=204` routes have no `return` statement and no `response_model`.

---

## Step 11: Create `app/main.py`

`main.py` is the entry point for the entire application. It creates the FastAPI app instance, tells SQLAlchemy to create the database tables, and mounts all the routers:

```python
from fastapi import FastAPI
from .database import engine, Base
from .routers import users, buildings, rooms, requests, schedules, reservations

Base.metadata.create_all(bind=engine)

app = FastAPI(title="FCITR Maintenance & Room Reservation System")

app.include_router(users.router, prefix="/api/users", tags=["Users"])
app.include_router(buildings.router, prefix="/api/buildings", tags=["Buildings"])
app.include_router(rooms.router, prefix="/api/rooms", tags=["Rooms"])
app.include_router(requests.router, prefix="/api/requests", tags=["Requests"])
app.include_router(schedules.router, prefix="/api/schedules", tags=["Schedules"])
app.include_router(reservations.router, prefix="/api/reservations", tags=["Reservations"])
```

`Base.metadata.create_all(bind=engine)` runs on startup. It looks at all the models defined in `models.py` and creates any tables that don't exist yet in the database. It is safe to call repeatedly — it skips tables that are already there.

`tags` is a label that groups endpoints together in the Swagger UI, making it easier to navigate when there are many endpoints.

---

## Step 12: Run and Test

Start the server:

```bash
uvicorn app.main:app --reload
```

`--reload` watches your files and restarts the server automatically when you make changes — essential during development.

Open your browser at `http://localhost:8000/docs`. FastAPI generates an interactive **Swagger UI** directly from your code. Every endpoint is listed with its expected inputs, request body shape, and response schema. You can call any endpoint from the browser without writing any client code.

To test an endpoint in the UI:
1. Click on the endpoint to expand it
2. Click **Try it out**
3. Fill in the parameters or paste a request body
4. Click **Execute**
5. Check the **Response body** and **Response code** below

**Suggested test sequence** — each step builds on the previous:

1. `POST /api/users/` with body `{"email": "alice@example.com", "name": "Alice", "password": "secret123", "role": 1}` — note the `user_id` in the response
2. `GET /api/users/` — confirm Alice appears
3. `GET /api/users/{user_id}` — fetch Alice by ID
4. `POST /api/buildings/` with body `{"building_name": "Main Building", "location": "North Campus"}` — note the `building_id`
5. `POST /api/rooms/` with body `{"room_number": "101", "building_id": 1, "capacity": 30, "room_type": 0}` — use the `building_id` from step 4
6. `GET /api/buildings/{building_id}` — confirm the room appears nested in the response
7. `PUT /api/users/{user_id}` with body `{"name": "Alice Updated"}` — update just the name
8. `DELETE /api/users/{user_id}` — confirm the response is `204`
9. `GET /api/users/{user_id}` — confirm the response is `404`

---

## Summary

You have:

- Understood how a web API works and why the frontend and backend communicate through HTTP
- Learned HTTP methods (`GET`, `POST`, `PUT`, `DELETE`) and status codes (`200`, `201`, `204`, `400`, `404`)
- Understood the three ways data enters a route: path parameters, query parameters, and request body
- Written routers that connect HTTP endpoints to CRUD functions using `APIRouter`
- Used `response_model` to control what JSON the client receives and to filter out sensitive fields
- Used `Depends(get_db)` to inject a database session automatically into every route
- Wired all routers together in `main.py` with prefixes and tags
- Tested the full API using the Swagger UI at `/docs`

**What comes next:** Tutorial 07 will cover authentication — issuing JWT tokens on login and protecting routes so only authenticated users can access them.
