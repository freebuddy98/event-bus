### **Python Microservices Project Refactoring and Management Specification**

**Version**: 4.0 (Final)
**Date**: June 11, 2025

#### **Introduction**

This specification aims to provide a clear and complete set of instructions for an AI Agent to refactor an existing Python microservices project into a standardized, manageable, testable, and container-ready monorepo structure.

---

### **I. Overall Project Structure**

The project will adopt a monorepo model. All microservices and related code will reside in a single Git repository. The top-level directory should clearly separate service code from cross-service testing code.

```
my_project/
├── .dockerignore        # Files to be ignored by Docker builds
├── .gitignore           # Git ignore file, must include .venv/
├── Makefile             # The single entry point for all automated tasks
├── pyproject.toml       # [Root-level] For managing development tools for the entire project
├── poetry.lock          # [Root-level] Lock file for development tools
├── tests/               # Contains all cross-service testing code
│   ├── integration/     # Integration tests
│   │   ├── docker-compose.yml              # Defines all services and test scenarios (profiles)
│   │   ├── Dockerfile.test_runner        # Dockerfile for the test runner
│   │   └── test_*.py                       # Test scripts for various scenarios
│   └── e2e/             # End-to-end tests
│       └── ...
└── service_a/           # Microservice A
    ├── .venv/           # [Service-level] Python virtual environment for Service A (ignored by git)
    ├── Dockerfile         # Containerization config for Service A
    ├── docker-compose.dev.yml # Local development environment for Service A
    ├── pyproject.toml     # [Service-level] Manages runtime dependencies for Service A only
    ├── poetry.lock      # [Service-level] Lock file for Service A
    ├── tests/
    │   └── unit/        # Unit tests for Service A
    │       └── test_*.py
    └── src/             # Source code for Service A
```

---

### **II. Dependency Management Specification (Poetry)**

The project will use `Poetry` as the dependency management tool, configured to create virtual environments within each project's directory and following a two-level `pyproject.toml` management model.

**1. One-time Global Poetry Configuration**
This specification requires all developers to perform a one-time global configuration to ensure Poetry creates virtual environments within the project directory.
```bash
poetry config virtualenvs.in-project true
```
Subsequently, running `poetry install` or `poetry add` in any project root (i.e., where a `pyproject.toml` file resides) will automatically create a `.venv` folder in that directory to store the virtual environment.

**2. Service-Level `pyproject.toml`**
* **Responsibility**: To strictly manage the **runtime** dependencies required for that specific microservice, as well as its **unit testing** development dependencies.
* **Example (`/service_a/pyproject.toml`)**:
    ```toml
    [tool.poetry]
    name = "service-a"
    version = "0.1.0"
    description = "Handles order processing."
    authors = ["Your Team <team@example.com>"]

    [tool.poetry.dependencies]
    python = "^3.10"
    flask = "^3.0"
    redis = "^5.0"

    [tool.poetry.group.dev.dependencies]
    pytest = "^8.0"
    pytest-mock = "^3.12"
    ```

**3. Root-Level `pyproject.toml`**
* **Responsibility**: To uniformly manage the **global** tools used during **development and CI/CD** across the entire project.
* **Example (`/pyproject.toml`)**:
    ```toml
    [tool.poetry]
    name = "my-project-devtools"
    version = "0.1.0"
    description = "Development tools for the monorepo."
    authors = ["Your Team <team@example.com>"]

    [tool.poetry.group.dev.dependencies]
    python = "^3.10"
    black = "^24.0"
    ruff = "^0.4.0"
    pytest = "^8.0"
    requests = "^2.31" # For integration/E2E test scripts
    ```

---

### **III. Containerization Specification (Docker)**

#### **3.1. `Dockerfile` Best Practices**
Each microservice must have a `Dockerfile` that uses **multi-stage builds** to create small and secure production images.

**`/service_a/Dockerfile` Template:**
```dockerfile
# --- Stage 1: Builder ---
FROM python:3.10-slim-bookworm as builder
WORKDIR /app
RUN pip install poetry
COPY poetry.lock pyproject.toml ./
# --no-root: Does not install the project itself; --without dev: does not install dev dependencies
RUN poetry install --no-interaction --no-ansi --without dev --no-root

# --- Stage 2: Final Image ---
FROM python:3.10-slim-bookworm
RUN useradd --create-home --shell /bin/bash appuser
WORKDIR /home/appuser/app
USER appuser
# Copy the installed virtual environment from the builder stage
COPY --from=builder --chown=appuser:appuser /app/.venv ./.venv
# Add the virtual environment's bin directory to the PATH
ENV PATH="/home/appuser/app/.venv/bin:$PATH"
# Copy the application source code
COPY --chown=appuser:appuser ./src ./src
EXPOSE 5000
CMD ["flask", "run", "--host=0.0.0.0"]
```

#### **3.2. Local Development Environment (`docker-compose.dev.yml`)**
Each service directory should provide a `docker-compose.dev.yml` for local development, using a **bind mount** for live code reloading.

**`/service_a/docker-compose.dev.yml` Example:**
```yaml
version: '3.9'
services:
  service-a:
    build: .
    ports: ["5001:5000"]
    volumes: ["./src:/home/appuser/app/src"] # Mount local src directory into the container
    environment:
      - FLASK_ENV=development
```
**Usage**: From within the `/service_a` directory, run `docker-compose -f docker-compose.dev.yml up`.

#### **3.3. Integration Testing Environment (Unified `docker-compose.yml` with Profiles)**
A single `docker-compose.yml` in the `/tests/integration/` directory, using `profiles`, will manage all integration test scenarios.

