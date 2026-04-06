# Tutorial 01: Environment Setup

## Goal

By the end of this tutorial you will have Python installed, a PostgreSQL database running, a virtual environment created, and all project dependencies installed — everything needed to start building the backend.

---

## Prerequisites

- None. This is the first tutorial.

---

## Concepts Introduced

- Python virtual environments and why they matter
- Installing packages with `pip`
- What PostgreSQL is and how to create a database for the project

---

## Step 1: Install Python

**Windows**
1. Download Python 3.11+ from [python.org](https://python.org)
2. Run the installer — check **"Add Python to PATH"** before clicking Install
3. Verify:
```
python --version
```

**macOS**
```bash
brew install python@3.11
python3 --version
```

> **Note:** If Homebrew is not installed, get it from [brew.sh](https://brew.sh).

---

## Step 2: Install VS Code

Download and install VS Code from [code.visualstudio.com](https://code.visualstudio.com), then install the **Python** extension from the Extensions panel.

---

## Step 3: Install and Configure PostgreSQL

**Windows**
1. Download the installer from [postgresql.org/download/windows](https://www.postgresql.org/download/windows/)
2. Run it, set a password for the `postgres` user (write it down), and keep the default port `5432`
3. If `psql` is not found after installation, add `C:\Program Files\PostgreSQL\16\bin` to your PATH and restart the terminal

**macOS**
```bash
brew install postgresql@15
brew services start postgresql@15
```

### Create the database and user

**Windows** — open a terminal and run:
```bash
psql -U postgres
```

**macOS** — open a terminal and run:
```bash
psql postgres
```

Then in the PostgreSQL prompt run these commands:

```sql
CREATE DATABASE fcitr_db;
CREATE USER fcitr_user WITH PASSWORD 'password123';
GRANT ALL PRIVILEGES ON DATABASE fcitr_db TO fcitr_user;
\c fcitr_db
GRANT ALL ON SCHEMA public TO fcitr_user;
\q
```

Verify the connection:
```bash
psql -U fcitr_user -d fcitr_db -h localhost
```

You should see the `fcitr_db=>` prompt. Type `\q` to exit.

---

## Step 4: Create the Project Structure

Create the following folders and empty files:

**Windows**
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
type nul > .env
type nul > .gitignore
```

**macOS**
```bash
mkdir -p app/routers
touch app/__init__.py app/main.py app/database.py app/models.py app/schemas.py app/crud.py
touch app/routers/__init__.py
touch .env .gitignore
```

Your project should look like this:
```
fcitr-maint/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── database.py
│   ├── models.py
│   ├── schemas.py
│   ├── crud.py
│   └── routers/
│       └── __init__.py
├── .env
└── .gitignore
```

---

## Step 5: Create a Virtual Environment

A virtual environment keeps this project's dependencies isolated from other Python projects on your machine.

**Windows**
```bash
python -m venv venv
venv\Scripts\activate
```

**macOS**
```bash
python3 -m venv venv
source venv/bin/activate
```

You should see `(venv)` in your terminal prompt. You must activate the virtual environment every time you open a new terminal to work on this project.

---

## Step 6: Install Dependencies

With the virtual environment active, install the required packages:

```bash
pip install fastapi uvicorn sqlalchemy psycopg2-binary python-dotenv
```

| Package | Purpose |
|---|---|
| `fastapi` | Web framework for building the API |
| `uvicorn` | Server that runs the FastAPI application |
| `sqlalchemy` | ORM for talking to the database |
| `psycopg2-binary` | PostgreSQL driver (connects SQLAlchemy to PostgreSQL) |
| `python-dotenv` | Loads environment variables from a `.env` file |

Save the installed packages to a requirements file:
```bash
pip freeze > requirements.txt
```

---

## Step 7: Configure Environment Variables

Edit `.env` and add your database connection string:

```
DATABASE_URL=postgresql://fcitr_user:password123@localhost:5432/fcitr_db
```

This tells SQLAlchemy how to connect to the database. The format is:
```
postgresql://[user]:[password]@[host]:[port]/[database]
```

Edit `.gitignore` to make sure you never commit sensitive files:
```
venv/
__pycache__/
*.py[cod]
.env
.DS_Store
```

---

## Summary

You have:
- Installed Python and VS Code
- Set up a PostgreSQL database with a dedicated user
- Created the project folder structure
- Created and activated a virtual environment
- Installed all project dependencies and saved them to `requirements.txt`
- Configured the `.env` file with the database connection string

**What comes next:** Tutorial 02 will cover SQLAlchemy ORM and defining the database models.
