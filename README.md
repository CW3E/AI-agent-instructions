# AI Agent Guides

Context and instruction guides for AI assistants working with CW3E computing systems, processes, & procedures.

These guides provide the environment details and local conventions an assistant needs before coding, providing advice, generating documentation, etc.

## Available guides

| Guide | Scope |
| --- | --- |
| [Coding for AWARE](AWARE_GENAI_CODING_GUIDANCE.md) | AWARE environment, Slurm resources and seasonal reservations, BeeGFS storage, node-local scratch, concurrency, I/O, and development workflows. |

## Using a guide

1. Select the guide for the system or workflow you are working on.
2. Attach it to the conversation, paste its contents, or make the file available to your coding agent. Explicitly ask the assistant to read and apply it before starting.
3. Provide the specific task, relevant code, and any additional workload or project requirements.

For example:

> Read `AWARE_GENAI_CODING_GUIDANCE.md` and apply it to this task. The code will run on AWARE. Account for the available Slurm resources, shared filesystem load, and concurrent jobs when proposing changes.

For tools that support [AGENTS.md](https://agents.md/), a project's `AGENTS.md` can instruct the agent to read the relevant guide at an accessible path. Loading behavior varies by tool; putting a guide in this repository does not ensure an assistant will read it automatically.

## Maintaining the guides

Update a guide when the system configuration, scheduling policies, or recommended workflows change. Submit corrections and additions through pull requests.

- Keep each guide focused on its named system, workflow, process, or procedure.
- Distinguish confirmed configuration from recommendations, assumptions, and illustrative examples.
- Include sources and review dates where they help establish whether information is current.
- Explain the reason for constraints when that helps an assistant make an appropriate implementation choice.
- Keep changes focused and avoid duplicating configuration details across multiple guides.

Current site administrator instructions take precedence over older guidance. If an unresolved conflict or missing detail would materially change the implementation, ask for clarification before proceeding.
