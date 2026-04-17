# aos-client-sdk

Application framework and Python SDK for building **Azure Functions apps** powered by the **Agent Operating System**. Client applications use this SDK to define business workflows, communicate with AOS via HTTP and Azure Service Bus, handle authentication, register with AOS for infrastructure provisioning, and deploy to Azure — all while staying focused on business logic.

## Overview

The AOS Client SDK provides:

- **`AOSApp`** — Azure Functions Blueprint framework with `@workflow` decorators.  Generates HTTP triggers, Service Bus triggers, health endpoints, and auth middleware automatically via `func.Blueprint`.
- **`AOSClient`** — Async HTTP/Service Bus client for agent discovery and orchestration.
- **`AOSAuth`** — Azure IAM authentication and role-based access control.
- **`AOSServiceBus`** — Bidirectional async communication via Azure Service Bus (scale-to-zero).
- **`AOSRegistration`** — Client app registration with AOS (triggers infrastructure provisioning).
- **`AOSDeployer`** — Code deployment to Azure Functions.

## Quick Start

```bash
pip install aos-client-sdk[azure]
```

### Define Workflows (business logic only)

```python
# workflows.py
from aos_client import AOSApp, WorkflowRequest

aos_app = AOSApp(name="my-app")

@aos_app.workflow("strategic-review")
async def strategic_review(request: WorkflowRequest):
    agents = await request.client.list_agents()
    c_suite = [a.agent_id for a in agents if a.agent_type in ("LeadershipAgent", "CMOAgent")]
    return await request.client.start_orchestration(
        agent_ids=c_suite,
        purpose="strategic_review",
        context=request.body,
    )
```

### Azure Functions entry point (Blueprint pattern)

```python
# function_app.py
import azure.functions as func
from my_app.workflows import aos_app

bp = aos_app.get_blueprint()
app = func.FunctionApp()
app.register_blueprint(bp)
```

That's it.  The SDK creates all HTTP triggers, Service Bus triggers, health
endpoints, and authentication middleware as a Blueprint that is registered
with the Azure Functions consumption-plan-compatible ``FunctionApp``.

## AOS Ecosystem

The SDK is the client-facing layer of the Agent Operating System.  It
communicates with deployed AOS services over HTTP and Azure Service Bus.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Client Application (e.g. business-infinity)                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  workflows.py       @app.workflow decorators               │   │
│  │  function_app.py    app.register_blueprint(bp)           │   │
│  │    └─ aos-client-sdk provides Blueprint with triggers    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  Zero Azure Functions boilerplate.                                   │
│  Zero agent code. Zero infrastructure code.                          │
└──────────────┬───────────────────────┬──────────────────────────────┘
               │ HTTPS                 │ Azure Service Bus
               ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Agent Operating System (AOS)                                        │
│                                                                      │
│  ┌─────────────────────┐  ┌────────────────────┐  ┌──────────────┐  │
│  │  aos-dispatcher     │  │  realm-of-agents   │  │    mcp       │  │
│  │  POST /orchestrations│  │  GET /realm/agents │  │  MCP server  │  │
│  │  Service Bus trigger │  │  Agent catalog     │  │  registry    │  │
│  └──────────┬──────────┘  └────────────────────┘  └──────────────┘  │
│             │                                                        │
│  ┌──────────▼─────────────────────────────────────────────────────┐  │
│  │  aos-kernel                                                    │  │
│  │  FoundryAgentManager · FoundryOrchestrationEngine              │  │
│  │  Messaging · Storage · Auth · Reliability · Observability      │  │
│  └──────────────────────────────┬─────────────────────────────────┘  │
│                                 │                                    │
│  ┌──────────────────────────────▼─────────────────────────────────┐  │
│  │  aos-intelligence                                              │  │
│  │  LoRA adapters · DPO training · RAG · Multi-LoRA inference     │  │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### How the SDK Connects to the Ecosystem

| Component | Connection | How |
|-----------|-----------|-----|
| **aos-dispatcher** | HTTP REST + Service Bus | `AOSClient` calls `/api/orchestrations`, `/api/knowledge/*`, etc. |
| **realm-of-agents** | HTTP REST | `AOSClient.list_agents()` calls `/api/realm/agents` |
| **mcp** | HTTP REST via dispatcher | `AOSClient.list_mcp_servers()` and `call_mcp_tool()` |
| **aos-kernel** | Optional Python package | `foundry.py` module delegates to `FoundryAgentManager` when installed |
| **aos-intelligence** | Optional Python package | LoRA routing enabled via `LoRAOrchestrationRouter` when installed |

### Optional AOS Package Extras

Install with AOS server-side packages when running AOS components in the
same process (embedded mode, local development, or custom deployments):

```bash
# Include aos-kernel for direct kernel integration
pip install aos-client-sdk[aos]

# Include aos-intelligence for LoRA/RAG/ML features
pip install aos-client-sdk[intelligence]

# Azure services (Service Bus, Functions, Identity)
pip install aos-client-sdk[azure]
```

