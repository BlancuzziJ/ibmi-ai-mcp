# Connecting AI Assistants to IBM i (AS/400) with MCP

**Published:** May 2026  
**Topic:** Model Context Protocol (MCP) · IBM i · DB2 for i · VS Code Copilot / Claude

---

## Overview

This guide walks through how to build an **MCP (Model Context Protocol) server** that gives AI coding assistants — VS Code Copilot agent mode, Claude Code, or any MCP-compatible client — live, read-capable access to an IBM i (AS/400) system.

With this setup, you can ask your AI assistant things like:

- *"List all the physical files in library MYLIB that start with MST"*
- *"Show me the field definitions for ORDHDRPF"*
- *"Read the source member CALCSHIP in QRPGLESRC"*
- *"Search all RPG source for occurrences of 'CHAIN CUSTOMER'"*
- *"Run this SQL query against the production DB and show me the results"*

...and the AI will execute those operations live against your IBM i system and return real results into the conversation.

---

## How It Works — Architecture

The chain has four layers:

```
AI Assistant (MCP client)
    ↓  JSON-RPC over stdio
mcp-server.js  (Node.js entry point)
    ↓  require()
db2-mcp-connector.js  (MCP protocol + tool handlers)
    ↓  child_process.spawn()
django_pyodbc_adapter.js
    ↓  python manage.py dbquery --db <connection> --query <sql>
dbquery.py  (Django management command)
    ↓  Django connection pool
IBM i Access ODBC Driver 64-bit → IBM i DB2
```

**Why this layered design?**

- The Node.js MCP layer handles the JSON-RPC stdio protocol that AI clients expect
- The Python/Django layer handles the actual IBM i ODBC connection — using drivers and packages that are well-established for AS/400 connectivity
- Django's connection pooling manages reconnects and connection lifecycle automatically
- You don't need any native Node.js IBM DB2 drivers — all DB connectivity runs through Python

---

## Prerequisites

### On your server (Linux)

- Python 3.9+ with a virtual environment
- Django 4.x or 5.x
- Node.js 18+
- IBM i Access Client Solutions — specifically the **IBM i Access ODBC Driver 64-bit**
- `pyodbc` Python package (`pip install pyodbc`)
- `csv-stringify` npm package (for CSV export tool)

### On your IBM i

- A user profile with `*USE` authority to the libraries and objects you want to expose
- Network access from your Linux server to the IBM i (port 446 or 8470 depending on your SSL setup)

### Verify your ODBC driver is installed

```bash
odbcinst -q -d
```

You should see something like:
```
[IBM i Access ODBC Driver 64-bit]
[IBM DB2 ODBC DRIVER]
```

---

## Step 1 — Credential Storage (Do This First)

Before writing a single line of configuration, set up secure credential storage. **Never put hostnames, usernames, or passwords directly in `settings.py` or any file that gets committed to source control.**

### Install python-decouple

```bash
source venv/bin/activate
pip install python-decouple
```

### Create a `.env` file in your project root

```ini
# .env  — local only, never commit this file
IBMI_HOST=192.168.1.xxx
IBMI_NAME=YOUR_SYSTEM_NAME
IBMI_USER=your_ibmi_user
IBMI_PASSWORD=your_ibmi_password
IBMI_PORT=446
```

### Add `.env` to `.gitignore` immediately

```bash
echo ".env" >> .gitignore
echo "*.env" >> .gitignore
```

Verify it's excluded before your first commit:

```bash
git status .env   # should show "nothing to commit" or "ignored"
```

