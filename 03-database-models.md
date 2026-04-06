# Tutorial 02: ORM & Database Models with SQLAlchemy

## Goal

By the end of this tutorial you will have written `app/database.py` and `app/models.py` — the two files that define how your Python code connects to PostgreSQL and what tables exist in the database.

---

## Prerequisites

- Tutorial 01 (environment setup) is complete
- PostgreSQL is running and your database exists
- Your virtual environment is active and dependencies are installed

---

## Concepts Introduced

- What an ORM is and why it is useful
- SQLAlchemy's declarative base and how models map to tables
- Column types and constraints
- Primary keys, foreign keys, and composite unique constraints
- `server_default` vs `default`, and `onupdate`
- Relationships between models
- `ondelete` behaviour (CASCADE and SET NULL)

---

## Step 1: What Is an ORM?

An **ORM (Object-Relational Mapper)** is a library that lets you work with a relational database using Python classes and objects instead of writing raw SQL.

Without an ORM you would write:

```sql
INSERT INTO users (email, name, hashed_password, role)
VALUES ('alice@example.com', 'Alice', 'hashed...', 2);
```

With SQLAlchemy you write:

```python
user = User(email="alice@example.com", name="Alice", hashed_password="hashed...", role=2)
db.add(user)
db.commit()
```

The ORM translates your Python code into SQL and sends it to the database. Benefits:

- You write Python, not SQL
- Your code is database-agnostic (you could switch from PostgreSQL to another database with minimal changes)
- Relationships between tables become Python attributes you can traverse naturally

---

## Step 2: SQLAlchemy Core Ideas

Before writing any code, understand these three building blocks:

| Concept | What it is |
|---|---|
| **Engine** | The connection to the database. Created once at startup. |
| **Session** | A unit of work — a temporary workspace where you stage changes before committing them to the database. |
| **Base** | A special class that all your models inherit from. SQLAlchemy uses it to discover your table definitions. |

---

## Step 3: Create `app/database.py`

This file sets up the engine, session factory, and Base.

Create `app/database.py`:

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import os
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

# Get database URL from environment
SQLALCHEMY_DATABASE_URL = os.getenv("DATABASE_URL")

# Create database engine
# Engine manages connections to the database
engine = create_engine(SQLALCHEMY_DATABASE_URL)

# Create SessionLocal class
# Each instance is a database session
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Create Base class
# All models will inherit from this
Base = declarative_base()

# Dependency function for FastAPI
# Provides a database session for each request
def get_db():
    db = SessionLocal()
    try:
        yield db  # Provide session to the route
    finally:
        db.close()  # Close session after request
```

**Key points:**

- `create_engine(...)` — takes the connection URL and creates a pool of database connections.
- `sessionmaker(autocommit=False, autoflush=False, bind=engine)` — creates a factory for sessions. `autocommit=False` means you must explicitly call `db.commit()` to save changes.
- `declarative_base()` — returns the `Base` class. Every model you define will inherit from it, and SQLAlchemy will use that to find all your tables.
- `get_db()` — a FastAPI dependency. It opens a session, hands it to a route function via `yield`, then closes it when the request is done. You will use this in every router.

---

## Step 4: How Models Work

A **model** is a Python class that represents a database table. Each class attribute that is a `Column(...)` maps to a column in the table.

```python
class User(Base):
    __tablename__ = "users"       # the actual PostgreSQL table name

    user_id = Column(Integer, primary_key=True, index=True)
    email   = Column(String, unique=True, nullable=False)
