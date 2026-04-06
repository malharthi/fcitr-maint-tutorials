# Complete FastAPI + SQLAlchemy + PostgreSQL Backend Guide

A comprehensive guide for teaching bachelor students to build REST APIs.

---

## Table of Contents
1. [Python Environment Setup](#part-1-python-environment-setup)
2. [PostgreSQL Installation & Setup](#part-2-postgresql-installation--setup)
3. [Project Initialization](#part-3-project-initialization)
4. [Database Configuration](#part-4-database-configuration)
5. [Building the Backend](#part-5-building-the-backend)
6. [Running & Testing](#part-6-running--testing)
7. [Next Steps & Best Practices](#part-7-next-steps--best-practices)

---

## Part 1: Python Environment Setup

### **Windows Setup**

**Step 1.1: Install Python**
1. Download Python 3.11+ from [python.org](https://python.org)
2. Run installer - **CRITICAL**: Check "Add Python to PATH"
3. Verify installation:
```bash
python --version
pip --version
```

**Step 1.2: Install Code Editor**
- Download VS Code from [code.visualstudio.com](https://code.visualstudio.com)
- Install the Python extension (search "Python" in Extensions)

---

### **macOS Setup**

**Step 1.1: Install Homebrew** (if not installed)
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

**Step 1.2: Install Python**
```bash
brew install python@3.11
python3 --version
pip3 --version
```

**Step 1.3: Install Code Editor**
```bash
brew install --cask visual-studio-code
```

---

## Part 2: PostgreSQL Installation & Setup

### **Windows - PostgreSQL Installation**

**Step 2.1: Download and Install PostgreSQL**
1. Visit [https://www.postgresql.org/download/windows/](https://www.postgresql.org/download/windows/)
2. Download the installer (PostgreSQL 15 or 16)
3. Run installer and configure:
   - **Set a password for 'postgres' user** (write it down!)
   - Port: **5432** (default - keep it)
   - Locale: Default

**Step 2.2: Verify Installation**
```bash
# Open Command Prompt
psql --version
```

If you get "command not found":
- Add PostgreSQL to PATH: `C:\Program Files\PostgreSQL\15\bin`
- Restart terminal

**Step 2.3: Create Database and User**
```bash
# Login as postgres superuser
psql -U postgres
# Enter the password you set during installation
```

In the PostgreSQL prompt:
```sql
-- Create database
CREATE DATABASE fastapi_db;

-- Create user
CREATE USER fastapi_user WITH PASSWORD 'SecurePass123!';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE fastapi_db TO fastapi_user;

-- Connect to the new database
\c fastapi_db

-- Grant schema privileges (PostgreSQL 15+)
GRANT ALL ON SCHEMA public TO fastapi_user;

-- Verify
\l                    -- List databases
\du                   -- List users

-- Exit
\q
```

---

### **macOS - PostgreSQL Installation**

**Step 2.1: Install PostgreSQL**
```bash
brew install postgresql@15
```

**Step 2.2: Start PostgreSQL Service**
```bash
# Start and enable auto-start on boot
brew services start postgresql@15

# Verify it's running
brew services list
```

**Step 2.3: Create Database and User**
```bash
# Login to PostgreSQL (uses your macOS username by default)
psql postgres
```

In the PostgreSQL prompt:
```sql
-- Create database
CREATE DATABASE fastapi_db;

-- Create user
CREATE USER fastapi_user WITH PASSWORD 'SecurePass123!';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE fastapi_db TO fastapi_user;

-- Connect to the new database
\c fastapi_db

-- Grant schema privileges (PostgreSQL 15+)
GRANT ALL ON SCHEMA public TO fastapi_user;

-- Verify
\l                    -- List databases
\du                   -- List users

-- Exit
\q
```

---

### **Test PostgreSQL Connection**

**Both Windows and macOS:**
```bash
# Test connection with your new user
psql -U fastapi_user -d fastapi_db -h localhost

# If successful, you'll see:
# Password for user fastapi_user: [enter SecurePass123!]
# fastapi_db=>

# Exit
\q
```

---

## Part 3: Project Initialization

### **Step 3.1: Create Project Directory**

**Windows:**
```bash
mkdir fastapi-project
cd fastapi-project
```

**macOS:**
```bash
mkdir fastapi-project
cd fastapi-project
```

---

### **Step 3.2: Create Virtual Environment**

**Windows:**
```bash
# Create virtual environment
python -m venv venv

# Activate it (Command Prompt)
venv\Scripts\activate

# OR activate in PowerShell (if execution policy error, see below)
venv\Scripts\Activate.ps1
```

**If you get PowerShell execution policy error:**
```powershell
# Run PowerShell as Administrator
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**macOS:**
```bash
# Create virtual environment
python3 -m venv venv

# Activate it
source venv/bin/activate
```

You should see `(venv)` prefix in your terminal.

---

### **Step 3.3: Install Python Dependencies**

**Windows:**
```bash
pip install fastapi uvicorn sqlalchemy psycopg2-binary python-dotenv
```

**macOS:**
```bash
pip3 install fastapi uvicorn sqlalchemy psycopg2-binary python-dotenv
```

**Package breakdown:**
- `fastapi` - Web framework for building APIs
- `uvicorn` - ASGI server to run FastAPI
- `sqlalchemy` - ORM (Object-Relational Mapping) for database
- `psycopg2-binary` - PostgreSQL adapter for Python
- `python-dotenv` - Manage environment variables from .env file

---

### **Step 3.4: Create Project Structure**

Create this folder structure (use File Explorer/Finder or terminal):

```
fastapi-project/
│
├── venv/                      # Virtual environment (don't commit)
│
├── app/
│   ├── __init__.py           # Makes 'app' a Python package
│   ├── main.py               # FastAPI application entry point
│   ├── database.py           # Database configuration
│   ├── models.py             # SQLAlchemy models (database tables)
│   ├── schemas.py            # Pydantic schemas (validation)
│   ├── crud.py               # Database operations (Create, Read, Update, Delete)
│   │
│   └── routers/
│       ├── __init__.py
│       └── users.py          # User endpoints
│
├── .env                      # Environment variables (DON'T COMMIT)
├── .gitignore                # Git ignore file
└── requirements.txt          # Python dependencies
```

**Create files quickly (terminal):**

**Windows:**
```bash
mkdir app
mkdir app\routers
type nul > app\__init__.py
type nul > app\main.py
type nul > app\database.py
type nul > app\models.py
type nul > app\schemas.py
type nul > app\crud.py
type nul > app\routers\__init__.py
type nul > app\routers\users.py
type nul > .env
type nul > .gitignore
```

**macOS:**
```bash
mkdir -p app/routers
touch app/__init__.py
touch app/main.py
touch app/database.py
touch app/models.py
touch app/schemas.py
touch app/crud.py
touch app/routers/__init__.py
touch app/routers/users.py
touch .env
touch .gitignore
```

---

### **Step 3.5: Create `.gitignore`**

**Important:** Never commit sensitive data or virtual environments to Git!

Edit `.gitignore`:
```gitignore
# Virtual Environment
venv/
env/

# Python
__pycache__/
*.py[cod]
*$py.class
*.so

# Environment variables
.env
.env.local

# Database
*.db
*.sqlite3

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
```

---

## Part 4: Database Configuration

### **Step 4.1: Create `.env` File**

Edit `.env` file:
```bash
# Database Configuration
DATABASE_URL=postgresql://fastapi_user:SecurePass123!@localhost:5432/fastapi_db

# Application Settings
APP_NAME="FastAPI Backend"
DEBUG=True
```

**Connection String Breakdown:**
```
postgresql://[username]:[password]@[host]:[port]/[database_name]
              │          │           │       │     │
              │          │           │       │     └─ Database name
              │          │           │       └─────── Port (default: 5432)
              │          │           └─────────────── Host (localhost)
              │          └─────────────────────────── Password
              └────────────────────────────────────── Username
```

---

### **Step 4.2: Create `database.py`**

Edit `app/database.py`:
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

**Teaching Points:**
- `engine`: Connection to database
- `SessionLocal`: Factory for creating database sessions
- `Base`: Base class for all ORM models
- `get_db()`: Dependency injection - provides DB session to routes

---

## Part 5: Building the Backend

### **Step 5.1: Create Database Models**

Edit `app/models.py`:
```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from .database import Base

class User(Base):
    """
    User model - represents the 'users' table in PostgreSQL
    """
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    username = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

**Teaching Points:**
- `__tablename__`: Name of table in PostgreSQL
- `Column`: Defines table columns
- `primary_key=True`: Unique identifier for each row
- `index=True`: Creates database index for faster queries
- `unique=True`: Ensures no duplicate values
- `nullable=False`: Column cannot be empty
- `func.now()`: SQLAlchemy function for current timestamp

---

### **Step 5.2: Create Pydantic Schemas**

Edit `app/schemas.py`:
```python
from pydantic import BaseModel, EmailStr
from datetime import datetime
from typing import Optional

# Schema for creating a user (request body)
class UserCreate(BaseModel):
    """
    Schema for user creation - what client sends
    """
    email: str
    username: str
    password: str

# Schema for updating a user
class UserUpdate(BaseModel):
    """
    Schema for user updates - all fields optional
    """
    email: Optional[str] = None
    username: Optional[str] = None
    password: Optional[str] = None
    is_active: Optional[bool] = None

# Schema for reading a user (response body)
class User(BaseModel):
    """
    Schema for user response - what API returns
    Does NOT include password!
    """
    id: int
    email: str
    username: str
    is_active: bool
    created_at: datetime
    updated_at: Optional[datetime] = None

    class Config:
        # Allows Pydantic to work with SQLAlchemy models
        from_attributes = True
```

**Teaching Points:**
- Pydantic validates data automatically
- `UserCreate`: What we accept from client (includes password)
- `User`: What we return to client (no password!)
- Separation of concerns: input vs output
- `Config.from_attributes`: Converts SQLAlchemy objects to Pydantic

---

### **Step 5.3: Create CRUD Operations**

Edit `app/crud.py`:
```python
from sqlalchemy.orm import Session
from . import models, schemas
from typing import Optional, List

# CREATE
def create_user(db: Session, user: schemas.UserCreate):
    """
    Create a new user in the database
    
    In production: Use proper password hashing (bcrypt, passlib)
    """
    # WARNING: This is NOT secure - just for learning!
    fake_hashed_password = user.password + "_notreallyhashed"
    
    db_user = models.User(
        email=user.email,
        username=user.username,
        hashed_password=fake_hashed_password
    )
    db.add(db_user)       # Add to session
    db.commit()           # Save to database
    db.refresh(db_user)   # Refresh to get ID and timestamps
    return db_user

# READ - Single user by ID
def get_user(db: Session, user_id: int):
    """
    Get a user by ID
    Returns None if not found
    """
    return db.query(models.User).filter(models.User.id == user_id).first()

# READ - Single user by email
def get_user_by_email(db: Session, email: str):
    """
    Get a user by email
    Useful for checking if email already exists
    """
    return db.query(models.User).filter(models.User.email == email).first()

# READ - Single user by username
def get_user_by_username(db: Session, username: str):
    """
    Get a user by username
    """
    return db.query(models.User).filter(models.User.username == username).first()

# READ - Multiple users with pagination
def get_users(db: Session, skip: int = 0, limit: int = 100):
    """
    Get multiple users with pagination
    
    skip: Number of records to skip (for pagination)
    limit: Maximum number of records to return
    """
    return db.query(models.User).offset(skip).limit(limit).all()

# UPDATE
def update_user(db: Session, user_id: int, user_update: schemas.UserUpdate):
    """
    Update user fields
    Only updates fields that are provided
    """
    db_user = get_user(db, user_id)
    if not db_user:
        return None
    
    # Update only provided fields
    update_data = user_update.dict(exclude_unset=True)
    
    if "password" in update_data:
        # Hash the new password (not secure - just for learning!)
        update_data["hashed_password"] = update_data.pop("password") + "_notreallyhashed"
    
    for field, value in update_data.items():
        setattr(db_user, field, value)
    
    db.commit()
    db.refresh(db_user)
    return db_user

# DELETE
def delete_user(db: Session, user_id: int):
    """
    Delete a user by ID
    Returns the deleted user or None if not found
    """
    db_user = get_user(db, user_id)
    if db_user:
        db.delete(db_user)
        db.commit()
    return db_user
```

**Teaching Points:**
- CRUD = Create, Read, Update, Delete
- `db.add()`: Adds object to session (not saved yet)
- `db.commit()`: Saves changes to database
- `db.refresh()`: Reloads object from database (gets ID, timestamps)
- `.first()`: Returns first result or None
- `.all()`: Returns list of all results
- `offset()` and `limit()`: Pagination

---

### **Step 5.4: Create API Routes**

Edit `app/routers/users.py`:
```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List

from .. import crud, schemas
from ..database import get_db

# Create router
router = APIRouter(
    prefix="/users",           # All routes start with /users
    tags=["users"]            # Groups endpoints in documentation
)

# CREATE - Register new user
@router.post("/", response_model=schemas.User, status_code=status.HTTP_201_CREATED)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    """
    Create a new user
    
    - **email**: User's email (must be unique)
    - **username**: User's username (must be unique)
    - **password**: User's password (will be hashed)
    """
    # Check if email already exists
    db_user = crud.get_user_by_email(db, email=user.email)
    if db_user:
        raise HTTPException(
            status_code=400,
            detail="Email already registered"
        )
    
    # Check if username already exists
    db_user = crud.get_user_by_username(db, username=user.username)
    if db_user:
        raise HTTPException(
            status_code=400,
            detail="Username already taken"
        )
    
    return crud.create_user(db=db, user=user)

# READ - Get all users
@router.get("/", response_model=List[schemas.User])
def read_users(
    skip: int = 0,
    limit: int = 100,
    db: Session = Depends(get_db)
):
    """
    Retrieve all users with pagination
    
    - **skip**: Number of users to skip (default: 0)
    - **limit**: Maximum number of users to return (default: 100)
    """
    users = crud.get_users(db, skip=skip, limit=limit)
    return users

# READ - Get user by ID
@router.get("/{user_id}", response_model=schemas.User)
def read_user(user_id: int, db: Session = Depends(get_db)):
    """
    Get a specific user by ID
    
    - **user_id**: The ID of the user to retrieve
    """
    db_user = crud.get_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user

# UPDATE - Update user
@router.put("/{user_id}", response_model=schemas.User)
def update_user(
    user_id: int,
    user_update: schemas.UserUpdate,
    db: Session = Depends(get_db)
):
    """
    Update a user's information
    
    - **user_id**: The ID of the user to update
    - Only provided fields will be updated
    """
    db_user = crud.update_user(db, user_id=user_id, user_update=user_update)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user

# DELETE - Delete user
@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_user(user_id: int, db: Session = Depends(get_db)):
    """
    Delete a user
    
    - **user_id**: The ID of the user to delete
    """
    db_user = crud.delete_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return None
```

**Teaching Points:**
- `@router.post()`, `@router.get()`: HTTP methods
- `response_model`: Validates and formats response
- `Depends(get_db)`: Dependency injection for database session
- `HTTPException`: Returns error responses
- Status codes: 200 (OK), 201 (Created), 204 (No Content), 404 (Not Found)
- Path parameters: `{user_id}`
- Query parameters: `skip`, `limit`

---

### **Step 5.5: Create Main Application**

Edit `app/main.py`:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from .database import engine, Base
from .routers import users

# Create all database tables
# This reads all models and creates tables in PostgreSQL
Base.metadata.create_all(bind=engine)

# Initialize FastAPI application
app = FastAPI(
    title="Student Project API",
    description="REST API built with FastAPI and SQLAlchemy",
    version="1.0.0",
    docs_url="/docs",          # Swagger UI
    redoc_url="/redoc"         # ReDoc UI
)

# Configure CORS (Cross-Origin Resource Sharing)
# Allows frontend applications to access the API
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],        # In production: specify exact origins
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(users.router)

# Root endpoint
@app.get("/")
def root():
    """
    Root endpoint - API health check
    """
    return {
        "message": "Welcome to FastAPI Backend",
        "status": "running",
        "docs": "/docs"
    }

# Health check endpoint
@app.get("/health")
def health_check():
    """
    Health check endpoint for monitoring
    """
    return {"status": "healthy"}
```

**Teaching Points:**
- `Base.metadata.create_all()`: Creates all tables in PostgreSQL
- CORS: Allows browser-based frontends to access API
- `include_router()`: Adds route handlers
- `/docs`: Interactive API documentation (Swagger UI)
- `/redoc`: Alternative API documentation

---

### **Step 5.6: Create `requirements.txt`**

```bash
# Generate requirements file
pip freeze > requirements.txt
```

Your `requirements.txt` should contain:
```
fastapi==0.109.0
uvicorn==0.27.0
sqlalchemy==2.0.25
psycopg2-binary==2.9.9
python-dotenv==1.0.0
```

---

## Part 6: Running & Testing

### **Step 6.1: Verify Database Connection**

Create a test file `test_connection.py` in project root:
```python
from app.database import engine
from sqlalchemy import text

print("Testing database connection...")

try:
    with engine.connect() as connection:
        result = connection.execute(text("SELECT version();"))
        version = result.fetchone()
        print(f"✅ Connected to PostgreSQL!")
        print(f"Version: {version[0]}")
except Exception as e:
    print(f"❌ Connection failed: {e}")
```

Run:
```bash
python test_connection.py
```

---

### **Step 6.2: Start the Server**

**Both Windows and macOS:**
```bash
uvicorn app.main:app --reload
```

You should see:
```
INFO:     Uvicorn running on http://127.0.0.1:8000
INFO:     Application startup complete.
```

**What `--reload` does:** Auto-restarts server when you change code (development only)

---

### **Step 6.3: Access the API**

Open your browser:

1. **API Root**: http://localhost:8000
2. **Interactive Docs (Swagger UI)**: http://localhost:8000/docs
3. **Alternative Docs (ReDoc)**: http://localhost:8000/redoc

---

### **Step 6.4: Test Endpoints**

#### **Method 1: Using Swagger UI (Easiest for beginners)**

1. Go to http://localhost:8000/docs
2. Click on **POST /users/**
3. Click **"Try it out"**
4. Enter test data:
```json
{
  "email": "student@example.com",
  "username": "student1",
  "password": "password123"
}
```
5. Click **"Execute"**
6. See the response below

#### **Method 2: Using curl (Command line)**

**Create a user:**
```bash
curl -X POST "http://localhost:8000/users/" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "username": "johndoe",
    "password": "secret123"
  }'
```

**Get all users:**
```bash
curl "http://localhost:8000/users/"
```

**Get specific user:**
```bash
curl "http://localhost:8000/users/1"
```

**Update user:**
```bash
curl -X PUT "http://localhost:8000/users/1" \
  -H "Content-Type: application/json" \
  -d '{
    "is_active": false
  }'
```

**Delete user:**
```bash
curl -X DELETE "http://localhost:8000/users/1"
```

#### **Method 3: Using Python requests**

Create `test_api.py`:
```python
import requests

BASE_URL = "http://localhost:8000"

# Create user
response = requests.post(
    f"{BASE_URL}/users/",
    json={
        "email": "test@example.com",
        "username": "testuser",
        "password": "testpass123"
    }
)
print("Create user:", response.json())

# Get all users
response = requests.get(f"{BASE_URL}/users/")
print("All users:", response.json())

# Get specific user
response = requests.get(f"{BASE_URL}/users/1")
print("User 1:", response.json())
```

Install requests first:
```bash
pip install requests
```

Run:
```bash
python test_api.py
```

---

### **Step 6.5: Verify Data in PostgreSQL**

```bash
# Connect to database
psql -U fastapi_user -d fastapi_db

# View users table
SELECT * FROM users;

# Exit
\q
```

---

## Part 7: Next Steps & Best Practices

### **Important Improvements for Students**

#### **1. Password Hashing** (Security - HIGH PRIORITY)

Install:
```bash
pip install passlib bcrypt
```

Update `app/crud.py`:
```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def create_user(db: Session, user: schemas.UserCreate):
    # SECURE password hashing
    hashed_password = pwd_context.hash(user.password)
    
    db_user = models.User(
        email=user.email,
        username=user.username,
        hashed_password=hashed_password  # Now properly hashed!
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

def verify_password(plain_password: str, hashed_password: str):
    """Verify password against hash"""
    return pwd_context.verify(plain_password, hashed_password)
```

---

#### **2. Authentication with JWT Tokens**

Install:
```bash
pip install python-jose[cryptography]
```

Create `app/auth.py`:
```python
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext

SECRET_KEY = "your-secret-key-change-this"  # Move to .env!
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt
```

---

#### **3. Database Migrations with Alembic**

Install:
```bash
pip install alembic
```

Initialize:
```bash
alembic init alembic
```

Configure `alembic.ini`:
```ini
# Replace this line
sqlalchemy.url = driver://user:pass@localhost/dbname

# With this (or use env variable)
sqlalchemy.url = postgresql://fastapi_user:SecurePass123!@localhost:5432/fastapi_db
```

Edit `alembic/env.py`:
```python
from app.database import Base
from app import models  # Import your models

target_metadata = Base.metadata
```

Create first migration:
```bash
alembic revision --autogenerate -m "Initial migration"
alembic upgrade head
```

---

#### **4. Environment Variables Best Practices**

Update `.env`:
```bash
# Database
DATABASE_URL=postgresql://fastapi_user:SecurePass123!@localhost:5432/fastapi_db

# Security
SECRET_KEY=your-super-secret-key-min-32-characters
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# Application
APP_NAME=FastAPI Backend
DEBUG=True
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:8080
```

---

#### **5. Testing with pytest**

Install:
```bash
pip install pytest httpx
```

Create `tests/test_users.py`:
```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_user():
    response = client.post(
        "/users/",
        json={
            "email": "test@example.com",
            "username": "testuser",
            "password": "testpass"
        }
    )
    assert response.status_code == 201
    assert response.json()["email"] == "test@example.com"

def test_read_users():
    response = client.get("/users/")
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

Run tests:
```bash
pytest
```

---

#### **6. Add Logging**

Update `app/main.py`:
```python
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

@app.on_event("startup")
async def startup_event():
    logger.info("Application startup")

@app.on_event("shutdown")
async def shutdown_event():
    logger.info("Application shutdown")
```

---

#### **7. Database Relationships**

Example: Users and Posts

`app/models.py`:
```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    
    # Relationship
    posts = relationship("Post", back_populates="owner")

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, index=True)
    content = Column(String)
    user_id = Column(Integer, ForeignKey("users.id"))
    
    # Relationship
    owner = relationship("User", back_populates="posts")
```

---

### **Common Issues & Solutions**

**Issue 1: "ModuleNotFoundError: No module named 'app'"**
- Solution: Make sure you're in the project root directory
- Run: `uvicorn app.main:app --reload` (not inside the app folder)

**Issue 2: "Could not connect to PostgreSQL"**
- Check PostgreSQL is running: `brew services list` (macOS) or Services (Windows)
- Verify credentials in `.env` file
- Test connection: `psql -U fastapi_user -d fastapi_db`

**Issue 3: "Table already exists"**
- Solution: Drop and recreate tables (development only!)
```python
# In Python console
from app.database import engine, Base
Base.metadata.drop_all(bind=engine)
Base.metadata.create_all(bind=engine)
```

**Issue 4: Port 8000 already in use**
- Solution: Use different port
```bash
uvicorn app.main:app --reload --port 8001
```

---

### **Deployment Checklist**

Before deploying to production:

- [ ] Use proper password hashing (bcrypt)
- [ ] Implement JWT authentication
- [ ] Set up database migrations (Alembic)
- [ ] Use environment variables for all secrets
- [ ] Set up proper CORS origins (not `["*"]`)
- [ ] Add request rate limiting
- [ ] Implement proper logging
- [ ] Write tests
- [ ] Use HTTPS
- [ ] Set up monitoring and error tracking

---

### **Learning Resources for Students**

1. **FastAPI Documentation**: https://fastapi.tiangolo.com/
2. **SQLAlchemy Documentation**: https://docs.sqlalchemy.org/
3. **PostgreSQL Tutorial**: https://www.postgresqltutorial.com/
4. **REST API Best Practices**: https://restfulapi.net/
5. **JWT Authentication**: https://jwt.io/

---

### **Project Ideas for Students**

1. **Blog API** - Users, Posts, Comments, Tags
2. **Task Manager** - Users, Projects, Tasks, Due Dates
3. **E-commerce** - Products, Orders, Cart, Payments
4. **Social Media** - Users, Posts, Likes, Followers
5. **Learning Management** - Courses, Lessons, Quizzes, Grades

---

This complete guide should give your students everything they need to build a production-ready REST API backend! Would you like me to expand on any specific section, such as authentication, testing, or deployment?