# aos-client-sdk — Boardroom Workflow SDK Extensions

## Objective

Extend the AOS Client SDK to support the generic boardroom workflow system,
including MCP app payload delivery, workflow YAML loading with owner agent
resolution, step navigation, and validation against the boardroom schema.

## Context

The boardroom supports multiple YAML-defined workflows, each owned by a
specific agent.  The SDK provides the runtime infrastructure to load any
workflow, validate it, navigate between steps, and deliver MCP app payloads
to the frontend.  The same SDK APIs support both structured workflows and
dynamic CXO discussions.

## Requirements

### 1. MCP App Payload API

Add `send_app_payload()` to the client interface:

```python
await request.client.send_app_payload(
    app_id="boardroom_ui",
    payload={
        "narrative": "...",
        "response": "...",
        "actions": [...],
        "navigation": {"next": "step_id", "back": "step_id"},
    },
)
```

This sends the payload to the connected client session via the AOS SSE
transport layer.  The payload is also routed to `subconscious.asisaga.com`
for persistence.

### 2. Workflow YAML Loader

Add a utility for loading and validating any boardroom workflow YAML file:

```python
from aos_client.workflow import load_workflow_yaml

workflow = load_workflow_yaml("docs/workflow/samples/pitch.yaml")
step = workflow.get_step("paul_graham_intro")
owner = workflow.owner  # "founder"
workflow_id = workflow.workflow_id  # "pitch_business_infinity"
```

The loader must support all registered workflows:
- `pitch.yaml` (owner: founder)
- `onboarding.yaml` (owner: coo)
- `marketing.yaml` (owner: cmo)
- `crisis-response.yaml` (owner: ceo)
- `strategic-review.yaml` (owner: ceo)
- `product-launch.yaml` (owner: ceo)

Validation must ensure:
- All steps contain a `narrative` field
- All steps contain a `response` field
- All steps contain an `actions` array (may be empty)
- Navigation references valid step IDs
- Workflow has `workflow_id`, `version`, and `owner`

### 3. Step Navigation Helper

Add a step navigation helper that resolves `next`/`back` commands:

```python
from aos_client.workflow import StepNavigator

navigator = StepNavigator(workflow)
next_step = navigator.resolve("cmd:next", current_step_id="paul_graham_intro")
# → "paul_graham_dataset"
```

### 4. WorkflowRequest Extensions

Extend `WorkflowRequest` to expose the workflow YAML context when available:
- `request.workflow_steps` — loaded steps from the associated YAML
- `request.current_step_id` — current step from session state
- `request.workflow_owner` — owner agent ID from the YAML
- `request.workflow_id` — workflow identifier

### 5. Workflow Registry Integration

Provide a method to discover available workflows:

```python
from aos_client.workflow import list_available_workflows

workflows = list_available_workflows()
# [{"workflow_id": "pitch_business_infinity", "owner": "founder", ...}, ...]
```

### 6. Session State Management

Expose `subconscious.asisaga.com` MCP tools for session state via the client
interface so workflows can persist and resume boardroom sessions:

```python
# Set session state when a structured workflow starts
await request.client.set_session_state(
    session_id="session-123",
    state={
        "mode": "structured",
        "workflow_id": "pitch_business_infinity",
        "step_id": "paul_graham_intro",
        "owner": "founder",
        "agents": ["founder"],
        "status": "active",
    },
)

# Retrieve session state to resume after reconnection
state = await request.client.get_session_state(session_id="session-123")
current_step = state.get("step_id")
```

For dynamic CXO discussions the ``mode`` is ``"dynamic"`` and ``workflow_id``,
``step_id``, and ``owner`` are ``null``.

These calls are routed through the AOS SSE transport to
``subconscious.asisaga.com`` for durable storage.  The session state schema
is defined in `docs/workflow/pr/subconscious.asisaga.com/Readme.md`.

### 7. Boardroom State Context

Extend `OrchestrationRequest` to carry agent-level and boardroom-level
JSON-LD state so agents enter each orchestration with full situational
awareness:

```python
from aos_client import OrchestrationRequest, BoardroomContext

req = OrchestrationRequest(
    agent_ids=agent_ids,
    purpose=OrchestrationPurpose(purpose=..., purpose_scope=...),
    context=request.body,
    boardroom_context=BoardroomContext(
        # Per-agent static context: read-only identity and mandate
        agent_contexts={"ceo": {...}, "cfo": {...}, ...},
        # Per-agent dynamic content: mutable working memory and intent
        agent_contents={"ceo": {...}, "cfo": {...}, ...},
        # Per-agent ASI Saga and Business Infinity perspective state
        agent_company_states={"ceo": {...}, "cfo": {...}, ...},
        agent_product_states={"ceo": {...}, "cfo": {...}, ...},
        # Canonical shared manifests
        company_manifest={"@id": "asi:saga", ...},
        product_manifest=[{"@id": "bi:product:core", ...}, ...],
        # Collective boardroom state: topic, resonance, active directives
        boardroom_state={
            "status": "Operational",
            "current_topic": "...",
            "resonance_ledger": {...},
            "active_directives": [...],
        },
    ),
)
status = await request.client.submit_orchestration(req)
```

The `BoardroomContext` is injected into each agent turn so agents receive
their mutable ``content`` at the start of every response.  Their static
``context`` is carried as a read-only system prompt amendment, while
company and product state are available both as canonical manifests and as
per-agent perspective projections.

State files live in ``boardroom/state/`` (JSON-LD format) and are managed
by `BoardroomStateManager` in ``src/business_infinity/boardroom.py``.

## Dependencies

- `pyyaml` — YAML parsing for workflow definitions
- `jsonschema` — Workflow YAML validation against boardroom schema

## References

→ **Backend prompt**: `docs/workflow/prompts/backend.md`
→ **Boardroom schema**: `docs/workflow/boardroom.yaml`
→ **Workflow YAML samples**: `docs/workflow/samples/`
→ **Repository spec**: `.github/specs/repository.md`
→ **Multi-repo roadmap**: `docs/multi-repository-implementation.md`
