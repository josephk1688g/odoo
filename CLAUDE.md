# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Odoo v19.0 (Final) — open-source ERP/CRM platform. Python 3.10–3.13, PostgreSQL 13+, LGPL-3 license. 611+ addon modules covering accounting, inventory, sales, HR, manufacturing, eCommerce, and more.

## Key Commands

### Running the Server

```bash
./odoo-bin -d <dbname>                          # Start with specific database
./odoo-bin -d <dbname> --dev=all                # Dev mode (auto-reload, debug)
./odoo-bin -d <dbname> --http-port=8069         # Custom port
./odoo-bin -c /path/to/odoo.conf                # With config file
```

### Database Operations

```bash
./odoo-bin db init <dbname>                     # Initialize new database
./odoo-bin db list                              # List databases
```

### Module Management

```bash
./odoo-bin module install <module_name> -d <db> # Install module
./odoo-bin module upgrade <module_name> -d <db> # Upgrade module
./odoo-bin scaffold <module_name> addons/       # Generate module skeleton
```

### Running Tests

```bash
# All tests for a module
./odoo-bin -d <test_db> -i <module> --test-enable --stop-after-init

# Specific test by tag
./odoo-bin -d <test_db> -i <module> --test-tags=<module>.test_file_name --test-enable --stop-after-init

# Tag filters
./odoo-bin -d <test_db> --test-tags=standard,not\ external --test-enable --stop-after-init
```

### Linting

```bash
ruff check .                                    # Lint (ruff 0.15.0+)
ruff check --fix .                              # Auto-fix
```

## Architecture

### Directory Layout

- **`odoo-bin`** — Entry point, calls `odoo.cli.main()`
- **`odoo/`** — Core framework
  - `orm/` — ORM: models, fields, decorators, environments, registry, domains, commands
  - `cli/` — CLI commands: server, shell, db, module, scaffold, deploy, populate
  - `http.py` — WSGI app, request handling, `@route` decorators
  - `sql_db.py` — PostgreSQL connection pooling
  - `exceptions.py` — `ValidationError`, `AccessError`, `UserError`, `MissingError`
  - `tests/` — Test framework (`TransactionCase`, `HttpCase`, `Form` helper)
  - `tools/` — Utility functions
  - `modules/` — Module loading and registry
- **`addons/`** — 611+ business modules (each self-contained)

### Addon Module Structure

Each addon follows this convention:
```
addons/<module_name>/
├── __manifest__.py      # Metadata: name, depends, data files, assets
├── __init__.py          # Model imports
├── models/              # ORM model definitions
├── views/               # XML view definitions (form, tree, search, kanban)
├── security/            # Access rules (ir.model.access.csv, ir.rule XML)
├── data/                # Base/demo data (XML/CSV)
├── controllers/         # HTTP controllers (@route endpoints)
├── wizard/              # Transient models for wizard dialogs
├── report/              # Report templates and definitions
├── static/              # JS (Owl components), SCSS, images
└── tests/               # Test files (test_*.py, common.py for fixtures)
```

### ORM Model Patterns

**Model types:** `models.Model` (persistent), `models.TransientModel` (wizard/temporary), `models.AbstractModel` (mixin)

**Inheritance types:**
- Classical: `_inherit = 'existing.model'` — extends existing model in-place
- Delegation: `_inherits = {'parent.model': 'parent_id'}` — FK-based inheritance
- Mixin: `_inherit = ['mixin.a', 'mixin.b']` — multiple inheritance

**Key decorators:** `@api.depends()`, `@api.constrains()`, `@api.onchange()`, `@api.model`, `@api.model_create_multi`

**Relational field commands** (for `write()`/`create()` on One2many/Many2many):
```python
from odoo.fields import Command
Command.create(vals)        # (0, 0, vals)
Command.update(id, vals)    # (1, id, vals)
Command.delete(id)          # (2, id)
Command.unlink(id)          # (3, id)
Command.link(id)            # (4, id)
Command.clear()             # (5,)
Command.set(ids)            # (6, 0, ids)
```

### Test Framework

Test classes in `odoo.tests.common`:
- **`TransactionCase`** — each test in its own rolled-back transaction
- **`HttpCase`** — HTTP request simulation, browser/JS testing
- **`Form`** — simulates UI form interactions for testing onchange logic

Tests use `@tagged()` decorator for filtering. Common tags: `'standard'`, `'at_install'`, `'post_install'`, `'-at_install'`.

Use `@mute_logger('odoo.addons.module')` to suppress expected log noise in tests.

### Frontend

- **Owl** — component framework for client-side JS
- **QWeb** — template engine (both server-side Jinja2 and client-side JS)
- Assets declared in `__manifest__.py` under `'assets'` key with bundle names like `'web.assets_backend'`, `'web.assets_frontend'`

### HTTP Routing

```python
from odoo import http

class MyController(http.Controller):
    @http.route('/path', auth='user', type='http')    # Returns HTML
    @http.route('/api', auth='public', type='json')    # Returns JSON-RPC
```

Auth modes: `'public'`, `'user'`, `'none'`

## Ruff Configuration

Target: Python 3.10. Key rules enabled: `E`, `W`, `F`, `I` (isort), `BLE`, `C`, `COM`, `EM`, `G`, `LOG`, `PLE`, `PLW`, `RUF`, `SIM`, `TRY`, `UP`. Line length (`E501`) is **not enforced**.

Import order (isort): future → stdlib → third-party → first-party (`odoo`) → local-folder (`odoo.addons`)

`F401` (unused imports) is ignored in `__init__.py` files (re-exports are standard Odoo pattern).

## Odoo-Specific Conventions

- **`self.env`** is the ORM environment; access models via `self.env['model.name']`
- **`self.env.user`** / **`self.env.company`** for current user/company context
- **`self.env.context`** is an immutable dict; use `self.with_context(key=val)` to modify
- **Domain expressions** filter records: `[('field', 'operator', value)]`
- **XML IDs** (`module.xml_id`) reference records across modules: `self.env.ref('module.xml_id')`
- Database schema is auto-managed by the ORM — never write raw DDL
- Security is defined per-model via `ir.model.access.csv` (CRUD) and `ir.rule` XML (row-level)
