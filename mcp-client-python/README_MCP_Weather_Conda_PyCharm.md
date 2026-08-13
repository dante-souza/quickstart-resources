# MCP Weather Quickstart — Conda + uv + PyCharm + MCP Inspector

This README documents a working local setup for the official Model Context Protocol (MCP) Python quickstarts:

- **Weather MCP server**: `weather-server-python/weather.py`
- **Local MCP client**: `mcp-client-python/client.py`
- **Environment**: Conda
- **Dependency installation**: `uv pip`
- **IDE execution**: PyCharm Run Configuration
- **MCP transport**: stdio
- **Web inspection/debugging**: MCP Inspector

The goal is to keep both the MCP client and MCP server running with the **same Conda Python interpreter**, while using `uv` only as a fast dependency installer.

---

## 1. Project layout

Example layout used in this setup:

```text
M:\ATOL\GitProjects\quickstart-resources\
│
├── mcp-client-python\
│   ├── client.py
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── .env
│   └── Makefile
│
└── weather-server-python\
    ├── weather.py
    ├── pyproject.toml
    └── uv.lock
```

Conda environment:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\
└── python.exe
```

---

# 2. Architecture

## 2.1 Runtime architecture

```text
                     ┌─────────────────────┐
                     │       PyCharm       │
                     │     Run button      │
                     └──────────┬──────────┘
                                │
                                │ selected interpreter
                                ▼
       C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
                                │
                                ▼
                       ┌────────────────┐
                       │   client.py    │
                       │ MCP client/LLM │
                       └───────┬────────┘
                               │
                               │ launches subprocess with
                               │ command=sys.executable
                               ▼
                       ┌────────────────┐
                       │   weather.py   │
                       │   MCP server   │
                       └───────┬────────┘
                               │
                               │ HTTP
                               ▼
                     ┌─────────────────────┐
                     │ api.weather.gov     │
                     │ National Weather    │
                     │ Service API         │
                     └─────────────────────┘
```

The MCP client and server communicate using **stdio**:

```text
client.py
   │
   │ stdin/stdout
   │ MCP protocol messages
   ▼
weather.py
```

The client does **not** need a separately started weather process.

It launches `weather.py` itself.

---

## 2.2 Tool execution flow

For a user query such as:

```text
What weather alerts are active in California?
```

the flow is:

```text
Natural-language query
        │
        ▼
     client.py
        │
        ▼
       LLM
        │
        │ sees available MCP tools
        ▼
   get_alerts
        │
        │ arguments
        ▼
{"state": "CA"}
        │
        ▼
MCP CallToolRequest
        │
        ▼
    weather.py
        │
        ▼
api.weather.gov
        │
        ▼
MCP tool result
        │
        ▼
       LLM
        │
        ▼
Natural-language answer
```

A successful run confirms all of the following:

```text
PyCharm
  ✓
Conda Python
  ✓
client.py
  ✓
MCP stdio
  ✓
weather.py
  ✓
MCP tool discovery
  ✓
MCP tool execution
  ✓
api.weather.gov
  ✓
LLM response
```

---

# 3. Conda environment

Create the environment if needed:

```powershell
conda create -n quickstart-resources python=3.12 -y
```

Activate it in a terminal:

```powershell
conda activate quickstart-resources
```

Verify:

```powershell
python --version
where.exe python
```

Expected interpreter:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
```

You can also verify directly:

```powershell
python -c "import sys; print(sys.executable)"
```

Expected:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
```

---

# 4. Using uv with Conda

In this setup:

- **Conda** manages the Python environment.
- **uv** installs Python packages into that Conda environment.
- `uv run` is intentionally not used for normal execution.
- PyCharm launches the programs directly with the Conda interpreter.

Architecture:

```text
pyproject.toml
      │
      ▼
   uv pip
      │
      ▼
