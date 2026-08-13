# ==============================================================================
# MCP Quickstart - Makefile
# ==============================================================================
#
# Usage:
#   make help
#   make check
#   make install
#   make server
#   make client
#   make inspector
#   make mcp-weather
#   make mcp-list
#   make mcp-weather-info
#
# ==============================================================================


# ------------------------------------------------------------------------------
# Shell
# ------------------------------------------------------------------------------

SHELL := C:/Windows/System32/cmd.exe
.SHELLFLAGS := /C


# ------------------------------------------------------------------------------
# Tools
# ------------------------------------------------------------------------------

PYTHON := C:/Users/dante/miniconda3/envs/quickstart-resources/python.exe
UV := uv
NPX := npx.cmd
CLAUDE := claude


# ------------------------------------------------------------------------------
# Project paths
# ------------------------------------------------------------------------------

# Makefile lives in:
#   quickstart-resources/mcp-client-python/
#
# Therefore ".." is the quickstart-resources root.

ROOT := $(abspath ..)

CLIENT_DIR := $(ROOT)/mcp-client-python
SERVER_DIR := $(ROOT)/weather-server-python

CLIENT := $(CLIENT_DIR)/client.py
WEATHER_SERVER := $(SERVER_DIR)/weather.py


# ------------------------------------------------------------------------------
# MCP configuration
# ------------------------------------------------------------------------------

MCP_NAME := weather


# ------------------------------------------------------------------------------
# Phony targets
# ------------------------------------------------------------------------------

.PHONY: \
	help \
	uv \
	requirements \
	install \
	check \
	client \
	server \
	inspector \
	mcp-weather \
	mcp-list \
	mcp-weather-info


# ==============================================================================
# Help
# ==============================================================================

help:
	@echo.
	@echo MCP Quickstart
	@echo ==============
	@echo.
	@echo Installation:
	@echo   make uv                 Install uv
	@echo   make requirements       Install project dependencies
	@echo   make install            Install uv and dependencies
	@echo.
	@echo Development:
	@echo   make check              Check required tools
	@echo   make server             Start Weather MCP server
	@echo   make client             Start MCP client
	@echo   make inspector          Start MCP Inspector
	@echo.
	@echo Claude MCP:
	@echo   make mcp-weather        Add Weather MCP server to Claude Code
	@echo   make mcp-list           List Claude MCP servers
	@echo   make mcp-weather-info   Show Weather MCP configuration
	@echo.


# ==============================================================================
# Installation
# ==============================================================================

uv:
	@echo Installing uv...
	@pip install uv


requirements:
	@echo Installing project dependencies...
	@$(UV) pip install -r pyproject.toml


install: uv requirements
	@echo.
	@echo Installation complete.


# ==============================================================================
# Environment checks
# ==============================================================================

check:
	@echo.
	@echo Checking Python...
	@"$(PYTHON)" -c "import sys; print(sys.executable)"
	@echo.
	@echo Checking uv...
	@$(UV) --version
	@echo.
	@echo Checking Node...
	@node --version
	@echo.
	@echo Checking NPX...
	@$(NPX) --version
	@echo.
	@echo Checking Claude Code...
	@$(CLAUDE) --version
	@echo.
	@echo Environment check complete.


# ==============================================================================
# MCP application
# ==============================================================================

server:
	@echo.
	@echo Starting Weather MCP server...
	@echo Python: $(PYTHON)
	@echo Server: $(WEATHER_SERVER)
	@echo.
	@"$(PYTHON)" "$(WEATHER_SERVER)"


client:
	@echo.
	@echo Starting MCP client...
	@echo Client: $(CLIENT)
	@echo Server: $(WEATHER_SERVER)
	@echo.
	@"$(PYTHON)" "$(CLIENT)" "$(WEATHER_SERVER)"


# ==============================================================================
# MCP Inspector
# ==============================================================================

inspector:
	@echo.
	@echo Starting MCP Inspector...
	@echo Python: $(PYTHON)
	@echo Server: $(WEATHER_SERVER)
	@echo.
	@$(NPX) -y @modelcontextprotocol/inspector "$(PYTHON)" "$(WEATHER_SERVER)"


# ==============================================================================
# Claude Code MCP
# ==============================================================================

mcp-weather:
	@echo.
	@echo Adding Weather MCP server to Claude Code...
	@echo.
	@echo MCP name: $(MCP_NAME)
	@echo Python:   $(PYTHON)
	@echo Server:   $(WEATHER_SERVER)
	@echo.
	@$(CLAUDE) mcp add \
		--scope project \
		--transport stdio \
		$(MCP_NAME) \
		-- \
		"$(PYTHON)" \
		"$(WEATHER_SERVER)"
	@echo.
	@echo Weather MCP server configured.
	@echo Run "make mcp-list" to verify.


mcp-list:
	@echo.
	@echo Claude MCP servers:
	@echo.
	@$(CLAUDE) mcp list


mcp-weather-info:
	@echo.
	@echo Weather MCP configuration:
	@echo.
	@$(CLAUDE) mcp get $(MCP_NAME)