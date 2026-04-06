# Tutorial 04: Pydantic Schemas

## Goal

By the end of this tutorial you will have written `app/schemas.py` — the layer that validates incoming request data and controls what data is returned in API responses.

---

## Prerequisites

- Tutorial 03 (database models) is complete
- `app/models.py` exists with all six models

---

## Concepts Introduced

- What Pydantic is and why schemas are separate from models
- The Base / Create / Update / Response schema pattern
- `Optional` fields and how updates work
- `Field()` for validation rules and defaults
- `from_attributes = True` for converting SQLAlchemy objects to schemas
- Nested schemas for rich responses

---

## Step 1: Models vs Schemas — What's the Difference?

Your SQLAlchemy **models** define the database tables. Your Pydantic **schemas** define the shape of data flowing in and out of the API.

They are intentionally separate:

| | SQLAlchemy Model | Pydantic Schema |
|---|---|---|
| Purpose | Talks to the database | Validates API input/output |
| Defined with | `Column(...)` | Python type annotations |
| Used in | `crud.py` | Routers |

For example, your `User` model has a `hashed_password` column. You never want to expose that in an API response. By having a separate `UserResponse` schema, you simply leave `hashed_password` out of it.

---

## Step 2: The Four-Schema Pattern

For each resource you will define four schemas. Using Building as a concrete example:

| Schema | Purpose |
|---|---|
| `BuildingBase` | Common fields shared between Create and Response |
| `BuildingCreate` | Fields required to create a new building |
| `BuildingUpdate` | Fields that can be changed — all `Optional` |
| `BuildingResponse` | What the API returns — includes DB-generated fields like `building_id` and `created_at` |

`BuildingCreate` and `BuildingResponse` both inherit from `BuildingBase`, so they get the common fields automatically. `BuildingUpdate` stands alone because all its fields must be optional.

---

## Step 3: Set Up `app/schemas.py`

Create `app/schemas.py` and add the imports:

```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime, date, time
from typing import Optional, List
```

- `BaseModel` — the base class for all schemas
- `EmailStr` — a string type that validates email format
- `Field()` — adds validation rules or metadata to a field
- `Optional` — marks a field as not required (can be `None`)
- `List` — for fields that return multiple items

---

## Step 4: User Schemas

```python
# ==================== User Schemas ====================

class UserBase(BaseModel):
    email: EmailStr
    name: str
    role: int = Field(default=0, description="0: Admin, 1: Student, 2: Faculty, 3: Staff")


class UserCreate(UserBase):
    password: str = Field(..., min_length=8)


class UserUpdate(BaseModel):
    email: Optional[EmailStr] = None
    name: Optional[str] = None
    role: Optional[int] = None
    password: Optional[str] = Field(None, min_length=8)


class UserResponse(UserBase):
    user_id: int
    last_login: Optional[datetime] = None
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True
```

Key decisions to understand:

**`UserBase`** holds the fields that both creation and response share: `email`, `name`, `role`.

**`UserCreate`** inherits all of those and adds `password`. The `Field(..., min_length=8)` means the field is required (`...`) and must be at least 8 characters. Pydantic enforces this before any of your code runs.

**`UserUpdate`** does not inherit from `UserBase`. Every field is `Optional` with a default of `None`, because a PATCH request only sends the fields the client wants to change. Using `model_dump(exclude_unset=True)` in `crud.py` will later let you apply only the fields that were actually sent.

**`UserResponse`** adds `user_id`, `created_at`, `updated_at`, and `last_login` — fields that only exist after the record is saved to the database. It does not include `password` or `hashed_password`, so they are never returned to the client.

**`class Config: from_attributes = True`** tells Pydantic it's allowed to read data from a SQLAlchemy model object (which uses attribute access) rather than from a plain dictionary. Without this, converting a SQLAlchemy object to a schema would fail.

