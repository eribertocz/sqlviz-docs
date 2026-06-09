# SQLviz — Modes & CLI
**Version:** v0.1.0 (Draft)
**Status:** Work in Progress
**Last Updated:** 2026-06-08

---

## 1. Project Modes

SQLviz operates in three modes.
Each mode is designed for a specific use case.
The code is identical across all modes — only the storage changes.

### Mode 1 — Demo Mode

```
sqlviz
```

No arguments. No files. No configuration.
SQLviz starts instantly with an in-memory SQLite database.

```
Purpose:       Try SQLviz without creating any files
Storage:       SQLite :memory: — disappears on exit
Persistence:   None — all data is lost when SQLviz closes
Use case:      First time users, testing, presentations
```

Demo mode is not a limited version of SQLviz.
It is the full system running in memory.
Every feature available in persistent mode works in demo mode.

### Mode 2 — Persistent Mode

```
sqlviz my_project
sqlviz path/to/my_project
sqlviz C:\analytics\company_dashboard
```

One argument — the project name or path.
SQLviz creates or opens a `.sqlviz` file.

```
Purpose:       Real work, saved projects
Storage:       SQLite file on disk (.sqlviz extension)
Persistence:   Full — survives restarts
Use case:      Daily analytical work
```

### Mode 3 — Cloud Mode (Future — v0.4+)

```
sqlviz --cloud my_organization
```

Not yet implemented. Planned for v0.4.

```
Purpose:       Team collaboration, shared dashboards
Storage:       TursoDB or equivalent distributed SQLite
Persistence:   Full — synchronized across machines
Use case:      Teams sharing dashboards and insights
```

---

## 2. The CLI Design

SQLviz CLI is designed after DuckDB.
The simplest possible interface.

### The complete CLI specification

```bash
# Demo mode — no files, no arguments
sqlviz

# Persistent mode — create or open project in current directory
sqlviz my_project              # creates my_project.sqlviz
sqlviz my_project              # opens existing my_project.sqlviz

# Persistent mode — explicit path
sqlviz path/to/my_project      # creates path/to/my_project.sqlviz
sqlviz C:\work\company         # creates C:\work\company.sqlviz

# Persistent mode — explicit .sqlviz extension (same result)
sqlviz my_project.sqlviz       # opens or creates my_project.sqlviz
```

### The CLI decision tree

```
sqlviz called
    │
    ├── No arguments
    │       │
    │       └── Start in demo mode (:memory:)
    │           → Print: "Demo mode — data will not be saved"
    │           → Open browser: http://localhost:4000
    │
    └── Has argument (name or path)
            │
            ├── Argument ends in .sqlviz
            │       │
            │       ├── File exists + valid SQLviz signature
            │       │       └── Open existing project
            │       │           → Print: "Project: my_project"
            │       │
            │       ├── File exists + invalid signature
            │       │       └── Error: "Not a valid SQLviz project"
            │       │           → Exit cleanly
            │       │
            │       └── File does not exist
            │               └── Create new project
            │                   → Print: "Project created: my_project.sqlviz"
            │
            └── Argument has no extension
                    │
                    └── Add .sqlviz extension automatically
                        → Apply same logic as above
```

### CLI output format

```bash
# Demo mode
$ sqlviz
  ███████╗ ██████╗ ██╗     ██╗   ██╗██╗███████╗
  ...
  Demo mode — memory only, no files created
  Opening browser: http://localhost:4000

# New project
$ sqlviz company_analytics
  ███████╗ ██████╗ ██╗     ██╗   ██╗██╗███████╗
  ...
  Project created: company_analytics.sqlviz
  Opening browser: http://localhost:4000

# Existing project
$ sqlviz company_analytics
  ███████╗ ██████╗ ██╗     ██╗   ██╗██╗███████╗
  ...
  Project: company_analytics
  Opening browser: http://localhost:4000

# Invalid file
$ sqlviz some_database.sqlviz
  ███████╗ ██████╗ ██╗     ██╗   ██╗██╗███████╗
  ...
  Error: some_database.sqlviz is not a valid SQLviz project.
```

### Why no subcommands

DuckDB uses no subcommands. SQLviz follows the same philosophy.

```bash
# DuckDB
duckdb                    # in-memory
duckdb my_database.duckdb # open or create file

# SQLviz — identical philosophy
sqlviz                    # demo mode
sqlviz my_project         # open or create project
```

Subcommands add cognitive overhead.
The user should not need to remember `sqlviz run` vs `sqlviz start` vs `sqlviz open`.
One command. One behavior. Done.

---

## 3. The .sqlviz File

### What is a .sqlviz file

A `.sqlviz` file is a standard SQLite database
with a specific schema and a validation signature.

