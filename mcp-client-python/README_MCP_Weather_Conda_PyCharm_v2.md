# MCP Weather Quickstart on Windows
## Conda + uv + PyCharm + Local MCP Client + MCP Inspector

This README documents the working setup for the official MCP Python quickstarts:

- `weather-server-python/weather.py` — MCP weather server
- `mcp-client-python/client.py` — local MCP client
- Conda — Python environment
- `uv pip` — dependency installation
- PyCharm — normal execution
- MCP Inspector — browser-based MCP testing
- GNU Make on Windows — convenience commands

The design uses **one Conda environment for both the MCP client and MCP server**. `uv` installs packages but does not control runtime execution.


# 1. Final architecture

## 1.1 PyCharm client flow

```text
┌──────────────────────────────┐
│           PyCharm            │
│      Run config: client      │
└──────────────┬───────────────┘
               │
               ▼
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
               │
               ▼
         ┌─────────────┐
         │  client.py  │
         │ MCP client  │
         └──────┬──────┘
                │
                │ StdioServerParameters
                │ command=sys.executable
                ▼
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
                │
                ▼
         ┌─────────────┐
         │ weather.py  │
         │ MCP server  │
         └──────┬──────┘
                │
                ▼
       https://api.weather.gov
```

The local MCP transport is:

```text
client.py
   │
   │ MCP over stdin/stdout
   ▼
weather.py
```

There is no local TCP port in this configuration.


## 1.2 MCP Inspector flow

```text
┌─────────────────────┐
│ Browser              │
│ MCP Inspector UI     │
└──────────┬──────────┘
           │
           ▼
@modelcontextprotocol/inspector
           │
           │ launches stdio server
           ▼
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
           │
           ▼
      ┌─────────────┐
      │ weather.py  │
      │ MCP server  │
      └──────┬──────┘
             │
             ├── get_alerts
             └── get_forecast
```

`client.py` is **not involved** when using Inspector. Inspector itself acts as the MCP client.


# 2. Repository layout

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
    ├── uv.lock
    └── Makefile
```

Conda environment:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\
└── python.exe
```


# 3. Conda environment

Create:

```powershell
conda create -n quickstart-resources python=3.12 -y
```

Activate:

```powershell
conda activate quickstart-resources
```

Verify:

```powershell
python --version
python -c "import sys; print(sys.executable)"
where.exe python
```

Expected interpreter:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
```


# 4. Conda + uv responsibility split

```text
Conda
  ├── environment isolation
  └── Python interpreter

uv pip
  └── Python package installation
```

Dependency flow:

```text
pyproject.toml
      │
      ▼
   uv pip
      │
      ▼
Conda environment
```

For this setup, avoid using `uv run` for normal execution. PyCharm and `sys.executable` choose the runtime interpreter.


# 5. Install dependencies

Activate the environment:

```powershell
conda activate quickstart-resources
```

Client dependencies:

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\mcp-client-python
uv pip install -r pyproject.toml
```

Server dependencies:

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\weather-server-python
uv pip install -r pyproject.toml
```

Verify client:

```powershell
python -c "import anthropic; import mcp; import dotenv; print('Client OK')"
```

Verify server:

```powershell
python -c "import mcp; import pydantic; print('Server OK')"
```

Check installed packages:

```powershell
uv pip list
```


# 6. Required client.py change for pure Conda runtime

The upstream quickstart may launch Python MCP servers through `uv`, conceptually:

```python
server_params = StdioServerParameters(
    command="uv",
    args=["--directory", str(path.parent), "run", path.name],
    env=None,
)
```

For this setup, use the exact interpreter that PyCharm used to launch the client.

Add:

```python
import sys
```

Then use:

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

Runtime result:

```text
PyCharm
   ↓
Conda python.exe
   ↓
client.py
   ↓
sys.executable
   ↓
same Conda python.exe
   ↓
weather.py
```


# 7. PyCharm Run Configuration

Open:

```text
Run
→ Edit Configurations
```

Use:

```text
Name:
client

Interpreter:
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe

