# Tutorial 08: Authentication

## Goal

By the end of this tutorial you will have implemented JWT-based authentication. Users will log in with their email and password and receive a token. Every other endpoint will require that token, and will know who the caller is and what they are allowed to do.

---

## Prerequisites

- Tutorial 07 (Environment Configuration) is complete
- `app/config.py` exists with `SECRET_KEY`

---

## Concepts Introduced

- Authentication vs. authorization
- JWT (JSON Web Tokens) — structure, signing, and expiry
- The Bearer token flow in HTTP
- FastAPI's `OAuth2PasswordBearer` and `OAuth2PasswordRequestForm`
- Dependency chaining
- Two patterns for protecting routes: `dependencies=[]` vs. injected `current_user`

---

## Step 1: Authentication vs. Authorization

These two words are often confused, but they mean different things:

- **Authentication** — proving who you are. Logging in with your email and password authenticates you. The system verifies your identity.
- **Authorization** — deciding what you are allowed to do. Even after authentication, not everyone can do everything. A student can submit maintenance requests; only an admin can delete them.

This tutorial implements both. The login endpoint handles authentication (verifying identity and issuing a token). The route protection handles authorization (checking the token and enforcing role requirements before each request is processed).

---

## Step 2: What Is a JWT?

A **JWT (JSON Web Token)** is a compact, self-contained token. When a user logs in successfully, the server creates a JWT and sends it back. The client stores it and includes it in every subsequent request. The server reads the token and knows who is calling — without querying the database every time.

A JWT has three parts separated by dots:

```
header.payload.signature
```

- **Header** — the algorithm used to sign the token (`HS256`)
- **Payload** — the claims: data stored in the token, such as the user ID and expiry time
- **Signature** — a cryptographic hash of the header and payload, signed with `SECRET_KEY`. This guarantees the token has not been tampered with.

A typical payload for this project looks like:

```json
{
  "sub": "42",
  "exp": 1718000000
}
```

`sub` (subject) is the user ID. `exp` is the expiry timestamp — after this point the token is rejected.

The server never stores tokens. When a request arrives with a token, the server verifies the signature using `SECRET_KEY`. If the signature is valid and the token has not expired, the server trusts the payload.

---

## Step 3: The Login Flow

Here is the full flow from login to a protected request:

```
1. Client sends POST /api/auth/login with email + password

2. Server looks up the user by email, verifies the password

3. If valid: server creates a JWT containing the user's ID, returns it

4. Client stores the token

5. Client sends GET /api/rooms/ with header:
       Authorization: Bearer <token>

6. Server extracts the token, verifies the signature, reads the user ID from the payload

7. Server fetches the user from the database using that ID

8. Server checks the user's role against what the route requires

9. If all passes: the route function runs
```

The key insight is that the token carries the user's identity. The server does not need to look up a session — it trusts the signed token.

---

## Step 4: Install the Required Packages

Two packages handle the token and password work. Add them to your virtual environment if they are not already installed:

```bash
pip install python-jose passlib[bcrypt]
```

`python-jose` creates and verifies JWTs. `passlib` with the `bcrypt` scheme hashes and verifies passwords. You may also need `python-multipart` for the login form:

```bash
pip install python-multipart
```

---

## Step 5: Create `app/auth.py`

This module handles token creation and decoding. It does not touch the database — it only works with the token string itself.

```python
from datetime import datetime, timedelta, timezone
from jose import JWTError, jwt
from .config import SECRET_KEY

ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60 * 8  # 8 hours


def create_access_token(user_id: int) -> str:
    payload = {
        "sub": str(user_id),
        "exp": datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


def decode_access_token(token: str) -> int | None:
    """Decode a JWT and return the user_id, or None if invalid."""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = payload.get("sub")
        if user_id is None:
            return None
        return int(user_id)
    except JWTError:
        return None
```

`create_access_token` takes a user ID, builds a payload with the ID and an expiry time, and returns a signed token string.