Conda environment
      │
      ├── MCP SDK
      ├── Anthropic SDK
      ├── python-dotenv
      ├── Pydantic
      ├── HTTP client dependencies
      └── other project packages
```

Install the client dependencies:

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\mcp-client-python

uv pip install -r pyproject.toml
```

Install the weather server dependencies:

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\weather-server-python

uv pip install -r pyproject.toml
```

Because the Conda environment is active, `uv pip` installs into that environment.

---

## 4.1 Verify dependencies

Client:

```powershell
python -c "import anthropic; import mcp; import dotenv; print('Client OK')"
```

Expected:

```text
Client OK
```

Server:

```powershell
python -c "import mcp; import pydantic; print('Server OK')"
```

Expected:

```text
Server OK
```

You can also inspect installed packages:

```powershell
uv pip list
```

---

# 5. Important client.py adjustment for pure Conda execution

The upstream quickstart client may launch Python MCP servers using `uv`, for example:

```python
server_params = StdioServerParameters(
    command="uv",
    args=["--directory", str(path.parent), "run", path.name],
    env=None,
)
```

This caused the Windows error:

```text
FileNotFoundError: [WinError 2] The system cannot find the file specified
```

The reason was that PyCharm successfully launched `client.py` with Conda Python, but the child process then attempted to find an executable named `uv`.

For a **pure Conda runtime**, make the client launch the server using the same Python interpreter that is already running the client.

Add at the top of `client.py`:

```python
import sys
```

Then change the Python server launch block to:

```python
if is_python:
    path = Path(server_script_path).resolve()

    server_params = StdioServerParameters(
        command=sys.executable,
        args=[str(path)],
        env=None,
    )
```

The important line is:

```python
command=sys.executable
```

When PyCharm uses:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
```

then:

```python
sys.executable
```

returns that same path.

The server is therefore launched as:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
M:\ATOL\GitProjects\quickstart-resources\weather-server-python\weather.py
```

instead of:

```text
uv --directory ... run weather.py
```

---

## 5.1 Recommended connect_to_server pattern

The relevant section can look like:

```python
import sys
from pathlib import Path

from mcp import StdioServerParameters
from mcp.client.stdio import stdio_client


async def connect_to_server(self, server_script_path: str):
    is_python = server_script_path.endswith(".py")
    is_js = server_script_path.endswith(".js")

    if not (is_python or is_js):
        raise ValueError("Server script must be a .py or .js file")

    if is_python:
        path = Path(server_script_path).resolve()

        server_params = StdioServerParameters(
            command=sys.executable,
            args=[str(path)],
            env=None,
        )

    else:
        server_params = StdioServerParameters(
            command="node",
            args=[server_script_path],
            env=None,
        )

    # Continue with the original MCP stdio connection logic...
```

Keep the remainder of the official client logic unchanged.

---

# 6. PyCharm configuration

Open:

```text
Run
→ Edit Configurations
→ Add/Edit Python configuration
```

Use the following values.

## Name

```text
client
```

## Python interpreter

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
```

## Script path

```text
M:/ATOL/GitProjects/quickstart-resources/mcp-client-python/client.py
```

## Script parameters

This is required.

```text
M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py
```

Without it, the client prints:

```text
Usage: python client.py <path_to_server_script>
```

## Working directory

```text
M:/ATOL/GitProjects/quickstart-resources/mcp-client-python
```

## Environment variables

At minimum:

```text
PYTHONUNBUFFERED=1
```

The Anthropic API key can be placed either in the Run Configuration or in `.env`.

---

# 7. .env configuration

Create:

```text
M:\ATOL\GitProjects\quickstart-resources\mcp-client-python\.env
```

with:

```env
ANTHROPIC_API_KEY=sk-ant-your-key-here
```

Do not commit this file.

Add to `.gitignore`:

```gitignore
.env
```

Verify that it loads:

```powershell
python -c "from dotenv import load_dotenv; import os; load_dotenv(); print('API key loaded:', bool(os.getenv('ANTHROPIC_API_KEY')))"
```