Script:
M:/ATOL/GitProjects/quickstart-resources/mcp-client-python/client.py

Script parameters:
M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py

Working directory:
M:/ATOL/GitProjects/quickstart-resources/mcp-client-python

Environment variables:
PYTHONUNBUFFERED=1

Path to .env:
M:/ATOL/GitProjects/quickstart-resources/mcp-client-python/.env
```

The Script parameters field is essential. If it is empty, the client prints:

```text
Usage: python client.py <path_to_server_script>
```


# 8. Anthropic API key

Create:

```text
M:\ATOL\GitProjects\quickstart-resources\mcp-client-python\.env
```

with:

```env
ANTHROPIC_API_KEY=sk-ant-your-key-here
```

Add to `.gitignore`:

```gitignore
.env
```

Verify:

```powershell
python -c "from dotenv import load_dotenv; import os; load_dotenv(); print('API key loaded:', bool(os.getenv('ANTHROPIC_API_KEY')))"
```


# 9. Running the client

Click PyCharm:

```text
▶ Run
```

A successful connection looks like:

```text
Connected to server with tools: ['get_alerts', 'get_forecast']

MCP Client Started!
Type your queries or 'quit' to exit.

Query:
```

Example:

```text
What weather alerts are active in California?
```

Successful tool activity can include:

```text
Processing request of type ListToolsRequest
Processing request of type CallToolRequest
HTTP Request: GET https://api.weather.gov/alerts/active/area/CA "HTTP/1.1 200 OK"

[Calling tool get_alerts with args {'state': 'CA'}]
```


# 10. Query flow

```text
Natural-language query
        │
        ▼
     client.py
        │
        ▼
       LLM
        │
        │ sees MCP tools
        ▼
 tool selection
        │
        ▼
get_alerts / get_forecast
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
tool result
        │
        ▼
       LLM
        │
        ▼
natural-language answer
```

Examples:

```text
What weather alerts are active in California?
```

```text
Are there dangerous weather alerts in Texas?
```

```text
What's the forecast for Seattle, Washington?
```

```text
What's the forecast for Sacramento, California?
```

Exit:

```text
quit
```


# 11. MCP tools

Expected tools:

```text
get_alerts
get_forecast
```

Direct tool arguments:

## get_alerts

```json
{
  "state": "CA"
}
```

## get_forecast

```json
{
  "latitude": 47.6062,
  "longitude": -122.3321
}
```


# 12. Running weather.py directly

Test startup:

```powershell
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe `
  M:\ATOL\GitProjects\quickstart-resources\weather-server-python\weather.py
```

The process can appear idle because it is waiting on stdio for MCP protocol messages.

```text
weather.py
   ↓
wait for stdin MCP messages
```

Stop with:

```text
Ctrl+C
```

Do not manually keep the server running before starting `client.py`; the client spawns its own server process.


# 13. stdio logging rule

For a stdio MCP server:

```text
stdout = MCP protocol
```

Avoid:

```python
print("debug")
```

Use stderr:

```python
import sys
print("debug", file=sys.stderr)
```

Interpreter debugging:

```python
print("SERVER PYTHON:", sys.executable, file=sys.stderr)
```


# 14. MCP Inspector

Inspector gives you a browser UI for MCP development and testing.

It can:

```text
discover tools
inspect tool schemas
manually invoke tools
inspect protocol behavior
```

It is not the same as the LLM-driven chat client.

Comparison:

```text
Local client:
natural language → LLM → MCP tool selection → weather.py

Inspector:
human selects tool → supplies arguments → MCP → weather.py
```


# 15. Windows Node / npx setup

Verify:

```powershell
node --version
npx.cmd --version
where.exe npx
where.exe npx.cmd
```

On Windows GNU Make, use:

```text
npx.cmd
```

rather than relying on a Unix/Cygwin `npx` wrapper.


# 16. Why the first Inspector Makefile failed

The first version effectively did:

```text
GNU Make
   ↓
/bin/bash
   ↓
/cygdrive/c/Program Files/nodejs/npx
   ↓
ERROR
```

Error:

```text
/bin/bash: /cygdrive/c/Program Files/nodejs/npx: No such file or directory
```

