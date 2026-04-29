# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A simple social network application with a web-based UI backed by Amazon DocumentDB for persistence. Users can create profiles, make posts, follow other users, and view a timeline of posts from users they follow.

## Tech Stack

- **Backend**: Python with Flask
- **Frontend**: Jinja2 templates with vanilla CSS (server-side rendered)
- **Database**: Amazon DocumentDB (MongoDB 5.0 compatible)
- **Driver**: PyMongo (`pymongo` package)

## Architecture

```
social/
├── app.py                 # Flask app entry point
├── requirements.txt
├── .env                   # Connection string and config (not committed)
├── routes/
│   ├── auth.py            # Login, register, logout
│   ├── users.py           # Profile, follow/unfollow
│   ├── posts.py           # Create, delete, timeline
│   └── search.py          # User search
├── models/
│   ├── db.py              # DocumentDB connection management
│   ├── user.py            # User data access
│   └── post.py            # Post data access
├── templates/
│   ├── layout.html        # Base layout
│   ├── login.html
│   ├── register.html
│   ├── timeline.html
│   ├── profile.html
│   └── search.html
├── static/
│   └── css/
│       └── style.css
└── scripts/
    └── seed.py            # Seed data for development
```

## Build and Run Commands

```bash
pip install -r requirements.txt   # Install dependencies
flask run                         # Start the dev server
python app.py                     # Start the server directly
python scripts/seed.py            # Seed the database with sample data
```

## DocumentDB Specifics

- Connect using PyMongo with `tls=True` and the Amazon RDS CA bundle
- Connection string format: `mongodb://<user>:<pass>@<cluster-endpoint>:27017/?tls=true&tlsCAFile=global-bundle.pem&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false`
- **retryWrites must be false** — DocumentDB does not support retryable writes
- Use `readPreference=secondaryPreferred` for read-heavy timeline queries
- DocumentDB does not support `$lookup` with `let`/`pipeline` syntax — use only the basic equality form
- Change streams require explicit enablement on the cluster
- Transactions are supported but limited to 1MB of data and 100ms of lock wait time

## Collections

- **users**: `{ _id, username, email, password_hash, display_name, bio, followers: [user_id], following: [user_id], created_at }`
- **posts**: `{ _id, author_id, content, created_at }`

### Indexes

```javascript
// users
db.users.createIndex({ username: 1 }, { unique: true })
db.users.createIndex({ email: 1 }, { unique: true })

// posts
db.posts.createIndex({ authorId: 1, createdAt: -1 })
db.posts.createIndex({ createdAt: -1 })
```

## Key Conventions

- Use Flask Blueprints for route organization
- Passwords hashed with `werkzeug.security` (`generate_password_hash` / `check_password_hash`)
- Sessions managed via Flask's built-in `session` with a secret key
- All database access goes through the `models/` layer — routes never call the driver directly
- Environment variables via `.env` file using `python-dotenv`
- No ODM — use PyMongo directly

## Environment Variables

```
DOCDB_URI=mongodb://<user>:<pass>@<endpoint>:27017/?tls=true&tlsCAFile=global-bundle.pem&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false
SESSION_SECRET=<random-string>
FLASK_APP=app.py
FLASK_PORT=3000
```

## Constraints

- Do not use MongoEngine or other ODMs — use PyMongo directly
- Do not use `$lookup` with pipeline syntax — DocumentDB doesn't support it
- Do not enable retryable writes
- Keep the UI simple — no frontend frameworks, no build step for CSS/JS
- All queries must work within DocumentDB's supported MongoDB 5.0 API subset