```
my_project.sqlviz
    │
    ├── SQLite database (standard format)
    ├── SQLviz schema (dashboards, panels, rows, filters)
    └── _sqlviz_meta table (validation signature)
```

The `.sqlviz` extension is cosmetic.
SQLite opens any file regardless of extension.
The extension signals to the user and to SQLviz
that this file is a SQLviz project.

### Why a single file

```
✅ One file = one project
   Copy it, share it, back it up — just one file

✅ No folders, no loose SQL files
   Everything lives in the .sqlviz file

✅ Git friendly
   git add my_project.sqlviz
   git commit -m "add revenue dashboard"

✅ Identical to DuckDB philosophy
   Users already understand "one file = one database"
```

### The _sqlviz_meta signature

Every `.sqlviz` file contains a `_sqlviz_meta` table
that identifies it as a legitimate SQLviz project.

```sql
CREATE TABLE IF NOT EXISTS _sqlviz_meta (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

INSERT OR IGNORE INTO _sqlviz_meta VALUES ('app',     'sqlviz');
INSERT OR IGNORE INTO _sqlviz_meta VALUES ('version', '0.1.0');
INSERT OR IGNORE INTO _sqlviz_meta VALUES ('created', '2026-06-08T00:00:00Z');
```

### Validation logic

SQLviz uses a two-layer validation before opening any file:

```
Layer 1 — Extension check (fast)
    File must end in .sqlviz
    Filters 99% of accidental files

Layer 2 — Signature check (reliable)
    Open the file with SQLite
    Query _sqlviz_meta WHERE key = 'app'
    Value must equal 'sqlviz'
    Protects against renamed .sqlite files
```

```python
def is_sqlviz_project(path: Path) -> bool:
    """
    Returns True only if the file is a legitimate SQLviz project.
    Two-layer validation: extension + signature.
    """
    # Layer 1: extension
    if path.suffix != '.sqlviz':
        return False

    # Layer 2: signature
    try:
        with sqlite3.connect(str(path)) as conn:
            row = conn.execute(
                "SELECT value FROM _sqlviz_meta WHERE key = 'app'"
            ).fetchone()
            return row is not None and row[0] == 'sqlviz'
    except Exception:
        return False
```

### The .sqlviz schema

```sql
-- Project signature
CREATE TABLE _sqlviz_meta (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

-- Folder organization
CREATE TABLE folders (
    id         TEXT PRIMARY KEY,
    name       TEXT NOT NULL,
    parent_id  TEXT,
    sort_order INTEGER DEFAULT 0,
    created_at TEXT NOT NULL
);

-- Dashboards
CREATE TABLE dashboards (
    id         TEXT PRIMARY KEY,
    name       TEXT NOT NULL,
    folder_id  TEXT,
    visibility TEXT DEFAULT 'private',
    sort_order INTEGER DEFAULT 0,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    FOREIGN KEY (folder_id) REFERENCES folders(id)
);

-- Dashboard rows (layout)
CREATE TABLE rows (
    id           TEXT PRIMARY KEY,
    dashboard_id TEXT NOT NULL,
    width        INTEGER DEFAULT 100,
    height       INTEGER DEFAULT 0,
    align        TEXT DEFAULT 'left',
    sort_order   INTEGER DEFAULT 0,
    FOREIGN KEY (dashboard_id) REFERENCES dashboards(id)
);

-- Panels (the core unit)
CREATE TABLE panels (
    id           TEXT PRIMARY KEY,
    row_id       TEXT NOT NULL,
    name         TEXT NOT NULL,
    sql_content  TEXT DEFAULT '',   -- SQL lives here, not on disk
    width        INTEGER DEFAULT 100,
    chart_type   TEXT DEFAULT 'table',
    engine       TEXT DEFAULT 'duckdb',
    viz_engine   TEXT DEFAULT 'echarts',
    colors       TEXT,
    reactive     INTEGER DEFAULT 1,
    emits_clicks INTEGER DEFAULT 1,
    col_index    INTEGER DEFAULT 0,
    created_at   TEXT NOT NULL,
    FOREIGN KEY (row_id) REFERENCES rows(id)
);

-- Filter memory (last used values)
CREATE TABLE filter_memory (
    dashboard_id TEXT NOT NULL,
    variable     TEXT NOT NULL,
    value        TEXT,
    updated_at   TEXT NOT NULL,
    PRIMARY KEY (dashboard_id, variable)
);

-- Settings
CREATE TABLE settings (
    key        TEXT PRIMARY KEY,
    value      TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

---

## 4. Project Lifecycle

### Creating a new project

```
1. User runs: sqlviz my_project
2. SQLviz checks: does my_project.sqlviz exist?
3. File does not exist → create new project
4. SQLviz creates my_project.sqlviz
5. SQLviz initializes schema (all tables)
6. SQLviz inserts _sqlviz_meta signature
7. SQLviz creates example dashboard with 4 panels
8. SQLviz starts FastAPI server on port 4000
9. SQLviz opens browser at http://localhost:4000
10. User sees the example dashboard immediately
```

### Opening an existing project

```
1. User runs: sqlviz my_project
2. SQLviz checks: does my_project.sqlviz exist?
3. File exists → validate signature
4. Signature valid → open project
5. SQLviz runs migrations if schema is outdated
6. SQLviz starts FastAPI server on port 4000
7. SQLviz opens browser at http://localhost:4000
8. User sees their dashboards immediately
```

### Migrations

When SQLviz is updated, the schema may change.
Migrations run automatically on every project open.

```python
MIGRATIONS = [
    ("001", "ALTER TABLE panels ADD COLUMN sql_content TEXT DEFAULT ''"),
    ("002", "ALTER TABLE dashboards ADD COLUMN visibility TEXT DEFAULT 'private'"),
    # new migrations added here with each SQLviz version
]

