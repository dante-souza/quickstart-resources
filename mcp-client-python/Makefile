uv:
	pip install uv

requirements:
	uv pip install -r pyproject.toml


install: uv requirements

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