**`/tests/integration/docker-compose.yml` Example:**
```yaml
version: '3.9'
services:
  service-a:
    build: ../../service_a
    profiles: ["test-ab", "test-ac"]

  service-b:
    build: ../../service_b
    profiles: ["test-ab"]

  service-c:
    build: ../../service_c
    profiles: ["test-ac"]

  test-runner-ab:
    build:
      context: ../.. 
      dockerfile: ./tests/integration/Dockerfile.test_runner
    profiles: ["test-ab"]
    depends_on: [service-a, service-b]
    command: pytest test_ab_interaction.py

  test-runner-ac:
    # ... similar structure for another test scenario
```

**`/tests/integration/Dockerfile.test_runner` Example:**
```dockerfile
FROM python:3.10-slim-bookworm
WORKDIR /app
RUN pip install poetry
COPY pyproject.toml poetry.lock ./
# Install all development dependencies, including pytest, requests, etc.
RUN poetry install --no-interaction --no-ansi --only dev
COPY . .
```

---

### **IV. Testing Strategy and Organization**

A three-tiered testing strategy of **Unit, Integration, and End-to-End tests** will be adopted.

**1. Unit Tests**
* **Goal**: To test a single function, class, or module in isolation.
* **Location**: Within each service at `/<service_name>/tests/unit/`.
* **Key Points**: All external dependencies must be mocked. They should be fast and cover core business logic.

**2. Integration Tests**
* **Goal**: To verify the collaboration and communication contracts between a small group of real microservices.
* **Location**: In the root `/tests/integration/` directory.
* **Key Points**: Uses `docker-compose` with `profiles` to launch the required subset of services for testing.

**3. End-to-End (E2E) Tests**
* **Goal**: To simulate a real user journey and verify that a complete business workflow succeeds.
* **Location**: In the root `/tests/e2e/` directory.
* **Key Points**: Runs against a fully deployed staging environment. The number of tests should be minimal, covering only the most critical business scenarios.

---

### **V. Development and Testing Workflow**

This section defines the standard workflow for developers.

**1. Developing/Debugging/Unit Testing a Single Microservice (e.g., `service_a`)**
* **Steps**:
    1.  Enter the service directory: `cd service_a`
    2.  Install dependencies (first time): `poetry install`
    3.  Activate the virtual environment: `source .venv/bin/activate`
    4.  Develop, debug, or run unit tests in the activated environment: `flask run` or `pytest tests/unit`
    5.  Deactivate the environment when done: `deactivate`

**2. Running Global Code Quality Checks**
* **Steps**:
    1.  Ensure you are in the project **root** directory.
    2.  Install development tools (first time): `poetry install`
    3.  Execute commands via `poetry run` or `make`: `poetry run black .` or `make format`

**3. Running Integration or End-to-End Tests**
* **Steps**:
    1.  Ensure you are in the project **root** directory.
    2.  **No virtual environment activation is needed.** Simply run the `make` command. The Makefile and Docker will handle the environment.
    3.  Example: `make test-integration-ab`

---

### **VI. Automation Workflow Specification (Makefile)**

The root `Makefile` serves as the single entry point for all automated tasks.

```makefile
# Makefile in project root
.PHONY: help test-unit test-integration-ab test-integration-ac test-integration test-e2e clean lint format test-all

# Define all microservice directories
SERVICES := service_a service_b

# Find the poetry command automatically
POETRY := $(shell command -v poetry 2> /dev/null)

# ==============================================================================
# Default Target: Print Help Information
# ==============================================================================
help: ## Display this help message
	@echo "Usage: make <target>"
	@echo ""
	@echo "Available targets:"
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-25s\033[0m %s\n", $$1, $$2}'

# ==============================================================================
# Development Helper Tasks
# ==============================================================================
lint: ## (all) Run the linter on all service code
	@echo "--- Running linter for all services ---"
	$(POETRY) run ruff check .

format: ## (all) Format all service code
	@echo "--- Formatting code for all services ---"
	$(POETRY) run black .
	$(POETRY) run ruff format .

# ==============================================================================
# Testing Tasks
# ==============================================================================

SERVICE ?= $(SERVICES)

test-unit: ## (unit) Run unit tests for a specific service (or all)
	@for service in $(SERVICE); do \
		echo "--- Running unit tests for $$service ---"; \
		(cd ./$$service && $(POETRY) run pytest tests/unit); \
	done

test-integration-ab: ## (integration) Run integration tests for Service A <-> B
	@echo "--- Running integration tests for Service A <-> B ---"
	(cd ./tests/integration && docker-compose --profile test-ab up --build --abort-on-container-exit --exit-code-from test-runner-ab)

test-integration-ac: ## (integration) Run integration tests for Service A <-> C
	@echo "--- Running integration tests for Service A <-> C ---"
	(cd ./tests/integration && docker-compose --profile test-ac up --build --abort-on-container-exit --exit-code-from test-runner-ac)

test-integration: test-integration-ab test-integration-ac ## (integration) Run all defined integration test scenarios
	@echo "--- All integration tests passed ---"

test-e2e: ## (e2e) Run end-to-end tests
	@echo "--- Running End-to-End tests ---"
	(cd ./tests/e2e && docker-compose up --build --abort-on-container-exit --exit-code-from test-runner)

test-all: test-unit test-integration ## (all) Run all unit and integration tests
	@echo "--- All essential tests passed ---"
```

### **VII. CI/CD Guiding Principles**

1.  **On Pull Request**: The pipeline must run `make lint` and `make test-unit`.
2.  **On Merge to main/develop**: The pipeline should run `make test-integration` to execute all defined integration test scenarios.
3.  **Build Production Images**: In the pipeline, navigate to each service directory and run `docker build -t my-registry/service-name:tag .`.
4.  **After Deploying to Staging**: The pipeline can trigger `make test-e2e`.
5.  **Deploy to Production**: Deploy the tested and verified images to the production environment.