When `aos-kernel` is installed, the SDK's internal Foundry transport layer
(`aos_client.foundry`) automatically integrates with the kernel's
`FoundryAgentManager` for agent lifecycle management and becomes the
underlying Foundry backend for `FoundryOrchestrationEngine`.

## API Reference

### `AOSApp` — Application Framework

| Method | Description |
|--------|-------------|
| `@app.workflow(name)` | Register a business workflow (creates HTTP + Service Bus triggers) |
| `app.get_blueprint()` | Build Azure Functions Blueprint with all registered triggers |
| `app.get_workflow_names()` | List registered workflow names |

### `AOSClient` — AOS Communication

| Method | Description |
|--------|-------------|
| `list_agents(agent_type=)` | List agents from the RealmOfAgents catalog |
| `get_agent(agent_id)` | Get a single agent descriptor |
| `submit_orchestration(request, via_service_bus=)` | Submit via HTTP or Service Bus |
| `get_orchestration_status(id)` | Poll orchestration status |
| `start_orchestration(...)` | Convenience: submit and return status (perpetual) |
| `stop_orchestration(id)` | Stop a perpetual orchestration |
| `cancel_orchestration(id)` | Cancel a running orchestration |
| `health_check()` | Check AOS health |

### `AOSAuth` — Authentication & Access Control

| Method | Description |
|--------|-------------|
| `validate_token(token)` | Validate Azure AD bearer token |
| `require_role(claims, role)` | Assert token has a specific role |
| `require_any_allowed_role(claims)` | Assert token has any allowed role |
| `get_credential()` | Get Azure credential for outbound calls |

### `AOSServiceBus` — Async Communication

| Method | Description |
|--------|-------------|
| `send_orchestration_request(request)` | Send orchestration via Service Bus |
| `parse_orchestration_result(body)` | Parse status update from Service Bus message |
| `parse_orchestration_status(body)` | Parse status from Service Bus message |

### `AOSRegistration` — App Registration & Provisioning

| Method | Description |
|--------|-------------|
| `register_app(name, workflows)` | Register with AOS (provisions infrastructure) |
| `get_app_status(name)` | Check registration/provisioning status |
| `deregister_app(name)` | Remove registration |

### `AOSDeployer` — Deployment

| Method | Description |
|--------|-------------|
| `deploy(slot=)` | Deploy to Azure Functions |
| `ensure_host_json()` | Create host.json if missing |
| `ensure_local_settings(...)` | Create local.settings.json if missing |

### Models

| Model | Description |
|-------|-------------|
| `WorkflowRequest` | Request passed to workflow handlers |
| `AgentDescriptor` | Agent metadata from the RealmOfAgents catalog |
| `OrchestrationRequest` | Request to start a perpetual agent orchestration |
| `OrchestrationStatus` | Current status of an orchestration (ACTIVE, STOPPED) |
| `AppRegistration` | Client app registration record |
| `DeploymentResult` | Deployment operation result |
| `TokenClaims` | Parsed Azure AD token claims |

## Authentication

The SDK integrates Azure IAM for authentication and access control:

```python
from aos_client import AOSApp, AOSAuth

auth = AOSAuth(
    tenant_id="your-tenant-id",
    client_id="your-client-id",
    allowed_roles=["Workflows.Execute"],
)

app = AOSApp(name="my-app", auth=auth)
```

For local development, omit the auth configuration for anonymous access.

## Service Bus Communication

Both AOS and client apps scale to zero and wake on Service Bus triggers:

```python
from aos_client import AOSClient

async with AOSClient(
    endpoint="https://my-aos.azurewebsites.net",
    service_bus_connection_string="Endpoint=sb://...",
    app_name="my-app",
) as client:
    # Submit via Service Bus (async, scale-to-zero friendly)
    status = await client.submit_orchestration(request, via_service_bus=True)
```

## App Registration

Register your app with AOS to provision infrastructure automatically:

```python
from aos_client import AOSRegistration

async with AOSRegistration(aos_endpoint="https://my-aos.azurewebsites.net") as reg:
    info = await reg.register_app(
        app_name="my-app",
        workflows=["strategic-review", "market-analysis"],
    )
    print(info.service_bus_connection_string)
```

## Related Repositories

- [aos-kernel](https://github.com/ASISaga/aos-kernel) — OS kernel (orchestration, messaging, storage, auth)
- [aos-intelligence](https://github.com/ASISaga/aos-intelligence) — ML intelligence layer (LoRA, DPO, RAG)
- [aos-dispatcher](https://github.com/ASISaga/aos-dispatcher) — AOS orchestration API (HTTP + Service Bus dispatcher)
- [realm-of-agents](https://github.com/ASISaga/realm-of-agents) — Agent catalog (RealmOfAgents function app)
- [mcp](https://github.com/ASISaga/mcp) — MCP server registry (Model Context Protocol function app)
- [business-infinity](https://github.com/ASISaga/business-infinity) — Example client application
- [purpose-driven-agent](https://github.com/ASISaga/purpose-driven-agent) — Agent base class

## License

Apache License 2.0 — see [LICENSE](LICENSE)