`decode_access_token` does the reverse: it verifies the signature and expiry, then extracts and returns the user ID. If anything is wrong — bad signature, expired token, malformed token — it returns `None` instead of raising an exception. The caller decides what to do with `None`.

---

## Step 6: Create `app/dependencies.py`

Dependencies are functions that FastAPI calls automatically before a route runs. This module defines three of them, all related to auth.

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session

from . import crud, models
from .auth import decode_access_token
from .database import get_db

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/login")


def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db),
) -> models.User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Invalid or expired token",
        headers={"WWW-Authenticate": "Bearer"},
    )
    user_id = decode_access_token(token)
    if user_id is None:
        raise credentials_exception

    user = crud.get_user(db, user_id)
    if user is None:
        raise credentials_exception

    return user


def require_admin(current_user: models.User = Depends(get_current_user)) -> models.User:
    """Allow only Staff and Staff Manager (roles 0–1). Full system management access."""
    if current_user.role not in (0, 1):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Insufficient permissions",
        )
    return current_user


def require_monitor(current_user: models.User = Depends(get_current_user)) -> models.User:
    """Allow Staff, Staff Manager, Dean, Vice Dean, and Department Head (roles 0–4). Read/monitor access."""
    if current_user.role not in (0, 1, 2, 3, 4):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Insufficient permissions",
        )
    return current_user
```

**`oauth2_scheme`** tells FastAPI where to find the token. `OAuth2PasswordBearer` reads the `Authorization: Bearer <token>` header from incoming requests. `tokenUrl` is only used by Swagger UI to know where to send the login form — it does not change the actual behavior.

**`get_current_user`** is the core dependency. It:
1. Receives the raw token string (extracted from the header by `oauth2_scheme`)
2. Decodes it to get the user ID
3. Fetches the user from the database
4. Returns the user object — or raises `401` if anything fails

**`require_admin`** and **`require_monitor`** depend on `get_current_user` (dependency chaining). They call it first, then check the role. If the role check fails they raise `403`. Otherwise they return the user.

---

## Step 7: Create `app/routers/auth.py`

```python
from datetime import datetime, timezone

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session
from pydantic import BaseModel

from .. import crud
from ..auth import create_access_token
from ..database import get_db

router = APIRouter()


class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"


@router.post("/login", response_model=TokenResponse)
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db),
):
    user = crud.get_user_by_email(db, form_data.username)
    if not user or not crud.verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    user.last_login = datetime.now(timezone.utc)
    db.commit()

    token = create_access_token(user.user_id)
    return TokenResponse(access_token=token)
```

`OAuth2PasswordRequestForm` expects form data (not JSON) with `username` and `password` fields. By convention, `username` holds the email — the field name is fixed by the OAuth2 standard, but the value is an email.

If the email is not found, or the password does not match, the route returns a `401`. The error message is intentionally vague (`"Incorrect email or password"`) — you never tell the caller whether the email or the password was wrong, as that would help an attacker enumerate valid accounts.

On success, the route records `last_login`, creates a token, and returns it.

---

## Step 8: Two Patterns for Protecting Routes

Before updating the routers, understand the two patterns you will see.

### Pattern 1 — Pure gating with `dependencies=[]`

Use this when a route must enforce a role requirement, but the function body does not need to know who is calling:

```python
@router.delete("/{building_id}", status_code=204, dependencies=[Depends(require_admin)])
def delete_building(building_id: int, db: Session = Depends(get_db)):
    ...
