# Tutorial 09: Room QR Codes

## Goal

By the end of this tutorial you will have added a `GET /api/rooms/{room_id}/qr` endpoint that returns a QR code image. Scanning the code opens the maintenance request form in the frontend, pre-filled with the room's ID.

---

## Prerequisites

- Tutorial 08 (Authentication) is complete
- `app/config.py` exists with `FRONTEND_BASE_URL`

---

## Concepts Introduced

- Generating QR codes in Python with the `qrcode` library
- Returning binary (non-JSON) responses from FastAPI using `StreamingResponse`
- Using `io.BytesIO` as an in-memory file buffer

---

## Step 1: What QR Codes Are Used For Here

Every room in the system has a physical space — a classroom, a lab, a meeting room. When something breaks in that room, a student or faculty member needs a quick way to report it without having to navigate a form, remember URLs, or know the room's internal ID.

The solution is a QR code printed and posted on the wall of each room. Scanning it opens a URL like:

```
http://localhost:5173/report?room_id=7
```

The frontend reads `room_id` from the URL and pre-fills the maintenance request form with the correct room. The user only needs to describe the problem and submit.

The backend generates the QR code on demand — when an admin calls the endpoint for a given room, the server creates the image and sends it back. The admin can then download and print it.

---

## Step 2: The `qrcode` Library

Install the library if it is not already in your environment:

```bash
pip install qrcode[pil]
```

The `[pil]` extra installs `Pillow`, which `qrcode` uses to render the image.

Basic usage is a single function call:

```python
import qrcode

img = qrcode.make("https://example.com")
img.save("output.png")
```

`qrcode.make` returns a `PIL.Image` object. You can save it to a file, or — more useful in an API — save it to an in-memory buffer and stream it directly in the response.

---

## Step 3: Returning a Binary Response with `StreamingResponse`

Normally FastAPI routes return Python objects that Pydantic serializes to JSON. But a QR code is a PNG image — binary data, not JSON. FastAPI's `StreamingResponse` handles this case: it accepts a file-like object and a media type, and streams the bytes to the client.

The pattern is:

```python
import io
from fastapi.responses import StreamingResponse

buf = io.BytesIO()          # an in-memory buffer (acts like a file)
img.save(buf, format="PNG") # write the image into the buffer
buf.seek(0)                 # rewind to the beginning so it can be read

return StreamingResponse(buf, media_type="image/png")
```

`io.BytesIO` is an in-memory byte stream — it behaves exactly like a file opened in binary mode, but nothing is written to disk. After saving the image into it, `seek(0)` rewinds the read position to the start, so `StreamingResponse` reads all the bytes from the beginning.

`media_type="image/png"` sets the `Content-Type` header in the response. The client (browser or admin tool) uses this to know it is receiving an image, not JSON.

---

## Step 4: Update `app/routers/rooms.py`

Add the QR code endpoint to the rooms router:

```python
import io
import qrcode
from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import StreamingResponse
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas
from ..config import FRONTEND_BASE_URL
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


@router.get("/{room_id}/qr", dependencies=[Depends(get_current_user)])
def get_room_qr(room_id: int, db: Session = Depends(get_db)):
    if not crud.get_room(db, room_id):
        raise HTTPException(status_code=404, detail="Room not found")
    url = f"{FRONTEND_BASE_URL}/report?room_id={room_id}"
    img = qrcode.make(url)
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    buf.seek(0)
    return StreamingResponse(buf, media_type="image/png")
```

The endpoint:
1. Verifies the room exists — returns `404` if not
2. Builds the target URL using `FRONTEND_BASE_URL` from config, so the link points to the correct frontend regardless of environment
3. Generates the QR code image
4. Streams it back as a PNG

Any authenticated user can call this endpoint. There is no admin restriction — a faculty member might want to scan the QR code to verify it works before reporting an issue.

---

## Step 5: Test It

Start the server and open `http://localhost:8000/docs`. Authenticate, then:

1. Create a building and a room if you do not have one already
2. Find `GET /api/rooms/{room_id}/qr`, click **Try it out**, enter the room ID, and execute
3. The Swagger UI will show a download link — click it to see the PNG

You can also open the URL directly in a browser:

```
http://localhost:8000/api/rooms/1/qr
```

With a valid `Authorization: Bearer <token>` header, the browser will display the QR code image inline. You can right-click and save it, or print it for the physical room.

To verify the QR code encodes the correct URL, scan it with your phone — it should open (or attempt to open) `http://localhost:5173/report?room_id=1`.

---

## Summary

You have:

- Understood how QR codes fit into the maintenance reporting workflow
- Used the `qrcode` library to generate a PNG image in memory
- Returned binary image data from a FastAPI route using `StreamingResponse` and `io.BytesIO`
- Added `GET /api/rooms/{room_id}/qr` to the rooms router

**What comes next:** The backend is now feature-complete. The next phase is building the frontend — the React application that consumes this API, including the maintenance request form that the QR codes link to.