The project is Windows-native, so force GNU Make to use Windows `cmd.exe`:

```makefile
SHELL := C:/Windows/System32/cmd.exe
.SHELLFLAGS := /C
```


# 17. Why the absolute quoted npx.cmd path failed

A later form using:

```makefile
"C:/Program Files/nodejs/npx.cmd"
```

produced:

```text
'\"C:/Program Files/nodejs/npx.cmd\"' is not recognized as an internal or external command
```

GNU Make + `cmd.exe` quoting around the `.cmd` file was the issue.

The robust solution for this machine is:

```makefile
npx.cmd
```

resolved from Windows `PATH`.


# 18. Final working Makefile

```makefile
SHELL := C:/Windows/System32/cmd.exe
.SHELLFLAGS := /C

PYTHON := C:/Users/dante/miniconda3/envs/quickstart-resources/python.exe

ROOT := M:/ATOL/GitProjects/quickstart-resources
CLIENT := $(ROOT)/mcp-client-python/client.py
WEATHER_SERVER := $(ROOT)/weather-server-python/weather.py

.PHONY: help check client server inspector

help:
	@echo Available commands:
	@echo   make check
	@echo   make client
	@echo   make server
	@echo   make inspector

check:
	@echo Checking Python...
	@"$(PYTHON)" -c "import sys; print(sys.executable)"
	@echo Checking Node...
	@node --version
	@echo Checking NPX...
	@npx.cmd --version

client:
	@"$(PYTHON)" "$(CLIENT)" "$(WEATHER_SERVER)"

server:
	@"$(PYTHON)" "$(WEATHER_SERVER)"

inspector:
	@echo Starting MCP Inspector...
	@echo Python: $(PYTHON)
	@echo Server: $(WEATHER_SERVER)
	@npx.cmd -y @modelcontextprotocol/inspector $(PYTHON) $(WEATHER_SERVER)
```

Important: the Inspector command intentionally does not add extra quotes around `$(PYTHON)` and `$(WEATHER_SERVER)` because these paths contain no spaces.


# 19. Start MCP Inspector

Run:

```powershell
make inspector
```

Expected beginning:

```text
Starting MCP Inspector...
Python: C:/Users/dante/miniconda3/envs/quickstart-resources/python.exe
Server: M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py
```

The effective command is:

```text
npx.cmd -y @modelcontextprotocol/inspector C:/Users/dante/miniconda3/envs/quickstart-resources/python.exe M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py
```


# 20. Correct Inspector server card

The server card should show:

```text
python.exe
```

not:

```text
python.exe"
```

It should show:

```text
STDIO
Standard I/O
```

and the command should begin with:

```text
C:/Users/dante/miniconda3/envs/quickstart-resources/python.exe
```

A trailing quote in `python.exe"` means the command-line quoting is wrong.


# 21. Read-only Inspector session

When Inspector is launched with an ad-hoc server command, the UI can show:

```text
Read-only session
```

and explain that the server list was launched with `--config` or an ad-hoc server.

This is expected.

It means the server definition can be used in the current session but is not editable/persisted from that screen.

```text
Read-only session
      │
      ├── server can still connect
      ├── tools can still be inspected
      └── tool calls can still be executed
```


# 22. Connecting Inspector

The card can initially show:

```text
Disconnected
```

Click the toggle.

Desired state:

```text
Connected
```

Flow:

```text
Browser
  ↓
MCP Inspector
  ↓
stdio
  ↓
Conda python.exe
  ↓
weather.py
```

Once connected, inspect:

```text
get_alerts
get_forecast
```


# 23. Inspector tests

For `get_alerts`:

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

Inspector calls the MCP server directly; the Anthropic API key and `client.py` are not required for these manual tool calls.


# 24. Useful Make commands

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


# 25. Troubleshooting

## `ModuleNotFoundError: No module named 'anthropic'`

Install:

```powershell
uv pip install anthropic
```

