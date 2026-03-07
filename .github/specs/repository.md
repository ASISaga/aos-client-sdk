# aos-client-sdk Repository Specification

**Version**: 1.0.0  
**Status**: Active  
**Last Updated**: 2026-03-07

## Overview

`aos-client-sdk` is the application framework and Python SDK for building **Azure Functions apps** powered by the **Agent Operating System (AOS)**. Client applications use this SDK to define business workflows, communicate with AOS via HTTP and Azure Service Bus, handle authentication, register with AOS for infrastructure provisioning, and deploy to Azure — while staying focused on business logic.

## Scope

- Repository role in the AOS ecosystem
- Technology stack, package layout, and coding patterns
- Testing and validation workflows
- Key design principles for contributors

## Repository Role

| Concern | Owner |
|---------|-------|
| SDK / framework: `AOSApp`, `AOSClient`, `AOSAuth`, `AOSServiceBus`, `AOSRegistration`, `AOSDeployer` | **aos-client-sdk** |
| Azure Functions scaffolding, HTTP/Service Bus triggers, auth middleware | **aos-client-sdk** (via `AOSApp`) |
| Agent lifecycle, perpetual orchestration, messaging, storage, monitoring | AOS infrastructure |
| Agent catalog (C-suite agents, capabilities) | RealmOfAgents |
| Business logic / workflow definitions | Client apps (e.g. `business-infinity`) |

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Python 3.10+ |
| Package | `aos-client-sdk` (`aos_client`) — version 7.0.0 |
| Core deps | `pydantic>=2.0.0`, `aiohttp>=3.9.0` |
| Azure extras | `azure-identity`, `azure-servicebus`, `azure-functions` |
| Tests | `pytest` + `pytest-asyncio` (asyncio_mode = auto) |
| Linter | `pylint` |
| Build / deploy | `azure.yaml` (Azure Developer CLI) |

## Directory Structure

```
aos-client-sdk/
├── src/
│   └── aos_client/
│       ├── __init__.py        # Public API — all exports live here
│       ├── app.py             # AOSApp, WorkflowRequest, workflow_template
│       ├── auth.py            # AOSAuth, TokenClaims
│       ├── client.py          # AOSClient — HTTP/Service Bus orchestration client
│       ├── deployment.py      # AOSDeployer, DeploymentResult
│       ├── foundry.py         # Foundry Agent Service integration (internal)
│       ├── gateway.py         # API gateway helpers
│       ├── identity.py        # Azure identity helpers
│       ├── mcp.py             # MCPServerConfig
│       ├── models.py          # All Pydantic models
│       ├── observability.py   # ObservabilityConfig, structured logging
│       ├── registration.py    # AOSRegistration, AppRegistration
│       ├── reliability.py     # Circuit breaker, retry, idempotency
│       ├── service_bus.py     # AOSServiceBus
│       └── testing.py         # MockAOSClient for local dev/tests
├── tests/                     # pytest unit tests (one file per module)
├── examples/                  # Usage examples
├── pyproject.toml             # Build config, dependencies, pytest settings
└── azure.yaml                 # Azure Developer CLI deployment config
```

## Core Patterns

### AOSApp Setup

```python
from aos_client import AOSApp, ObservabilityConfig

app = AOSApp(
    name="my-app",
    observability=ObservabilityConfig(
        structured_logging=True,
        correlation_tracking=True,
        health_checks=["aos", "service-bus"],
    ),
)
```

### Workflow Registration

```python
from aos_client import AOSApp, WorkflowRequest
from typing import Dict, Any

@app.workflow("workflow-name")
async def my_workflow(request: WorkflowRequest) -> Dict[str, Any]:
    agents = await request.client.list_agents()
    status = await request.client.start_orchestration(
        agent_ids=[a.agent_id for a in agents],
        purpose="Describe the perpetual goal",
        context=request.body,
    )
    return {"orchestration_id": status.orchestration_id, "status": status.status.value}
```

### Azure Functions Entry Point

```python
# function_app.py — zero boilerplate in client apps
from my_app.workflows import app
functions = app.get_functions()
```

### MockAOSClient for Testing

```python
from aos_client.testing import MockAOSClient

client = MockAOSClient()
# configure mock responses, then inject into workflow under test
```

## Testing Workflow

```bash
# Install dev dependencies
pip install -e ".[dev]"

# Run all tests
pytest tests/ -v

# Lint
pylint src/aos_client/ --disable=C0114,C0115,C0116

# Run a specific test file
pytest tests/test_app.py -v
```

**CI**: GitHub Actions runs `pytest` and `pylint` across Python 3.10, 3.11, and 3.12 on every push/PR to `main`.

→ **CI workflow**: `.github/workflows/ci.yml`

## Related Repositories

| Repository | Role |
|-----------|------|
| [business-infinity](https://github.com/ASISaga/business-infinity) | Example client application using this SDK |
| [aos-dispatcher](https://github.com/ASISaga/aos-dispatcher) | AOS Orchestration API |
| [aos-realm-of-agents](https://github.com/ASISaga/aos-realm-of-agents) | Agent catalog (C-suite) |
| [aos-kernel](https://github.com/ASISaga/aos-kernel) | OS kernel (orchestration, messaging, storage) |
| [purpose-driven-agent](https://github.com/ASISaga/purpose-driven-agent) | Agent base class |

## Key Design Principles

1. **Zero boilerplate for clients** — Client apps declare `@app.workflow`; the SDK handles all Azure Functions scaffolding
2. **Purpose-driven orchestrations** — Perpetual, described by intent not implementation
3. **Foundry-internal** — Foundry Agent Service is an AOS implementation detail, never exposed to SDK consumers
4. **Enterprise-ready** — Knowledge Base, Risk Registry, Audit Trail, Covenant Management, Analytics built in
5. **Testable** — `MockAOSClient` enables full unit testing without AOS infrastructure

## References

→ **Agent framework**: `.github/specs/agent-intelligence-framework.md`  
→ **Conventional tools**: `.github/docs/conventional-tools.md`  
→ **Python coding standards**: `.github/instructions/python.instructions.md`  
→ **Azure Functions patterns**: `.github/instructions/azure-functions.instructions.md`
