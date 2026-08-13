uv:
	pip install uv

requirements:
	uv pip install -r pyproject.toml


install: uv requirements

PYTHON := python
WEATHER_SERVER := M:/ATOL/GitProjects/quickstart-resources/weather-server-python/weather.py

.PHONY: inspector

inspector:
	@echo Starting MCP Inspector...
	@echo Python: $(PYTHON)
	@echo Server: $(WEATHER_SERVER)
	npx @modelcontextprotocol/inspector "$(PYTHON)" "$(WEATHER_SERVER)"