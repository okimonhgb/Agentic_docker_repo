# Workflow Builder

Enables users to create custom agent workflows through natural language conversation instead of writing JSON configs manually.

The workflow builder enables users to create custom agent workflows through natural language conversation. Instead of writing JSON configs by hand, users describe what they want the chatbot to do, and the system generates a complete LangGraph workflow configuration.

## Workflow builder agents

Defined in `agentic/agents/workflow_builder/`:

### Intent analyzer
[[agentic/agents/workflow_builder/intent_analyzer.py]] — Determines whether the user wants to create a new workflow, modify an existing one, or just chat.

### Capabilities agent
[[agentic/agents/workflow_builder/capabilities_agent.py]] — Explains available agent types, tools, and capabilities to the user.

### Clarification agent
[[agentic/agents/workflow_builder/clarification_agent.py]] — Asks targeted questions to refine ambiguous workflow requirements.

### Feasibility checker
[[agentic/agents/workflow_builder/feasibility_checker.py]] — Validates that a proposed workflow is technically feasible with available agents and tools.

### Graph generator
[[agentic/agents/workflow_builder/graph_generator.py]] — Produces the final JSON graph configuration from the collected requirements.

### Proposal summarizer
[[agentic/agents/workflow_builder/proposal_summarizer.py]] — Presents the generated workflow in human-readable form for user approval.

### Workflow coordinator
[[agentic/agents/workflow_builder/workflow_coordinator.py]] — Orchestrates the overall workflow creation process, managing state transitions between agents.

## Router

[[agentic/agents/workflow_builder/router.py#decide_next_action]] — Central routing logic that decides which agent should handle the next turn based on workflow creation progress.

## State

[[agentic/workflow_builder/state.py]] defines workflow-builder-specific state fields including the accumulated workflow specification, agent selections, and tool configurations.

## Integration with graph engine

Generated workflows are saved and immediately available for execution via the graph engine.

Generated workflows are saved via `POST /api/workflow/save-with-id` and stored in the `workflows` database table. They are immediately available for execution through the graph engine's `build_graph()` with the stored config. See [[concepts/graph-engine]] for execution details.