> **Note:** Only `Response` schemas need `Config`. `Create` and `Update` schemas receive plain data from the client, not SQLAlchemy objects.

---

## Step 5: Building Schemas

```python
# ==================== Building Schemas ====================

class BuildingBase(BaseModel):
    building_name: str
    location: str


class BuildingCreate(BuildingBase):
    pass


class BuildingUpdate(BaseModel):
    building_name: Optional[str] = None
    location: Optional[str] = None


class BuildingResponse(BuildingBase):
    building_id: int
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True
```

`BuildingCreate` just uses `pass` because `BuildingBase` already has everything needed to create a building. This is common when no extra fields are required.

---

## Step 6: Exercise — Room Schemas

Write the `Room` schemas following the same pattern. Open `schema.md` to see the full column list.

A few things to keep in mind:

- `RoomBase` should include `room_number`, `building_id`, `capacity`, and `room_type`
- `RoomCreate` can use `pass`
- In `RoomUpdate`, `capacity` benefits from a `Field` with a description — look at how `role` uses `Field` in `UserBase` as a reference
- `RoomResponse` should add `room_id`, `created_at`, `updated_at`, and include `Config`

Once you've written the four Room schemas, add these two **nested schemas** below them:

```python
class RoomWithBuilding(RoomResponse):
    building: BuildingResponse


class BuildingWithRooms(BuildingResponse):
    rooms: List[RoomResponse] = []
```

These inherit from the response schemas and add relationship data. `RoomWithBuilding` embeds the full building object inside the room response. `BuildingWithRooms` embeds a list of rooms inside the building response. SQLAlchemy's `relationship()` attributes on the model objects make this work automatically — Pydantic will traverse them when `from_attributes = True` is set.

---

## Step 7: Exercise — Request, Schedule, and Reservation Schemas

Write the remaining three groups of schemas. Open `schema.md` for the column definitions. Use the User and Building schemas as your guide for structure.

There are a few intentional differences from the standard pattern — think carefully about each:

**Request:**
- `RequestCreate` should **not** inherit from `RequestBase`. When a user submits a maintenance request, they provide only `room_id` and `description` — the `user_id` comes from the authenticated session, not the request body, and `status` defaults to `0` (Open) automatically.
- Add a `RequestWithDetails` schema that extends `RequestResponse` with a `user: UserResponse` and a `room: RoomWithBuilding` field.

**Schedule:**
- `day_of_week` should use `Field(..., ge=0, le=6)` — `ge` means "greater than or equal to", `le` means "less than or equal to". This constrains the value to 0–6.
- `start_time` and `end_time` use the `time` type; `valid_from` and `valid_until` use the `date` type (both imported at the top).
- `updated_at` in `ScheduleResponse` should be `Optional[datetime] = None` because a schedule entry may never be updated.
- Add a `ScheduleWithDetails` schema with `user: UserResponse` and `room: RoomWithBuilding`.

**Reservation:**
- `request_id` in `ReservationBase` is `Optional[int] = None` because the link to a maintenance request is optional.
- `ReservationCreate` should not inherit from `ReservationBase` — the client provides `request_id` (optional), `room_id`, `reservation_date`, `start_time`, and `end_time`. The `reserved_by` (user) and `status` are set server-side.
- `ReservationUpdate` only allows changing `status`.
- `updated_at` in `ReservationResponse` should be `Optional[datetime] = None`.
- Add a `ReservationWithDetails` schema with `user: UserResponse`, `room: RoomWithBuilding`, and `request: Optional[RequestResponse] = None`.

---

## Summary

You have:

- Learned why schemas and models are kept separate
- Applied the Base / Create / Update / Response pattern for every resource
- Used `Field()` to enforce validation rules on input data
- Configured `from_attributes = True` on response schemas to work with SQLAlchemy objects
- Built nested schemas that embed related objects in responses

**What comes next:** Tutorial 05 will cover CRUD — the functions that use these schemas to read and write to the database.
