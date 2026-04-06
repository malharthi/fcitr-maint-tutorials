# Tutorial 05: CRUD Operations

## Goal

By the end of this tutorial you will have written `app/crud.py` — the functions that read and write data to the database — and used Claude to generate the CRUD operations for the remaining resources.

---

## Prerequisites

- Tutorial 04 (schemas) is complete
- `app/schemas.py` exists with all schemas

---

## Concepts Introduced

- The CRUD pattern (Create, Read, Update, Delete)
- SQLAlchemy session lifecycle: `add`, `commit`, `refresh`, `rollback`
- `model_dump(exclude_unset=True)` for partial updates
- Handling `IntegrityError` for unique constraint violations
- Password hashing with `passlib`
- Using Claude to generate repetitive code from an established pattern

---

## Step 1: What Is CRUD?

CRUD stands for **Create, Read, Update, Delete** — the four fundamental operations on any resource. In this project, `crud.py` contains plain Python functions that take a database session and some input, perform one of these operations, and return a result.

These functions are intentionally kept separate from the routers. Routers handle HTTP concerns (status codes, request/response shapes). CRUD functions handle database concerns. This separation makes both easier to test and maintain.

---

## Step 2: The Session Lifecycle

Every CRUD function receives a `db: Session` argument. Here is what the key session methods do:

| Method | What it does |
|---|---|
| `db.query(Model).filter(...).first()` | Runs a SELECT and returns one result or `None` |
| `db.query(Model).filter(...).all()` | Runs a SELECT and returns a list |
| `db.add(obj)` | Stages a new object — not saved yet |
| `db.commit()` | Writes all staged changes to the database |
| `db.refresh(obj)` | Reloads the object from the database (needed to get the auto-generated `id` and `created_at`) |
| `db.rollback()` | Undoes all staged changes since the last commit |
| `db.delete(obj)` | Stages an object for deletion |

`commit` and `rollback` work like a transaction: either everything succeeds and is committed, or something fails and you roll back to a clean state.

---

## Step 3: Install `passlib`

User passwords must be hashed before being stored. Add `passlib` and `bcrypt` to your environment:

```bash
pip install passlib bcrypt
pip freeze > requirements.txt
```

---

## Step 4: Set Up `app/crud.py`

Create `app/crud.py` with these imports and the password hashing helpers:

```python
from sqlalchemy.orm import Session
from sqlalchemy.exc import IntegrityError
from . import models, schemas
from passlib.context import CryptContext
from typing import List, Optional

# Password hashing configuration
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)
```

`CryptContext` manages the hashing algorithm. `hash_password` is used on creation and update; `verify_password` will be used in authentication later.

---

## Step 5: User CRUD

```python
# ==================== User CRUD Operations ====================

def get_user(db: Session, user_id: int) -> Optional[models.User]:
    return db.query(models.User).filter(models.User.user_id == user_id).first()


def get_user_by_email(db: Session, email: str) -> Optional[models.User]:
    return db.query(models.User).filter(models.User.email == email).first()


def get_users(db: Session, skip: int = 0, limit: int = 100) -> List[models.User]:
    return db.query(models.User).offset(skip).limit(limit).all()


def create_user(db: Session, user: schemas.UserCreate) -> models.User:
    db_user = models.User(
        email=user.email,
        name=user.name,
        hashed_password=hash_password(user.password),
        role=user.role
    )
    db.add(db_user)
    try:
        db.commit()
        db.refresh(db_user)
        return db_user
    except IntegrityError:
        db.rollback()
        raise ValueError(f"Email {user.email} already exists")


def update_user(db: Session, user_id: int, user: schemas.UserUpdate) -> Optional[models.User]:
    db_user = get_user(db, user_id)
    if not db_user:
        return None

    update_data = user.model_dump(exclude_unset=True)

    if "password" in update_data:
        update_data["hashed_password"] = hash_password(update_data.pop("password"))

    for field, value in update_data.items():
        setattr(db_user, field, value)

    try:
        db.commit()
        db.refresh(db_user)
        return db_user
    except IntegrityError:
        db.rollback()
        raise ValueError(f"Email {db_user.email} already exists")


def delete_user(db: Session, user_id: int) -> bool:
    db_user = get_user(db, user_id)
    if not db_user:
        return False
    db.delete(db_user)
    db.commit()
    return True
```

Walk through the key patterns:

**Read functions** (`get_user`, `get_users`) are straightforward queries. Pagination is handled by `.offset(skip).limit(limit)` — skip the first N records, then return at most `limit` records. This prevents returning the entire table on every call.

**`create_user`** — notice that `user.password` is hashed before being stored. The raw password never touches the database. The `try/except IntegrityError` catches unique constraint violations (duplicate email) and rolls back the session before raising a clean `ValueError`. Without the rollback, the session would be left in a broken state.