```

The dependency runs before the function. If the user is not an admin, FastAPI raises `403` and the function never runs. The function signature stays clean — no `current_user` parameter cluttering it.

### Pattern 2 — Injected `current_user`

Use this when the function body needs the user — typically for ownership checks where the allowed action depends on *who* is calling:

```python
@router.get("/{user_id}", response_model=schemas.UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    if current_user.role not in (0, 1) and current_user.user_id != user_id:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
    ...
```

Here, any authenticated user can call the endpoint, but non-admins can only fetch their own profile. The function needs the user object to make that decision, so it is injected as a parameter.

Use Pattern 1 when access is purely role-based. Use Pattern 2 when the decision depends on the specific caller.

---

## Step 9: Update `app/routers/users.py`

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas, models
from ..database import get_db
from ..dependencies import get_current_user, require_admin

router = APIRouter()


@router.get("/", response_model=List[schemas.UserResponse], dependencies=[Depends(require_admin)])
def list_users(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_users(db, skip=skip, limit=limit)


@router.post("/", response_model=schemas.UserResponse, status_code=201, dependencies=[Depends(require_admin)])
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    try:
        return crud.create_user(db, user)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/{user_id}", response_model=schemas.UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    if current_user.role not in (0, 1) and current_user.user_id != user_id:  # type: ignore[operator]
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
    db_user = crud.get_user(db, user_id)
    if not db_user:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user


@router.put("/{user_id}", response_model=schemas.UserResponse)
def update_user(user_id: int, user: schemas.UserUpdate, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    if current_user.role not in (0, 1) and current_user.user_id != user_id:  # type: ignore[operator]
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
    if current_user.role not in (0, 1) and user.role is not None:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Cannot change your own role")
    try:
        db_user = crud.update_user(db, user_id, user)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    if not db_user:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user


@router.delete("/{user_id}", status_code=204, dependencies=[Depends(require_admin)])
def delete_user(user_id: int, db: Session = Depends(get_db)):
    if not crud.delete_user(db, user_id):
        raise HTTPException(status_code=404, detail="User not found")
```

`list_users` and `create_user` are admin-only — Pattern 1 keeps the function signatures clean. `get_user` and `update_user` use Pattern 2 because non-admins are allowed, but only for their own account. `update_user` adds a second check: non-admins cannot change their own role.

---

## Step 10: Update `app/routers/buildings.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas
from ..database import get_db
from ..dependencies import get_current_user, require_admin

router = APIRouter()


@router.get("/", response_model=List[schemas.BuildingResponse], dependencies=[Depends(get_current_user)])
def list_buildings(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_buildings(db, skip=skip, limit=limit)


@router.post("/", response_model=schemas.BuildingResponse, status_code=201, dependencies=[Depends(require_admin)])
def create_building(building: schemas.BuildingCreate, db: Session = Depends(get_db)):
    try:
        return crud.create_building(db, building)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/{building_id}", response_model=schemas.BuildingWithRooms, dependencies=[Depends(get_current_user)])
def get_building(building_id: int, db: Session = Depends(get_db)):
    db_building = crud.get_building(db, building_id)
    if not db_building:
        raise HTTPException(status_code=404, detail="Building not found")
    return db_building


@router.put("/{building_id}", response_model=schemas.BuildingResponse, dependencies=[Depends(require_admin)])
def update_building(building_id: int, building: schemas.BuildingUpdate, db: Session = Depends(get_db)):
    try:
        db_building = crud.update_building(db, building_id, building)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    if not db_building:
        raise HTTPException(status_code=404, detail="Building not found")
    return db_building


@router.delete("/{building_id}", status_code=204, dependencies=[Depends(require_admin)])
def delete_building(building_id: int, db: Session = Depends(get_db)):
    if not crud.delete_building(db, building_id):
        raise HTTPException(status_code=404, detail="Building not found")


@router.get("/{building_id}/rooms", response_model=List[schemas.RoomResponse], dependencies=[Depends(get_current_user)])
def list_rooms_by_building(building_id: int, skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    if not crud.get_building(db, building_id):
        raise HTTPException(status_code=404, detail="Building not found")
    return crud.get_rooms_by_building(db, building_id, skip=skip, limit=limit)
```

Read operations require any authenticated user (`get_current_user`). Write operations require admin (`require_admin`). Since neither case needs conditional logic based on *who* is calling, all of these use Pattern 1.

---

## Step 11: Update `app/routers/rooms.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas
from ..database import get_db
from ..dependencies import get_current_user, require_admin

router = APIRouter()


@router.get("/", response_model=List[schemas.RoomResponse], dependencies=[Depends(get_current_user)])
def list_rooms(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_rooms(db, skip=skip, limit=limit)


@router.post("/", response_model=schemas.RoomResponse, status_code=201, dependencies=[Depends(require_admin)])
def create_room(room: schemas.RoomCreate, db: Session = Depends(get_db)):
    try:
        return crud.create_room(db, room)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/{room_id}", response_model=schemas.RoomWithBuilding, dependencies=[Depends(get_current_user)])
def get_room(room_id: int, db: Session = Depends(get_db)):
    db_room = crud.get_room(db, room_id)
    if not db_room:
        raise HTTPException(status_code=404, detail="Room not found")
    return db_room


@router.put("/{room_id}", response_model=schemas.RoomResponse, dependencies=[Depends(require_admin)])
def update_room(room_id: int, room: schemas.RoomUpdate, db: Session = Depends(get_db)):
    try:
        db_room = crud.update_room(db, room_id, room)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    if not db_room:
        raise HTTPException(status_code=404, detail="Room not found")
    return db_room


@router.delete("/{room_id}", status_code=204, dependencies=[Depends(require_admin)])
def delete_room(room_id: int, db: Session = Depends(get_db)):
    if not crud.delete_room(db, room_id):
        raise HTTPException(status_code=404, detail="Room not found")
```

---

## Step 12: Update `app/routers/requests.py`

Requests introduce a mix of access levels: any authenticated user can submit one, monitors and admins can see all of them, but students and faculty can only see their own.

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas, models
from ..database import get_db
from ..dependencies import get_current_user, require_admin, require_monitor

router = APIRouter()


@router.get("/", response_model=List[schemas.RequestResponse], dependencies=[Depends(require_monitor)])
def list_requests(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_requests(db, skip=skip, limit=limit)


@router.post("/", response_model=schemas.RequestResponse, status_code=201)
def create_request(request: schemas.RequestCreate, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    try:
        return crud.create_request(db, request, current_user.user_id)  # type: ignore[arg-type]
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/user/{user_id}", response_model=List[schemas.RequestResponse])
def list_requests_by_user(user_id: int, skip: int = 0, limit: int = 100, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    if current_user.role not in (0, 1) and current_user.user_id != user_id:  # type: ignore[operator]
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
    if not crud.get_user(db, user_id):
        raise HTTPException(status_code=404, detail="User not found")
    return crud.get_requests_by_user(db, user_id, skip=skip, limit=limit)


@router.get("/room/{room_id}", response_model=List[schemas.RequestResponse], dependencies=[Depends(require_monitor)])
def list_requests_by_room(room_id: int, skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    if not crud.get_room(db, room_id):
        raise HTTPException(status_code=404, detail="Room not found")
    return crud.get_requests_by_room(db, room_id, skip=skip, limit=limit)


@router.get("/{request_id}", response_model=schemas.RequestWithDetails)
def get_request(request_id: int, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    db_request = crud.get_request(db, request_id)
    if not db_request:
        raise HTTPException(status_code=404, detail="Request not found")
    if current_user.role not in (0, 1, 2, 3, 4) and db_request.user_id != current_user.user_id:  # type: ignore[operator]
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
    return db_request


@router.put("/{request_id}", response_model=schemas.RequestResponse, dependencies=[Depends(require_admin)])
def update_request(request_id: int, request: schemas.RequestUpdate, db: Session = Depends(get_db)):
    try:
        db_request = crud.update_request(db, request_id, request)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    if not db_request:
        raise HTTPException(status_code=404, detail="Request not found")
    return db_request


@router.delete("/{request_id}", status_code=204, dependencies=[Depends(require_admin)])
def delete_request(request_id: int, db: Session = Depends(get_db)):
    if not crud.delete_request(db, request_id):
        raise HTTPException(status_code=404, detail="Request not found")
```

`create_request` uses Pattern 2 because it needs to record `current_user.user_id` as the submitter. `get_request` uses Pattern 2 because the ownership check compares the caller against the request's `user_id` field. The list and write routes use Pattern 1.

---

## Step 13: Update `app/routers/schedules.py`

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas, models
from ..database import get_db
from ..dependencies import get_current_user, require_admin

router = APIRouter()


@router.get("/", response_model=List[schemas.ScheduleResponse], dependencies=[Depends(get_current_user)])
def list_schedules(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_schedules(db, skip=skip, limit=limit)


@router.post("/", response_model=schemas.ScheduleResponse, status_code=201, dependencies=[Depends(require_admin)])
def create_schedule(schedule: schemas.ScheduleCreate, db: Session = Depends(get_db)):
    try:
        return crud.create_schedule(db, schedule)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/room/{room_id}", response_model=List[schemas.ScheduleResponse], dependencies=[Depends(get_current_user)])
def list_schedules_by_room(room_id: int, skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    if not crud.get_room(db, room_id):
        raise HTTPException(status_code=404, detail="Room not found")
    return crud.get_schedules_by_room(db, room_id, skip=skip, limit=limit)


@router.get("/user/{user_id}", response_model=List[schemas.ScheduleResponse])
def list_schedules_by_user(user_id: int, skip: int = 0, limit: int = 100, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    if current_user.role not in (0, 1) and current_user.user_id != user_id:  # type: ignore[operator]
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
    if not crud.get_user(db, user_id):
        raise HTTPException(status_code=404, detail="User not found")
    return crud.get_schedules_by_user(db, user_id, skip=skip, limit=limit)


@router.get("/{schedule_id}", response_model=schemas.ScheduleWithDetails, dependencies=[Depends(get_current_user)])
def get_schedule(schedule_id: int, db: Session = Depends(get_db)):
    db_schedule = crud.get_schedule(db, schedule_id)
    if not db_schedule:
        raise HTTPException(status_code=404, detail="Schedule not found")
    return db_schedule


@router.put("/{schedule_id}", response_model=schemas.ScheduleResponse, dependencies=[Depends(require_admin)])
def update_schedule(schedule_id: int, schedule: schemas.ScheduleUpdate, db: Session = Depends(get_db)):
    try:
        db_schedule = crud.update_schedule(db, schedule_id, schedule)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    if not db_schedule:
        raise HTTPException(status_code=404, detail="Schedule not found")
    return db_schedule


@router.delete("/{schedule_id}", status_code=204, dependencies=[Depends(require_admin)])
def delete_schedule(schedule_id: int, db: Session = Depends(get_db)):
    if not crud.delete_schedule(db, schedule_id):
        raise HTTPException(status_code=404, detail="Schedule not found")
```

---

## Step 14: Update `app/routers/reservations.py`

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas, models
from ..database import get_db
from ..dependencies import get_current_user, require_admin, require_monitor

router = APIRouter()


@router.get("/", response_model=List[schemas.ReservationResponse], dependencies=[Depends(require_monitor)])
def list_reservations(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_reservations(db, skip=skip, limit=limit)


@router.post("/", response_model=schemas.ReservationResponse, status_code=201)
def create_reservation(reservation: schemas.ReservationCreate, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    try:
        return crud.create_reservation(db, reservation, current_user.user_id)  # type: ignore[arg-type]
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("/user/{user_id}", response_model=List[schemas.ReservationResponse])
def list_reservations_by_user(user_id: int, skip: int = 0, limit: int = 100, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    if current_user.role not in (0, 1, 2, 3, 4) and current_user.user_id != user_id:  # type: ignore[operator]
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
    if not crud.get_user(db, user_id):
        raise HTTPException(status_code=404, detail="User not found")
    return crud.get_reservations_by_user(db, user_id, skip=skip, limit=limit)


@router.get("/room/{room_id}", response_model=List[schemas.ReservationResponse], dependencies=[Depends(require_monitor)])
def list_reservations_by_room(room_id: int, skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    if not crud.get_room(db, room_id):
        raise HTTPException(status_code=404, detail="Room not found")
    return crud.get_reservations_by_room(db, room_id, skip=skip, limit=limit)


@router.get("/{reservation_id}", response_model=schemas.ReservationWithDetails)
def get_reservation(reservation_id: int, db: Session = Depends(get_db), current_user: models.User = Depends(get_current_user)):
    db_reservation = crud.get_reservation(db, reservation_id)
    if not db_reservation:
        raise HTTPException(status_code=404, detail="Reservation not found")
    if current_user.role not in (0, 1, 2, 3, 4) and db_reservation.reserved_by != current_user.user_id:  # type: ignore[operator]
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
    return db_reservation


@router.put("/{reservation_id}", response_model=schemas.ReservationResponse, dependencies=[Depends(require_admin)])
def update_reservation(reservation_id: int, reservation: schemas.ReservationUpdate, db: Session = Depends(get_db)):
    try:
        db_reservation = crud.update_reservation(db, reservation_id, reservation)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    if not db_reservation:
        raise HTTPException(status_code=404, detail="Reservation not found")
    return db_reservation


@router.delete("/{reservation_id}", status_code=204, dependencies=[Depends(require_admin)])
def delete_reservation(reservation_id: int, db: Session = Depends(get_db)):
    if not crud.delete_reservation(db, reservation_id):
        raise HTTPException(status_code=404, detail="Reservation not found")
```

`list_reservations_by_user` allows monitors and admins to see any user's reservations, but students and faculty can only see their own — hence Pattern 2 with the expanded role check `not in (0, 1, 2, 3, 4)`.

---

## Step 15: Update `app/main.py`

Add the auth router:

```python
from fastapi import FastAPI
from .database import engine, Base
from .routers import auth, users, buildings, rooms, requests, schedules, reservations

Base.metadata.create_all(bind=engine)

app = FastAPI(title="FCITR Maintenance & Room Reservation System")

app.include_router(auth.router, prefix="/api/auth", tags=["Auth"])
app.include_router(users.router, prefix="/api/users", tags=["Users"])
app.include_router(buildings.router, prefix="/api/buildings", tags=["Buildings"])
app.include_router(rooms.router, prefix="/api/rooms", tags=["Rooms"])
app.include_router(requests.router, prefix="/api/requests", tags=["Requests"])
app.include_router(schedules.router, prefix="/api/schedules", tags=["Schedules"])
app.include_router(reservations.router, prefix="/api/reservations", tags=["Reservations"])
```

---

## Step 16: Test in Swagger

Start the server:

```bash
uvicorn app.main:app --reload
```

Open `http://localhost:8000/docs`. You will notice a new **Authorize** button at the top right. This is added automatically because `OAuth2PasswordBearer` is present in the app.

**Test the full login flow:**

1. First, create a user. Find `POST /api/users/` — but notice it now requires auth. Since no users exist yet, you need a workaround: temporarily comment out `dependencies=[Depends(require_admin)]` on the `create_user` route to seed the first admin user. Create a user with `role: 0` (Staff). Then restore the dependency.

   > **Note:** In a real project you would create the first admin user via a seeding script or directly in the database rather than temporarily disabling auth.

2. Call `POST /api/auth/login` with `username` = the email you just created and `password` = the password. You will get back an `access_token`.

3. Click the **Authorize** button, paste the token, and click **Authorize**.

4. Now try any protected endpoint — it should work. Try removing the token and calling the same endpoint — you should get a `401`.

5. Create a user with `role: 6` (Student) and log in as them. Try calling `GET /api/users/` — you should get a `403`.

---

## Summary

You have:

- Understood the difference between authentication and authorization
- Learned how JWTs work: structure, signing, and expiry
- Implemented token creation and decoding in `app/auth.py`
- Written three dependency functions in `app/dependencies.py`: `get_current_user`, `require_admin`, `require_monitor`
- Created the login endpoint in `app/routers/auth.py`
- Protected every router using two patterns: `dependencies=[Depends(...)]` for pure role gating and injected `current_user` for ownership-based logic

**What comes next:** Tutorial 09 will add a QR code endpoint to the rooms router, generating a scannable code that links directly to the maintenance request form for a specific room.