For production/server deployments, set these as **OS-level environment variables** (systemd `EnvironmentFile=`, Docker secrets, or your platform's secrets manager) rather than a `.env` file on disk.

---

## Step 1b — Django Database Configuration

With credentials stored safely, reference them from `settings.py` using `python-decouple`:

```python
from decouple import config

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    },
    'ibmi_db': {
        'ENGINE': 'ibm_db_django',
        'NAME': config('IBMI_NAME'),
        'USER': config('IBMI_USER'),
        'PASSWORD': config('IBMI_PASSWORD'),
        'HOST': config('IBMI_HOST'),
        'PORT': config('IBMI_PORT', default='446'),
        'OPTIONS': {
            'driver': 'IBM i Access ODBC Driver 64-bit',
            'autocommit': True,
        },
    },
}
```

`config()` reads from the `.env` file in development and falls back to real environment variables in production — no code change required between environments.

> You can also use a database router if you have multiple connections. Django's multi-database support works well here.

---

## Step 2 — The Django Management Command (The Bridge)

This is the critical piece that connects the Node.js MCP server to the IBM i through Django's connection pool. Create a Django app (e.g., `dbtools`) and add this management command:

**`dbtools/management/commands/dbquery.py`**

```python
import json
from django.core.management.base import BaseCommand, CommandError
from django.db import connections


class Command(BaseCommand):
    help = 'Execute a SQL query against a Django database connection and return JSON results'

    def add_arguments(self, parser):
        parser.add_argument('--db', type=str, required=True,
                            help='Django DATABASES key to use')
        parser.add_argument('--query', type=str, required=True,
                            help='SQL query to execute')
        parser.add_argument('--params', type=str, default='none',
                            help='Comma-separated query parameters (use "null" for NULL)')

    def handle(self, *args, **options):
        db_name = options['db']
        query = options['query']
        params_str = options['params']

        # Parse parameters
        params = []
        if params_str and params_str != 'none':
            params = params_str.split(',')
            params = [None if p == 'null' else p for p in params]

        # Validate the connection name before attempting to use it
        if db_name not in connections.databases:
            available = list(connections.databases.keys())
            raise CommandError(
                f"Database '{db_name}' not configured. Available: {available}"
            )

        with connections[db_name].cursor() as cursor:
            cursor.execute(query, params)
            columns = [col[0] for col in cursor.description] if cursor.description else []
            rows = cursor.fetchall()
            results = [dict(zip(columns, row)) for row in rows]
            # default=str handles dates, Decimal, and other non-JSON-serializable types
            self.stdout.write(json.dumps(results, default=str))
```

> **Note on parameter parsing:** The comma-split approach works well for simple scalar parameters. If your use case requires string values that may contain commas (e.g. names), consider switching to a JSON-encoded params argument instead.

Test it from the command line:

```bash
source venv/bin/activate
python manage.py dbquery --db ibmi_db --query "SELECT CURRENT_DATE AS TODAY FROM SYSIBM.SYSDUMMY1"
```

You should see: `[{"TODAY": "2026-05-01"}]`

---

## Step 3 — The Node.js Adapter

This adapter is what the MCP connector calls. It spawns the Django management command and parses the JSON output.

**`mcp/connectors/django_pyodbc_adapter.js`**

```javascript
const { spawn } = require('child_process');
const path = require('path');

const PROJECT_ROOT = process.env.DJANGO_PROJECT_ROOT || '/path/to/your/project';
const PYTHON_PATH = path.join(PROJECT_ROOT, 'venv', 'bin', 'python');
const MANAGE_PY = path.join(PROJECT_ROOT, 'manage.py');

async function getConnection(connectionName) {
    return {
        query: async (sql, params = []) => {
            return new Promise((resolve, reject) => {
                const paramStr = params.length > 0
                    ? params.map(p => p === null ? 'null' : String(p)).join(',')
                    : 'none';

                const args = [
                    MANAGE_PY, 'dbquery',
                    '--db', connectionName,
                    '--query', sql,
                    '--params', paramStr,
                ];

                const proc = spawn(PYTHON_PATH, args, {
                    cwd: PROJECT_ROOT,
                    env: { ...process.env, DJANGO_SETTINGS_MODULE: 'your_project.settings' },
                });

                let stdout = '';
                let stderr = '';
                proc.stdout.on('data', d => stdout += d.toString());
                proc.stderr.on('data', d => stderr += d.toString());

                proc.on('close', code => {
                    if (code !== 0) {
                        return reject(new Error(`dbquery failed (exit ${code}): ${stderr}`));
                    }
                    // Find the last line that looks like JSON
                    const lines = stdout.trim().split('\n');
                    const jsonLine = [...lines].reverse().find(l =>
                        l.trim().startsWith('[') || l.trim().startsWith('{')
                    );
                    if (!jsonLine) {
                        return reject(new Error(`No JSON found in output: ${stdout}`));
                    }
                    try {
                        resolve(JSON.parse(jsonLine));
                    } catch (e) {
                        reject(new Error(`JSON parse error: ${e.message}\nOutput: ${jsonLine}`));
                    }
                });
            });
        },
        close: async () => Promise.resolve(), // Django manages connection lifecycle
    };
}

module.exports = { getConnection };
```

---

## Step 4 — The MCP Server Entry Point

**`mcp-server.js`** (project root)

```javascript
#!/usr/bin/env node
process.chdir(__dirname);

process.env.DJANGO_PROJECT_ROOT = __dirname;
process.env.DJANGO_DB_CONNECTION = 'ibmi_db';  // matches your DATABASES key
process.env.DB2_DEFAULT_SCHEMA   = 'YOUR_LIB'; // default library/schema
process.env.LOG_LEVEL            = 'info';
process.env.LOG_FILE             = __dirname + '/logs/mcp.log';

require('./mcp/connectors/db2-mcp-connector.js');
```

---

## Step 5 — The MCP Connector (Tool Definitions)

The connector implements the MCP stdio JSON-RPC protocol and defines the tools that the AI assistant can call. Here is a condensed skeleton:

**`mcp/connectors/db2-mcp-connector.js`**

```javascript
const readline = require('readline');
const adapter = require('./django_pyodbc_adapter');

const MCP_PROTOCOL_VERSION = '2024-11-05';
const DB_CONNECTION = process.env.DJANGO_DB_CONNECTION || 'ibmi_db';
const DEFAULT_SCHEMA = process.env.DB2_DEFAULT_SCHEMA || 'YOUR_LIB';

// Tool definitions (sent to the AI client on initialize)
const TOOLS = [
    {
        name: 'runQuery',
        description: 'Execute a SQL query against IBM i DB2',
        inputSchema: {
            type: 'object',
            properties: {
                query: { type: 'string', description: 'SQL query to execute' },
                params: { type: 'array', items: { type: 'string' }, description: 'Query parameters' },
            },
            required: ['query'],
        },
    },
    {
        name: 'listTables',
        description: 'List tables in a schema. Always provide a search filter on large libraries.',
        inputSchema: {
            type: 'object',
            properties: {
                schema:  { type: 'string' },
                search:  { type: 'string', description: 'Filter by name (partial match)' },
                limit:   { type: 'number', default: 50 },
            },
        },
    },
    // ... additional tools (see full list below)
];

// MCP stdio loop
const rl = readline.createInterface({ input: process.stdin });
rl.on('line', async (line) => {
    let msg;
    try { msg = JSON.parse(line); } catch { return; }

    if (msg.method === 'initialize') {
        respond(msg.id, {
            protocolVersion: MCP_PROTOCOL_VERSION,
            capabilities: { tools: {} },
            serverInfo: { name: 'IBM-System-i', version: '1.0.0' },
        });
    } else if (msg.method === 'tools/list') {
        respond(msg.id, { tools: TOOLS });
    } else if (msg.method === 'tools/call') {
        const result = await handleTool(msg.params.name, msg.params.arguments || {});
        respond(msg.id, { content: [{ type: 'text', text: JSON.stringify(result, null, 2) }] });
    }
});

function respond(id, result) {
    process.stdout.write(JSON.stringify({ jsonrpc: '2.0', id, result }) + '\n');
}
```

---

## Available Tools

Here are the 16 tools exposed to the AI assistant, all using IBM i catalog syntax:

| Tool | Description | Required Params |
|------|-------------|-----------------|
| `runQuery` | Execute any SQL | `query` |
| `listTables` | List tables in a schema | — (use `search` filter!) |
| `describeTable` | Column structure of a table | `table` |
| `runSqlFile` | Execute a `.sql` script | `filePath` |
| `exportResults` | Run query, save CSV | `query`, `outputPath` |
| `readSource` | Read a source member (RPG, CL, DDS…) | `lib`, `file`, `member` |
| `listMembers` | List members in a source file | `lib`, `file` |
| `listSourceFiles` | List source files in a library | `lib` |
| `describeFile` | Field definitions for a PF or LF | `lib`, `file` |
| `listFiles` | List PF/LF files in a library | `lib` |
| `getFileKeys` | Key fields for a PF or LF | `lib`, `file` |
| `getLogicals` | Logical files over a physical file | `lib`, `pf` |
| `listPrograms` | List programs in a library | `lib` |
| `listObjects` | List objects by type | `lib` |
| `getJobLog` | Retrieve recent job log entries | — |
| `searchSource` | Search source text across members | `lib`, `pattern` |

> ⚠️ **Critical:** Always pass a `search`, `namePattern`, or `type` filter when calling listing tools against large libraries. An unfiltered call against a large library can return millions of rows and overflow the AI's context window.

---

## IBM i Catalog Tables — Use QSYS2, Not SYSCAT

This is the most common mistake when adapting generic DB2 examples for IBM i.  
IBM i uses its own system catalog under `QSYS2` — **not** the `SYSCAT` schema used by DB2 LUW (Linux/Unix/Windows).

| Purpose | IBM i (AS/400) | DB2 LUW — ❌ Do NOT use |
|---------|---------------|------------------------|
| List tables | `QSYS2.SYSTABLES` | `SYSCAT.TABLES` |
| Column info | `QSYS2.SYSCOLUMNS` | `SYSCAT.COLUMNS` |
| Source members | `QSYS2.SYSPARTITIONSTAT` | N/A |
| Program objects | `QSYS2.PROGRAM_INFO` | N/A |
| Read IFS / source | `QSYS2.IFS_READ()` | N/A |
| Health check | `SYSIBM.SYSDUMMY1` | `SYSIBM.SYSDUMMY1` |

### Example — list tables in a library

```sql
SELECT TABLE_NAME, TABLE_TYPE, TABLE_TEXT
FROM QSYS2.SYSTABLES
WHERE TABLE_SCHEMA = 'MYLIB'
  AND TABLE_NAME LIKE 'MST%'
ORDER BY TABLE_NAME
FETCH FIRST 50 ROWS ONLY
```

### Example — read a source member

```sql
SELECT ORDINAL_POSITION, LINE
FROM TABLE(QSYS2.IFS_READ(
    PATH_NAME => '/QSYS.LIB/MYLIB.LIB/QRPGLESRC.FILE/MYPGM.MBR',
    END_OF_LINE => 'ANY'
)) AS SRC
ORDER BY ORDINAL_POSITION
```

---

## VS Code Configuration

Add this to your workspace's `.vscode/mcp.json`:

```json
{
    "mcpServers": {
        "IBM-System-i": {
            "command": "node",
            "args": ["mcp-server.js"],
            "cwd": "/path/to/your/project"
        }
    }
}
```

Restart VS Code after adding or changing this file. The server name **IBM-System-i** will appear in Copilot's agent mode tool list.

---

## Claude Desktop Configuration

There are two ways to connect Claude Desktop depending on where your MCP server runs.

---

### Option A — Local (Claude Desktop and MCP server on the same machine)

Add to `claude_desktop_config.json`:

```json
{
    "mcpServers": {
        "IBM-System-i": {
            "command": "node",
            "args": ["/path/to/your/project/mcp-server.js"]
        }
    }
}
```

---

### Option B — Remote over the internet via `mcp-remote`

If your MCP server runs on a **remote Linux server** (not on the machine running Claude Desktop), you can expose it over HTTPS using an SSE (Server-Sent Events) endpoint and connect to it with `npx mcp-remote`.

This is the approach to use when:
- Your IBM i connectivity lives on a server, not your local workstation
- You want multiple team members to connect their Claude Desktop to the same MCP server
- Your local machine doesn't have the IBM i ODBC drivers installed

#### Step 1 — Expose your MCP server over HTTPS

You need a reverse proxy (nginx, Caddy, etc.) sitting in front of your MCP server's SSE endpoint, with a valid TLS certificate. A common pattern using Caddy:

```
# Caddyfile
mcp.yourdomain.com {
    reverse_proxy localhost:3000
}
```

Your MCP server needs to listen on HTTP (the reverse proxy handles TLS termination).

#### Step 2 — Add a token to secure the endpoint

**Never expose an unauthenticated MCP endpoint on the public internet.** Add a secret token that must be passed as a query parameter. In your MCP server, validate it before accepting connections:

```javascript
// In your MCP server's HTTP/SSE handler
const VALID_TOKEN = process.env.MCP_ACCESS_TOKEN;

app.get('/sse', (req, res) => {
    if (req.query.token !== VALID_TOKEN) {
        return res.status(401).json({ error: 'Unauthorized' });
    }
    // ... proceed with SSE connection
});
```

Store the token in your `.env` file (never hardcode it):

```ini
# .env
MCP_ACCESS_TOKEN=generate-a-long-random-string-here
```

Generate a strong token:
```bash
openssl rand -hex 32
```

#### Step 3 — Configure Claude Desktop

Install `mcp-remote` (a one-time setup, no global install needed — `npx` handles it):

```json
{
    "mcpServers": {
        "IBM-System-i": {
            "command": "npx",
            "args": [
                "mcp-remote",
                "https://mcp.yourdomain.com/sse?token=YOUR_SECRET_TOKEN_HERE"
            ]
        }
    }
}
```

> **Security note:** The token in the URL is visible in your `claude_desktop_config.json`. Treat this file like a password file — ensure it has restrictive file permissions (`chmod 600`) and is never committed to source control.

#### Step 4 — Verify the connection

In Claude Desktop, open a new conversation and look for **IBM-System-i** in the tool list (hammer icon). If it appears, the connection is live. Test it by asking:

> *"Use the IBM-System-i tool to run: SELECT CURRENT_DATE AS TODAY FROM SYSIBM.SYSDUMMY1"*

---

### Configuration file locations

| OS | `claude_desktop_config.json` path |
|----|----------------------------------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

Restart Claude Desktop after any changes to this file.

---

## IBM i SQL Tips & Gotchas

These tripped us up during development:

### 1. The `AS` keyword in CTEs

When using Common Table Expressions (CTEs), **do not** use `AS` for table aliases inside the CTE body — only use it for column aliases:

```sql
-- ✅ Correct
WITH recent AS (
    SELECT * FROM MYLIB.ORDHDR t WHERE t.STATUS = 'O'
)
SELECT * FROM recent

-- ❌ Wrong — "Keyword AS not expected"
WITH recent AS (
    SELECT * FROM MYLIB.ORDHDR AS t WHERE t.STATUS = 'O'
)
```

### 2. Date arithmetic — use `DAYS()`

```sql
-- ✅ IBM i style
WHERE DAYS(CURRENT_DATE) - DAYS(ORDER_DATE) <= 30

-- ❌ Standard SQL interval syntax may not work
WHERE ORDER_DATE >= CURRENT_DATE - INTERVAL 30 DAYS
```

### 3. YYYYMMDD packed dates

IBM i applications often store dates as 8-digit packed decimal numbers. Convert them like this:

```sql
DATE(
    SUBSTR(DIGITS(DATE_FIELD), 1, 4) || '-' ||
    SUBSTR(DIGITS(DATE_FIELD), 5, 2) || '-' ||
    SUBSTR(DIGITS(DATE_FIELD), 7, 2)
)
```

### 4. NULL handling in Python

IBM DB2 for i returns `None` for nullable fields. When processing results in Python, always use:

```python
# ✅ Safe — handles None, empty string, and normal values
value = (row.get('FIELD_NAME') or '').strip()

# ❌ Will crash if the field returns None
value = row.get('FIELD_NAME', '').strip()
```

### 5. Nested COUNT with CTEs

IBM i does not support `SELECT COUNT(*) FROM (WITH ... SELECT ...)` patterns. Instead, fetch the full result set in Python and use `len()`:

```python
cursor.execute(main_sql)
rows = cursor.fetchall()
total_count = len(rows)
```

---

## Security Considerations

### Credential Storage — The Non-Negotiables

| Rule | Why it matters |
|------|----------------|
| **Never commit credentials to source control** | Git history is permanent. A leaked password in a commit from 3 years ago is still a live vulnerability. |
| **Use a `.env` file locally, OS env vars in production** | `.env` files are simple and effective for development; real environment variables are the standard for servers. |
| **Add `.env` to `.gitignore` before your first commit** | It's much harder to undo an accidental commit than to prevent one. |
| **Use `python-decouple` or `django-environ`** | These read from `.env` in dev and fall back to real env vars in production automatically. |
| **Do not log credentials** | Ensure your MCP log file (`mcp.log`) does not capture connection strings or query parameters containing sensitive values. |

### Secrets Management for Production

For production deployments, prefer a proper secrets manager over `.env` files on disk:

- **Linux/systemd:** Use `EnvironmentFile=` in your service unit pointing to a file with `600` permissions owned by your service user
- **Docker / Kubernetes:** Use Docker secrets or Kubernetes Secrets (base64-encoded, mounted as environment variables)
- **Cloud platforms:** AWS Secrets Manager, Azure Key Vault, GCP Secret Manager — all integrate with Python via their respective SDKs
- **HashiCorp Vault:** A solid self-hosted option if you manage your own infrastructure

Example systemd `EnvironmentFile` approach:

```ini
# /etc/myproject/ibmi.env  — owned by service user, chmod 600
IBMI_HOST=192.168.1.xxx
IBMI_USER=your_ibmi_user
IBMI_PASSWORD=your_ibmi_password
```

```ini
# /etc/systemd/system/myproject.service
[Service]
EnvironmentFile=/etc/myproject/ibmi.env
ExecStart=/path/to/venv/bin/gunicorn ...
```

### IBM i User Profile Authority

- Create a **dedicated service account** on IBM i for this integration — do not use a personal or admin profile
- Grant only `*USE` authority to the libraries and objects you need to expose
- Do not grant `*CHANGE`, `*ALL`, or `*ALLOBJ` special authority
- Enable object-level auditing (`CHGOBJAUD`) on sensitive files so you have a record of what was read
- Consider using `QSYS2.AUTHORITY_COLLECTION` to verify the service account's effective authority before going live

### Network & Runtime

- The MCP server communicates over **stdio only** — it has no open network port and cannot be accessed remotely
- Restrict IBM i port 446 (or 8470) at the firewall to only the IP of your Linux server
- Rotate the IBM i service account password on a schedule and update your secrets store — not your source code
- Consider adding a query allowlist or read-only enforcement at the Django layer if your team needs guardrails around which SQL the AI can execute

---

## Troubleshooting

### "No module named 'pyodbc'"

```bash
source venv/bin/activate
pip install pyodbc
```

### "Data source name not found"

Verify your ODBC driver is installed and the driver name in `settings.py` exactly matches the output of `odbcinst -q -d`.

### dbquery command not found

```bash
python manage.py dbquery --help
```

If this fails, check that `dbtools` (or your equivalent app) is in `INSTALLED_APPS` in `settings.py`.

### MCP server not appearing in Copilot

1. Confirm `.vscode/mcp.json` exists and is valid JSON
2. Confirm `node mcp-server.js` runs without errors from the project root
3. Restart VS Code — the MCP server list is loaded at startup

### Test the full chain from Node.js

```bash
node -e "
const adapter = require('./mcp/connectors/django_pyodbc_adapter');
adapter.getConnection('ibmi_db').then(conn =>
    conn.query('SELECT CURRENT_DATE AS TODAY FROM SYSIBM.SYSDUMMY1')
).then(r => console.log('Connected:', JSON.stringify(r)));
"
```

---

## What's Possible Once Connected

Once this is running, your AI assistant can assist with real IBM i work in context:

- **Code review:** *"Read CALCSHIP from QRPGLESRC and explain what it does"*
- **Schema exploration:** *"What fields are in the order header file? Which ones are key fields?"*
- **Data investigation:** *"Show me the 10 most recent orders where the status is still open"*
- **Migration prep:** *"Search all RPG source in MYLIB for any references to the old API QCMDEXC"*
- **Documentation:** *"Describe all the logical files built over CUSTMST and explain their key structures"*

The AI has no write access by default — the `dbquery` command runs `SELECT` queries and any DDL you explicitly provide, but there is no automatic mutation path.

---

## Contributing & Community

If you build on this pattern or adapt it for your environment, consider sharing back:

- Different ODBC driver configurations (e.g., SSL, port 8470)
- Additional tool definitions (e.g., `submitJob`, `getSpooledFile`)
- Adaptations for other MCP clients (Cursor, Zed, etc.)
- Tips for specific IBM i versions (V7R2, V7R3, V7R4, V7R5)

The IBM i community has decades of valuable knowledge — integrating modern AI tooling with these systems opens a lot of doors for teams maintaining legacy RPG/CL codebases and for anyone looking to modernize or document existing AS/400 applications.

---

*Built on Django 5.x · Node.js 18+ · IBM i Access ODBC Driver 64-bit · MCP Protocol 2024-11-05*