or all client dependencies:

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\mcp-client-python
uv pip install -r pyproject.toml
```

## PyCharm says `<No interpreter>`

Select:

```text
C:\Users\dante\miniconda3\envs\quickstart-resources\python.exe
```

## `Usage: python client.py <path_to_server_script>`

Set PyCharm Script parameters to:

```text
M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py
```

## `[WinError 2]` from the MCP child process

If `client.py` uses:

```python
command="uv"
```

replace with:

```python
command=sys.executable
```

## `/bin/bash: /cygdrive/.../npx: No such file or directory`

Use:

```makefile
SHELL := C:/Windows/System32/cmd.exe
.SHELLFLAGS := /C
```

and:

```makefile
npx.cmd
```

## `\"C:/Program Files/nodejs/npx.cmd\" is not recognized`

Do not hardcode that quoted path in this Makefile. Use `npx.cmd` via `PATH`.

## Inspector displays `python.exe"`

Remove the extra quoting from the Inspector recipe.

Use:

```makefile
@npx.cmd -y @modelcontextprotocol/inspector $(PYTHON) $(WEATHER_SERVER)
```

## Inspector shows `Read-only session`

Expected for an ad-hoc Inspector launch.

## Inspector shows `Disconnected`

Toggle it on. If it immediately disconnects, inspect the terminal running `make inspector` for the actual server startup error.


# 26. Recommended workflows

## Normal application development

```text
PyCharm
   ↓
Run "client"
   ↓
client.py
   ↓
same Conda Python launches weather.py
   ↓
type natural-language queries
```

## MCP protocol/tool debugging

```text
Terminal
   ↓
make inspector
   ↓
Browser Inspector
   ↓
toggle Connected
   ↓
Tools
   ↓
get_alerts / get_forecast
```


# 27. Final system schematic

```text
                       ONE CONDA ENVIRONMENT
                               │
          ┌────────────────────┴────────────────────┐
          │                                         │
          ▼                                         ▼
   ┌───────────────┐                       ┌────────────────┐
   │ PyCharm       │                       │ MCP Inspector  │
   │ client.py     │                       │ Browser UI     │
   └───────┬───────┘                       └───────┬────────┘
           │                                       │
           │ sys.executable                        │ stdio launch
           │ MCP stdio                             │
           ▼                                       ▼
   ┌───────────────┐                       ┌────────────────┐
   │ weather.py    │                       │ weather.py     │
   │ MCP server    │                       │ MCP server     │
   └───────┬───────┘                       └───────┬────────┘
           │                                       │
           └──────────────────┬────────────────────┘
                              ▼
                     ┌──────────────────┐
                     │ api.weather.gov  │
                     └──────────────────┘
```

Normally use one server instance per test path.


# 28. Design summary

```text
Conda
  = environment + interpreter

uv pip
  = dependency installer

PyCharm
  = normal client launcher

sys.executable
  = guarantees client and server use the same Conda interpreter

MCP stdio
  = local client/server transport

MCP Inspector
  = browser-based MCP tool/protocol UI

cmd.exe + npx.cmd
  = Windows-safe Make execution for Inspector
```


# 29. Security

Never commit:

```text
.env
```

Never hardcode:

```text
ANTHROPIC_API_KEY
```

Recommended `.gitignore`:

```gitignore
.env
```


# 30. Quick reference

Activate:

```powershell
conda activate quickstart-resources
```

Check interpreter:

```powershell
python -c "import sys; print(sys.executable)"
```

Install client:

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\mcp-client-python
uv pip install -r pyproject.toml
```

Install server:

```powershell
cd M:\ATOL\GitProjects\quickstart-resources\weather-server-python
uv pip install -r pyproject.toml
```

PyCharm script:

```text
M:/ATOL/GitProjects/quickstart-resources/mcp-client-python/client.py
```

PyCharm parameter:

```text
M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py
```

Run client:

```text
▶ Run
```

Example query:

```text
What weather alerts are active in California?
```

Launch Inspector:

```powershell
make inspector
```

Inspector:

```text
Disconnected
   ↓ toggle
Connected
   ↓
Tools
```

Test:

```text
get_alerts
```

with:

```json
{
  "state": "CA"
}
```