def run_migrations(conn: sqlite3.Connection) -> None:
    conn.execute("""
        CREATE TABLE IF NOT EXISTS schema_migrations (
            id         TEXT PRIMARY KEY,
            applied_at TEXT NOT NULL
        )
    """)
    applied = {row[0] for row in conn.execute(
        "SELECT id FROM schema_migrations"
    ).fetchall()}

    for migration_id, sql in MIGRATIONS:
        if migration_id not in applied:
            try:
                conn.execute(sql)
                conn.execute(
                    "INSERT INTO schema_migrations VALUES (?, datetime('now'))",
                    (migration_id,)
                )
                conn.commit()
            except Exception as e:
                # Migration failed — log but continue
                print(f"Migration {migration_id} skipped: {e}")
```

### Closing a project

```
1. User closes the browser or presses Ctrl+C in terminal
2. SQLviz receives shutdown signal
3. SQLviz flushes any pending writes
4. SQLviz closes SQLite connection
5. Process exits cleanly
```

All data is already persisted — SQLite writes on every operation.
No data is lost on unexpected shutdown.

### Sharing a project

```
# Share with a colleague
cp my_project.sqlviz /shared/drive/

# Or via Git
git add my_project.sqlviz
git commit -m "share revenue dashboard"
git push

# Colleague opens it
sqlviz my_project.sqlviz
```

One file. That is all.

---

## 5. Demo Mode Deep Dive

Demo mode is a first-class citizen, not an afterthought.

### The singleton connection

The critical implementation detail of demo mode:

```
SQLite :memory: creates a new empty database
for every new connection.

If FastAPI opens a new connection per request,
every request sees an empty database.
This is the most common bug in SQLviz demo mode.

The solution: one global singleton connection
shared across all requests for the lifetime
of the process.
```

```python
_singleton_connection: sqlite3.Connection | None = None

def get_connection(db_path: str) -> sqlite3.Connection:
    global _singleton_connection

    if db_path == ':memory:':
        # Demo mode — always return the same connection
        if _singleton_connection is None:
            _singleton_connection = sqlite3.connect(
                ':memory:',
                check_same_thread=False  # required for FastAPI async
            )
            init_schema(_singleton_connection)
            create_example_dashboard(_singleton_connection)
        return _singleton_connection

    # Persistent mode — open the file
    conn = sqlite3.connect(db_path, check_same_thread=False)
    run_migrations(conn)
    return conn
```

### What demo mode guarantees

```
✅ All features work — same code as persistent mode
✅ Example dashboard visible immediately
✅ Create dashboards, panels, execute SQL
✅ Filters, insights, all features available
✅ No files created on disk
✅ Clean state on every restart
```

### What demo mode does not guarantee

```
❌ Data persists after closing SQLviz
❌ Data survives a crash
❌ Multiple sessions share data
```

Demo mode is designed for exploration, not production.
When the user is ready to save their work, they use persistent mode.

---

## Appendix — Port Configuration

SQLviz runs on port 4000 by default.

```bash
# Default
sqlviz my_project
# → http://localhost:4000

# Custom port (future feature)
sqlviz my_project --port 8080
# → http://localhost:8080
```

Port 4000 was chosen to avoid conflicts with:
```
3000 → common React/Node development servers
5173 → Vite development server
8000 → common FastAPI/Django development servers
8080 → common alternative HTTP port
```

---

*SQLviz Modes & CLI — v0.1.0 Draft*
*"One command. One file. Everything works."*
