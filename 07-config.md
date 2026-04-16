# Tutorial 07: Environment Configuration

## Goal

By the end of this tutorial you will have created `app/config.py`, a central place that reads configuration values from the environment. This keeps secrets out of your code and makes the project easy to configure across different machines.

---

## Prerequisites

- Tutorial 06 (HTTP API) is complete

---

## Concepts Introduced

- Environment variables
- `.env` files and why they are never committed
- Reading environment variables with `os.getenv`
- `python-dotenv` for loading `.env` files automatically

---

## Step 1: What Is an Environment Variable?

An **environment variable** is a value set outside your code — in the shell, a `.env` file, or a deployment platform — that your program reads at runtime.

The classic example is a database password or a secret key. You never want these inside your source code because:

- Your code is often shared (version control, colleagues, public GitHub)
- Different environments — your laptop, a staging server, production — need different values
- Rotating a secret should not require changing and redeploying code

Environment variables solve this cleanly: the code says *"read this value from the environment"*, and whoever runs the program supplies the value.

---

## Step 2: The `.env` File

Setting environment variables in the shell every time you start the server is inconvenient. The convention is to put them in a `.env` file at the root of the project:

```
SECRET_KEY=some-random-string-here
FRONTEND_BASE_URL=http://localhost:5173
```

This file is **never committed to version control**. Add it to `.gitignore`:

```
.env
```

Each developer creates their own `.env` locally. In production, the values are set directly in the hosting environment (as actual environment variables), not via a `.env` file.

---

## Step 3: `python-dotenv`

Python doesn't read `.env` files automatically — that's what the `python-dotenv` package is for. It reads the file and loads each line as an environment variable before your code accesses them.

You use it with a single call:

```python
from dotenv import load_dotenv

load_dotenv()
```

After `load_dotenv()` runs, any values in `.env` are available through `os.getenv`. If a variable was already set in the shell, `load_dotenv` leaves it alone — the real environment always takes priority.

---

## Step 4: `os.getenv` with a Default

`os.getenv(key, default)` reads an environment variable. If the variable isn't set, it returns `default` instead of raising an error.

```python
import os

SECRET_KEY = os.getenv("SECRET_KEY", "change-this-in-production")
```

This means:
- In development with no `.env`, the default applies — the app still starts
- In production, you set the real value and the default is never used

The default is intentionally descriptive (`"change-this-in-production"`) rather than a real secret, so it is obvious if someone forgets to set the variable properly.

---

## Step 5: Create `app/config.py`

```python
import os
from dotenv import load_dotenv

load_dotenv()

SECRET_KEY = os.getenv("SECRET_KEY", "change-this-in-production")
FRONTEND_BASE_URL = os.getenv("FRONTEND_BASE_URL", "http://localhost:5173")
```

`SECRET_KEY` will be used in the next tutorial to sign JWT tokens. `FRONTEND_BASE_URL` will be used later to generate room QR codes that link to the correct frontend URL.

Any module that needs a config value imports it directly:

```python
from .config import SECRET_KEY
```

---

## Step 6: Create a `.env` File

At the root of the project, create a file named `.env`:

```
SECRET_KEY=replace-this-with-a-long-random-string
```

You can generate a suitable secret key from the terminal:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

Copy the output into your `.env` file. Leave `FRONTEND_BASE_URL` out for now — the default `http://localhost:5173` is correct for local development.

---

## Summary

You have:

- Understood why secrets must not live in source code
- Created a `.env` file for local configuration and added it to `.gitignore`
- Used `python-dotenv` to load it automatically
- Created `app/config.py` as a central place for configuration values

**What comes next:** Tutorial 08 will use `SECRET_KEY` to implement JWT-based authentication.