**`update_user`** uses `model_dump(exclude_unset=True)`. This returns only the fields that were actually provided in the request — not the ones that defaulted to `None`. This way, sending `{"name": "Alice"}` only updates `name`; it doesn't wipe out `email` or `role`.

The password field requires special handling in updates: the schema calls it `password`, but the model column is `hashed_password`. The update loop pops `password` from the data dict, hashes it, and writes it back as `hashed_password`.

**`delete_user`** returns a `bool` instead of the object. This is a clean convention — the router only needs to know whether the deletion succeeded, not what was deleted.

---

## Step 6: Building CRUD

```python
# ==================== Building CRUD Operations ====================

def get_building(db: Session, building_id: int) -> Optional[models.Building]:
    return db.query(models.Building).filter(models.Building.building_id == building_id).first()


def get_building_by_name(db: Session, building_name: str) -> Optional[models.Building]:
    return db.query(models.Building).filter(models.Building.building_name == building_name).first()


def get_buildings(db: Session, skip: int = 0, limit: int = 100) -> List[models.Building]:
    return db.query(models.Building).offset(skip).limit(limit).all()


def create_building(db: Session, building: schemas.BuildingCreate) -> models.Building:
    db_building = models.Building(
        building_name=building.building_name,
        location=building.location
    )
    db.add(db_building)
    try:
        db.commit()
        db.refresh(db_building)
        return db_building
    except IntegrityError:
        db.rollback()
        raise ValueError(f"Building name {building.building_name} already exists")


def update_building(db: Session, building_id: int, building: schemas.BuildingUpdate) -> Optional[models.Building]:
    db_building = get_building(db, building_id)
    if not db_building:
        return None

    update_data = building.model_dump(exclude_unset=True)

    for field, value in update_data.items():
        setattr(db_building, field, value)

    try:
        db.commit()
        db.refresh(db_building)
        return db_building
    except IntegrityError:
        db.rollback()
        raise ValueError(f"Building name {db_building.building_name} already exists")


def delete_building(db: Session, building_id: int) -> bool:
    db_building = get_building(db, building_id)
    if not db_building:
        return False
    db.delete(db_building)
    db.commit()
    return True
```

The structure is identical to User CRUD — read by ID, read all, create, update with `model_dump`, delete returning bool. This is the pattern you will repeat for every resource.

---

## Step 7: Generate the Rest with Claude

Room, Request, Schedule, and Reservation follow the same pattern. Rather than writing each one manually, you can describe what you need to Claude and have it generate the code.

Open a new Claude Code conversation in your project and send a message like this:

```
Look at the building CRUD functions in app/crud.py. Using the exact same 
pattern, generate CRUD functions for the Room model. 

Room has these specifics:
- Primary key: room_id
- Schema types: RoomCreate, RoomUpdate
- For create_room, verify that the building_id exists by calling get_building 
  before inserting. Raise a ValueError if it doesn't.
- For update_room, if building_id is being changed, verify the new building 
  exists before committing.
- Add a get_rooms_by_building function that filters by building_id.
- The IntegrityError message should mention the room_number and building_id.
```

Then repeat for Request, Schedule, and Reservation. For each, note the resource-specific details Claude needs:

**Request:**
- Primary key: `request_id`
- `create_request` takes an extra `user_id: int` argument (not in the schema — it comes from the authenticated session)
- Verify both `user_id` and `room_id` exist before inserting
- Add `get_requests_by_user` and `get_requests_by_room` and `get_requests_by_status` filter functions

**Schedule:**
- Primary key: `schedule_id`
- Add `get_schedules_by_room` and `get_schedules_by_user` filter functions

**Reservation:**
- Primary key: `reservation_id`
- `create_reservation` takes an extra `reserved_by: int` argument
- Verify `room_id` and `reserved_by` exist before inserting
- If `request_id` is provided, verify it exists too
- Add `get_reservations_by_user` and `get_reservations_by_room` filter functions

> **Note:** Review the generated code carefully before accepting it. Verify that the model attribute names match your `models.py` exactly, that foreign key checks are in place where needed, and that `IntegrityError` is caught and rolled back on create and update.

---

## Summary

You have:

- Learned the SQLAlchemy session lifecycle and when to use `add`, `commit`, `refresh`, and `rollback`
- Written CRUD functions for User and Building, establishing the pattern
- Used `model_dump(exclude_unset=True)` to safely apply partial updates
- Handled `IntegrityError` to catch unique constraint violations cleanly
- Used Claude to generate the remaining CRUD operations from the established pattern

**What comes next:** Tutorial 06 will cover setting up the FastAPI routers — wiring these CRUD functions to HTTP endpoints.