```

When you call `Base.metadata.create_all(bind=engine)` in `main.py`, SQLAlchemy reads all classes that inherit from `Base` and creates the corresponding tables in PostgreSQL if they don't already exist.

### Column types

| SQLAlchemy type | PostgreSQL type | Python type |
|---|---|---|
| `Integer` | `INTEGER` | `int` |
| `String` | `VARCHAR` | `str` |
| `DateTime(timezone=True)` | `TIMESTAMPTZ` | `datetime` |
| `Date` | `DATE` | `date` |
| `Time` | `TIME` | `time` |

### Common column options

| Option | Meaning |
|---|---|
| `primary_key=True` | This column is the primary key |
| `index=True` | Create a database index for faster lookups |
| `unique=True` | No two rows may have the same value |
| `nullable=False` | The column cannot be NULL |
| `default=0` | Python-side default applied before INSERT |
| `server_default=func.now()` | Database-side default (the DB sets it) |
| `onupdate=func.now()` | Database automatically updates this column on every UPDATE |

> **Note:** Use `server_default` (not `default`) for timestamps like `created_at`. This way the database itself records the exact time of insertion, regardless of any clock differences between the app server and the database server.

---

## Step 5: Create `app/models.py`

Now write all six models. Create `app/models.py`:

```python
from sqlalchemy import (
    Column, ForeignKey, Integer,
    String, DateTime, UniqueConstraint, Date, Time
)
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from .database import Base
```

The imports bring in:
- Column types and `ForeignKey` from `sqlalchemy`
- `relationship` to declare links between models
- `func` for server-side functions like `func.now()`
- `Base` from our own `database.py`

### 5.1 User

```python
class User(Base):
    __tablename__ = "users"

    user_id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    name = Column(String, unique=False, nullable=False)
    hashed_password = Column(String, nullable=False)
    role = Column(Integer, default=0)  # 0: Admin, 1: Student, 2: Faculty, 3: Staff
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    last_login = Column(DateTime(timezone=True), nullable=True)
```

Notice:
- `last_login` has no `nullable=False`, so it defaults to allowing NULL — this is correct because a newly registered user has not logged in yet.
- `updated_at` uses `onupdate=func.now()` which means the database sets it automatically every time this row is updated. It stays NULL until the first update.
- We store `hashed_password`, never the plain-text password.

### 5.2 Building

```python
class Building(Base):
    __tablename__ = "buildings"

    building_id = Column(Integer, primary_key=True, index=True)
    building_name = Column(String, unique=True, index=True, nullable=False)
    location = Column(String, unique=False, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    rooms = relationship("Room", back_populates="building")
```

The `relationship("Room", back_populates="building")` line does not create a database column. It creates a Python attribute that lets you access all rooms belonging to a building:

```python
building.rooms  # returns a list of Room objects
```

`back_populates="building"` means the `Room` model has a matching `relationship` attribute named `building` that points back to this `Building`. Both sides must declare the link.

### 5.3 Room

```python
class Room(Base):
    __tablename__ = "rooms"

    room_id = Column(Integer, primary_key=True, index=True)
    room_number = Column(String, nullable=False)
    building_id = Column(
        Integer,
        ForeignKey("buildings.building_id", ondelete="CASCADE"),
        nullable=False
    )
    capacity = Column(Integer, nullable=False)
    room_type = Column(Integer, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    building = relationship("Building", back_populates="rooms")

    # Composite Unique Constraint: room_number + building_id
    __table_args__ = (
        UniqueConstraint('building_id', 'room_number', name='_building_room_uc'),
    )
```

**Foreign keys:** `ForeignKey("buildings.building_id", ondelete="CASCADE")` means:
1. `building_id` in this table must match an existing `building_id` in the `buildings` table.
2. `ondelete="CASCADE"` — if the referenced building is deleted, all its rooms are automatically deleted too.

**Composite unique constraint:** `room_number` alone is not unique (different buildings can have a "Room 101"). The combination of `building_id + room_number` must be unique. `__table_args__` lets you add table-level constraints that apply to multiple columns.

> **Note:** `__table_args__` must be a **tuple**. When you have only one constraint, don't forget the trailing comma: `(UniqueConstraint(...),)`.

### 5.4 Exercise: Request

Now it's your turn. Open the [Database Schema](https://gist.github.com/malharthi/7c21310098a7418afb3da69944616700) and find the `requests` table. Write the `Request` model yourself using everything you've learned so far.

A few things to keep in mind:

- This model has **two foreign keys** — one to `users` and one to `rooms`. Both use `ondelete="CASCADE"`.
- Add a `user` relationship pointing to `User` and a `room` relationship pointing to `Room`. These don't need `back_populates` — the link is one-directional.
- There is no composite unique constraint on this table.

---

### 5.5 Exercise: Schedule

Open the [Database Schema](https://gist.github.com/malharthi/7c21310098a7418afb3da69944616700) and find the `schedules` table. Write the `Schedule` model.

A few things to keep in mind:

- `start_time` and `end_time` use the `Time` type; `valid_from` and `valid_until` use the `Date` type. `Time` stores only the time-of-day (e.g., `09:00:00`), `Date` stores only the calendar date (e.g., `2025-09-01`). Both are already in the imports.
- Add a `room` relationship pointing to `Room` and a `user` relationship pointing to `User`.
- Add a composite unique constraint on `(room_id, day_of_week, start_time, valid_from)` — name it `_schedule_slot_uc`. This prevents the same room from being double-booked in the same weekly slot within a semester.

---

### 5.6 Exercise: Reservation

Open the [Database Schema](https://gist.github.com/malharthi/7c21310098a7418afb3da69944616700) and find the `reservations` table. Write the `Reservation` model. This is the most complex one — read the hints below before you start.

> **Hint — nullable FK with SET NULL:** `request_id` is optional. A reservation can exist without being linked to a maintenance request, and if a linked request is later deleted, the reservation should survive with `request_id` set to NULL. This requires two things together: `nullable=True` on the column, and `ondelete="SET NULL"` on the `ForeignKey`.

> **Hint — ambiguous relationships:** This model has two foreign keys to the `users` table. SQLAlchemy will raise an error because it can't determine automatically which FK the `user` relationship should use. You must tell it explicitly by passing `foreign_keys=[reserved_by]` to the `relationship()` call.

Add a composite unique constraint on `(room_id, reservation_date, start_time)` — name it `_reservation_slot_uc`.

---

## Step 6: Register Models in `main.py`

For SQLAlchemy to create the tables, two things must happen:

1. All model classes must be **imported** before `create_all` is called (importing the file is enough — Python executes the class definitions which register them with `Base`).
2. `Base.metadata.create_all(bind=engine)` must be called.

In `app/main.py`, add these lines near the top:

```python
from .database import engine, Base
from . import models  # This import registers all models with Base

Base.metadata.create_all(bind=engine)
```

> **Note:** `Base.metadata.create_all` is fine for development. For production or any schema changes after initial deployment you would use a migration tool like **Alembic** instead, which can safely alter existing tables.

---

## Step 7: Verify the Tables Were Created

Start the server to trigger `create_all`:

```bash
uvicorn app.main:app --reload
```

Then connect to your database and confirm the tables exist:

```bash
psql -U fastapi_user -d fastapi_db -h localhost
```

```sql
\dt
```

You should see all six tables listed: `users`, `buildings`, `rooms`, `requests`, `schedules`, `reservations`.

To inspect a table's columns:

```sql
\d users
\d rooms
```

Exit when done:

```sql
\q
```

---

## Summary

You have:

- Created `database.py` which sets up the SQLAlchemy engine, session factory, and `Base`
- Written six model classes in `models.py` that map to the six database tables
- Used column types (`Integer`, `String`, `DateTime`, `Date`, `Time`) and constraints (`nullable`, `unique`, `primary_key`, `ForeignKey`, `UniqueConstraint`)
- Understood the difference between `server_default` and `default`, and the purpose of `onupdate`
- Declared relationships with `relationship()` and `back_populates`
- Handled the two `ondelete` behaviours: `CASCADE` and `SET NULL`

**What comes next:** Tutorial 03 will cover Pydantic schemas — the layer that validates incoming request data and shapes outgoing response data before it reaches or leaves your models.