Expected:

```text
API key loaded: True
```

---

# 8. What PyCharm actually runs

With the configuration above, PyCharm effectively executes:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
M:\ATOL\GitProjects\quickstart-resources\mcp-client-python\client.py
M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py
```

Then `client.py` uses:

```python
sys.executable
```

to launch:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
M:\ATOL\GitProjects\quickstart-resources\weather-server-python\weather.py
```

This ensures the same Conda environment is used by both processes.

---

# 9. Running the MCP client

Click:

```text
▶ Run
```

in PyCharm.

A successful startup looks similar to:

```text
Connected to server with tools: ['get_alerts', 'get_forecast']

MCP Client Started!
Type your queries or 'quit' to exit.

Query:
```

This confirms that MCP tool discovery succeeded.

---

# 10. Querying the weather server

At:

```text
Query:
```

type natural-language questions.

Example:

```text
What weather alerts are active in California?
```

A successful request looks like:

```text
Processing request of type ListToolsRequest
Processing request of type CallToolRequest

HTTP Request:
GET https://api.weather.gov/alerts/active/area/CA
HTTP/1.1 200 OK

[Calling tool get_alerts with args {'state': 'CA'}]
```

This demonstrates that:

```text
User query
   ↓
LLM
   ↓
MCP tool selection
   ↓
get_alerts(state="CA")
   ↓
weather.py
   ↓
api.weather.gov
```

---

## 10.1 Alert queries

Examples:

```text
What weather alerts are active in California?
```

```text
Are there any dangerous weather alerts in Texas?
```

```text
What alerts are active in Nevada?
```

```text
Summarize only severe weather alerts in California.
```

The corresponding MCP tool is:

```text
get_alerts
```

with arguments similar to:

```json
{
  "state": "CA"
}
```

---

## 10.2 Forecast queries

Examples:

```text
What's the weather forecast for Seattle, Washington?
```

```text
What's the forecast for Sacramento, California?
```

```text
What's the forecast for San Francisco?
```

The client can choose:

```text
get_forecast
```

with latitude/longitude arguments.

Conceptually:

```json
{
  "latitude": 47.6062,
  "longitude": -122.3321
}
```

The server then queries the National Weather Service.

---

# 11. Stopping the client

At the prompt:

```text
Query:
```

type:

```text
quit
```

The client exits and closes its MCP subprocess/stdio connection.

---

# 12. Running weather.py manually

You can test that the server itself starts:

```powershell
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe `
  M:\ATOL\GitProjects\quickstart-resources\weather-server-python\weather.py
```

It may look like it is doing nothing.

That is expected because the server is waiting for MCP messages over stdio.

```text
weather.py
   │
   │ waiting
   ▼
stdin/stdout MCP transport
```

Stop it with:

```text
Ctrl+C
```

Do not keep a manually started copy running when the local `client.py` is also configured to spawn its own copy.

---

# 13. Important stdio rule: do not log to stdout

For an MCP server using stdio, stdout carries protocol messages.

Avoid:

```python
print("Debug message")
```

inside the server.

Instead use stderr:

```python
import sys

print("Debug message", file=sys.stderr)
```

You can verify the Python interpreter used by the server with:

```python
import sys

print(
    "SERVER PYTHON:",
    sys.executable,
    file=sys.stderr,
)
```

For the client, ordinary stdout logging is fine:

```python
import sys

print("CLIENT PYTHON:", sys.executable)
```

Both should report:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
```

---

# 14. MCP Inspector

The MCP Inspector provides a browser-based development/testing interface.

Architecture:

```text
Browser
   │
   ▼
MCP Inspector
   │
   │ stdio
   ▼
Conda Python
   │
   ▼
weather.py
   │
   ├── get_alerts
   └── get_forecast
```

The Inspector talks directly to the MCP server.

It does not need:

```text
client.py
```

