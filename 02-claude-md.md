# Tutorial 02: Project Requirements & CLAUDE.md

## Goal

By the end of this tutorial you will have a `CLAUDE.md` file at the root of your project that documents the project idea, functional requirements, non-functional requirements, and a reference to the database schema.

---

## Prerequisites

- Tutorial 01 (environment setup) is complete

---

## Concepts Introduced

- What `CLAUDE.md` is and how Claude Code uses it
- Functional vs non-functional requirements

---

## Step 1: What Is CLAUDE.md?

`CLAUDE.md` is a markdown file that Claude Code reads automatically at the start of every session. It gives Claude persistent context about your project — what it is, how it's structured, what rules to follow, and what you're trying to build.

Think of it as the project brief that you and Claude share. The better it is, the more useful Claude's assistance will be.

---

## Step 2: Create CLAUDE.md

Create a file named `CLAUDE.md` at the root of your project with the following structure:

```markdown
# [Your Project Name]

## 1. Project Idea

[A short paragraph describing what this system does and who it is for.]

---

## 2. Functional Requirements

[List what the system must do.]

---

## 3. Non-Functional Requirements

[List quality and constraint requirements.]
```

---

## Step 3: Fill In the Project Idea

Write 2–3 sentences describing:
- What problem the system solves
- Who the users are
- What the system allows them to do

**Example:**
> A web system for a university faculty that allows staff to report maintenance issues in rooms and reserve backup rooms while their primary room is being repaired. Admins manage the room schedule and handle incoming requests. Faculty and staff interact through a self-serve interface.

---

## Step 4: Define Functional Requirements

Functional requirements describe **what the system must do** — its features and behaviours.

Write them as a numbered list. Each item should be a concrete, testable capability.

**Example:**
```
1. Users can register and log in with an email and password.
2. Authenticated users can submit a maintenance request for a room.
3. Admins can view, update, and close maintenance requests.
4. Users can reserve an available room for a specific date and time slot.
5. The system prevents double-booking of the same room at the same time.
6. Admins can manage the weekly room schedule for the semester.
```

Write your own list based on the system you are building.

---

## Step 5: Define Non-Functional Requirements

Non-functional requirements describe **how well the system must work** — quality, constraints, and operational expectations. They don't describe features, they describe properties.

Common categories:

| Category | Example |
|---|---|
| Security | Passwords must be stored hashed, never in plain text |
| Performance | API responses should return within 500ms under normal load |
| Usability | The API must provide clear error messages for invalid input |
| Maintainability | Code must follow a consistent structure across all modules |
| Reliability | The system should handle database connection errors gracefully |

Write a short list of non-functional requirements relevant to your project.

---

## Step 6: Add a Database Schema Reference

Your project includes a `schema.md` file that documents all database tables, columns, types, and relationships. Add a section to `CLAUDE.md` that points Claude to it:

```markdown
## 4. Database Schema

See [schema.md](schema.md) for the full database schema — tables, column definitions, constraints, and relationships.
```

This matters because when Claude helps you write models, queries, or API endpoints, it needs to know the schema. A direct reference is more reliable than expecting Claude to find the file on its own.

---

## Summary

You have:
- Created a `CLAUDE.md` file that gives Claude persistent context about your project
- Defined what the system does and who uses it
- Listed the system's functional requirements (what it must do)
- Listed the system's non-functional requirements (how well it must do it)
- Added a reference to `schema.md` so Claude always knows where to find the database schema

**What comes next:** Tutorial 03 will cover SQLAlchemy ORM and defining the database models.