and it does not need the Anthropic API to manually invoke MCP tools.

---

## 14.1 Start Inspector manually

Using the exact Conda Python:

```powershell
npx @modelcontextprotocol/inspector `
  "C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe" `
  "M:\ATOL\GitProjects\quickstart-resources\weather-server-python\weather.py"
```

The Inspector opens a local browser interface.

You should be able to discover:

```text
get_alerts
get_forecast
```

and invoke them manually.

Example tool input:

```json
{
  "state": "CA"
}
```

For `get_forecast`:

```json
{
  "latitude": 47.6062,
  "longitude": -122.3321
}
```

---

# 15. Makefile for MCP Inspector

Create:

```text
M:\ATOL\GitProjects\quickstart-resources\mcp-client-python\Makefile
```

Recommended version:

```makefile
PYTHON := C:/Users/dante/miniconda3/envs/quickstart-resources/python.exe
WEATHER_SERVER := M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py

.PHONY: inspector

inspector:
	@echo Starting MCP Inspector...
	@echo Python: $(PYTHON)
	@echo Server: $(WEATHER_SERVER)
	npx @modelcontextprotocol/inspector "$(PYTHON)" "$(WEATHER_SERVER)"
```

**Important:** the recipe lines under `inspector:` must begin with a TAB.

Run:

```powershell
make inspector
```

Equivalent command:

```powershell
npx @modelcontextprotocol/inspector `
  "C:/Users/dante/miniconda3/envs/quickstart-resources/python.exe" `
  "M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py"
```

---

## 15.1 Optional Makefile with more commands

A more complete Makefile can be:

```makefile
PYTHON := C:/Users/dante/miniconda3/envs/quickstart-resources/python.exe

ROOT := M:/ATOL/GitProjects/quickstart-resources
CLIENT := $(ROOT)/mcp-client-python/client.py
WEATHER_SERVER := $(ROOT)/weather-server-python/weather.py

.PHONY: help client server inspector check

help:
	@echo Available commands:
	@echo "  make client     - Run MCP client + weather server"
	@echo "  make server     - Run weather server directly"
	@echo "  make inspector  - Start MCP Inspector"
	@echo "  make check      - Verify Python dependencies"

client:
	"$(PYTHON)" "$(CLIENT)" "$(WEATHER_SERVER)"

server:
	"$(PYTHON)" "$(WEATHER_SERVER)"

inspector:
	npx @modelcontextprotocol/inspector "$(PYTHON)" "$(WEATHER_SERVER)"

check:
	"$(PYTHON)" -c "import anthropic, mcp, dotenv, pydantic; print('Dependencies OK')"
```

Then:

```powershell
make help
```

```powershell
make check
```

```powershell
make client
```

```powershell
make server
```

```powershell
make inspector
```

---

# 16. Troubleshooting

## Problem: `ModuleNotFoundError: No module named 'anthropic'`

Meaning:

```text
PyCharm's selected Conda interpreter
```

does not have the Anthropic SDK installed.

Check the exact interpreter:

```powershell
python -c "import sys; print(sys.executable)"
```

Then:

```powershell
uv pip install anthropic
```

Or install all client dependencies:

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\mcp-client-python

uv pip install -r pyproject.toml
```

Verify:

```powershell
python -c "import anthropic; print(anthropic.__version__)"
```

---

## Problem: `Usage: python client.py <path_to_server_script>`

Meaning:

```text
Script parameters
```

is empty in the PyCharm Run Configuration.

Set:

```text
M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py
```

under:

```text
Run
→ Edit Configurations
→ Script parameters
```

---

## Problem: `<No interpreter>` in PyCharm

Select:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
```

under the Run Configuration's Python interpreter.

---

## Problem: `FileNotFoundError: [WinError 2]`

If the traceback occurs while MCP tries to create the child process, inspect `client.py`.

If it uses:

```python
command="uv"
```

change it to:

```python
command=sys.executable
```

and:

```python
args=[str(path)]
```

This keeps the client and server in the same Conda runtime.

---

## Problem: server starts and appears frozen

Expected for stdio transport.

The server is waiting for MCP protocol messages.

```text
weather.py
   ↓
waits on stdin
```

Use the local MCP client or MCP Inspector to interact with it.

---

## Problem: package installed but PyCharm still cannot import it

Compare:

```python
import sys
print(sys.executable)
```

inside PyCharm with:

```powershell
python -c "import sys; print(sys.executable)"
```

in the terminal.

They must refer to the same environment.

If necessary, force `uv` to target the exact interpreter:

```powershell
uv pip install `
  --python "C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe" `
  anthropic
```

---

# 17. Successful reference output

A working setup produced:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
M:\ATOL\GitProjects\quickstart-resources\mcp-client-python\client.py
M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py
```

followed by:

```text
Connected to server with tools: ['get_alerts', 'get_forecast']

MCP Client Started!
Type your queries or 'quit' to exit.

Query:
```

A successful tool invocation:

```text
Query: What weather alerts are active in California?

Processing request of type ListToolsRequest

Processing request of type CallToolRequest

HTTP Request:
GET https://api.weather.gov/alerts/active/area/CA
HTTP/1.1 200 OK

[Calling tool get_alerts with args {'state': 'CA'}]
```

This confirms the system works end-to-end.

---

# 18. Final setup summary

## Runtime

```text
PyCharm
   ↓
Conda Python
   ↓
client.py
   ↓
MCP stdio
   ↓
weather.py
   ↓
National Weather Service API
```

## Dependency management

```text
pyproject.toml
   ↓
uv pip
   ↓
Conda environment
```

## Development/debugging

```text
make inspector
   ↓
MCP Inspector
   ↓
Browser UI
   ↓
weather.py
```

## Important design choice

Use:

```python
command=sys.executable
```

for Python MCP subprocesses when the goal is:

> **PyCharm-selected Conda environment controls both client and server execution.**

Use `uv` for dependency installation:

```powershell
uv pip install ...
```

rather than introducing a separate uv-managed runtime environment.

---

# 19. Quick reference

### Activate environment

```powershell
conda activate quickstart-resources
```

### Verify Python

```powershell
python -c "import sys; print(sys.executable)"
```

### Install client dependencies

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\mcp-client-python
uv pip install -r pyproject.toml
```

### Install weather dependencies

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\weather-server-python
uv pip install -r pyproject.toml
```

### Run from PyCharm

```text
Script:
M:/ATOL/GitProjects/quickstart-resources/mcp-client-python/client.py

Script parameters:
M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py

Interpreter:
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
```

### Run from command line

```powershell
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe `
  M:\ATOL\GitProjects\quickstart-resources\mcp-client-python\client.py `
  M:\ATOL\GitProjects\quickstart-resources\weather-server-python\weather.py
```

### Inspector

```powershell
make inspector
```

### Example query

```text
What weather alerts are active in California?
```

### Exit

```text
quit
```

---

# 20. Security notes

Never commit:

```text
.env
```

Never hardcode:

```text
ANTHROPIC_API_KEY
```

in Python source.

Recommended `.gitignore` entry:

```gitignore
.env
```

For production or shared systems, use a proper secrets manager instead of local `.env` files.

---

# 21. Next possible extensions

This setup is a good base for experimenting with:

- multiple MCP servers
- a Streamlit or Gradio MCP client
- an HTTP/Streamable HTTP MCP transport
- remote MCP servers
- Claude Desktop / Claude Code MCP integration
- OpenAI-compatible clients
- CrewAI agents consuming MCP tools
- custom MCP tools
- authentication
- Docker packaging
- Kubernetes deployment
- observability and structured logging

The current design deliberately keeps the first working architecture simple:

```text
one Conda environment
+
one client
+
one local MCP server
+
stdio transport
+
optional Inspector UI
```
