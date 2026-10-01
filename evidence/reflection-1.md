# Reflection Brief — Harness Engineering Capstone (Rewritten Version)

## Name
Mani Teja G.

## Date
2026-09-25

## Environment

### Models Used
System 1 was evaluated with Claude Haiku 4.5 and Claude Sonnet 4.5 across multiple recorded runs. System 2 used the default model configuration supplied with the project. System 3 operated as a static validation environment without an active model, while System 4 relied on a prerecorded offline response fixture.

### Platform
Linux environment running in a code-server workspace with Python 3.13.

### Approximate Cost
System 1 incurred approximately $0.65 in API usage across the evaluated runs. Systems 3 and 4 generated no model cost, and System 2 did not expose cost information in its output.

---

## Part 1 – Per-System Analysis

### System 1 – Agentic Loop

The agent workflow is controlled through tool-driven state transitions rather than text inspection. During execution, the model repeatedly generates responses, requests tools when needed, and terminates only after reaching a valid end state. The implementation checks the model stop reason after every interaction. When tool usage is requested, the harness executes the tools, records the results, and continues the loop. When an end-turn signal is returned, the workflow produces the final state. Any unexpected stop condition is treated as an error.

A major design decision is avoiding control logic based on natural-language output. Tests explicitly prevent routing decisions from being inferred through string matching. During one run, the model produced explanatory text and immediately ended the conversation without invoking a routing tool. If the system relied on phrases inside the text, it could incorrectly conclude that routing had occurred. Instead, the harness marked the task as incomplete because no terminal action was actually executed.

The terminal tools have clearly separated responsibilities. Routing tools finalize a claim by assigning it to the correct queue, while escalation tools are reserved for situations involving uncertainty or insufficient evidence. Escalation captures structured information describing why confidence was low and which claim categories remained plausible. Tool failures are also represented as structured responses containing retryability metadata, allowing the agent to distinguish between temporary and permanent failures.

Cost analysis showed that claim processing expenses were driven primarily by repeated transmission of conversation history. Output token counts remained relatively small, whereas input tokens increased as history accumulated. Sonnet and Haiku produced similar token volumes, but Sonnet incurred a noticeably higher cost because of model pricing differences. Some runs also terminated early without invoking required terminal tools, contributing to incomplete outcomes.

### System 2 – Context Strategy

The context-reduction workflow substantially decreased prompt size while preserving information needed for evaluation. The active customer issue remained intact, whereas resolved segments were compressed into concise summaries. This approach reduced total token volume by more than half while maintaining critical details needed to answer evaluation questions.

The distinction between summarized and preserved content is intentional. Resolved conversations primarily require outcome information, whereas active conversations may still depend on exact wording, identifiers, or transaction details. Because of this difference, the active segment is protected from compression.

A separate facts block stores structured information that might otherwise disappear during summarization. Evaluation results demonstrated that some values can survive summarization, but status fields and other structured records are much more reliable when explicitly extracted into a dedicated facts section.

### System 3 – Claude Code Configuration

Path-based rules were selected because test files are distributed throughout the repository rather than being grouped in a single directory. A path-matching strategy allows testing standards to be applied consistently regardless of location. Multiple rule sets can also be layered together when a file matches several categories.

The deployment-check skill operates inside a forked context with a restricted tool allowlist. This architecture isolates diagnostic operations from the primary workflow and prevents unintended repository modifications. The skill can inspect repository state but cannot directly change source code.

Validation results confirmed that project-level and user-level configuration scopes are clearly separated. Shared project resources remain version controlled and available to all contributors, whereas personal commands and skills stay outside the repository and are not distributed across the team.

### System 4 – Orchestration

The orchestration layer minimizes prompt size by filtering data before it reaches the model. Queries retrieve only recently relevant records, ensuring that prompt growth remains controlled even as historical data accumulates. This design keeps long-running operations scalable.

Crash recovery behavior is governed by a time threshold. Recent incomplete runs are resumed, while older sessions are restarted from a clean state. This reduces the risk of continuing from stale assumptions and encourages retrieval of current information instead.

The hot-state file intentionally remains small. Only information required for future execution is preserved, while older data is moved into lower-cost storage tiers. This prevents state growth from gradually consuming model context and increasing operating costs.

---

## Part 2 – Synthesis

Three distinct layers appear throughout the project.

1. **Model Layer** – Responsible for reasoning, tool selection, and response generation.
2. **Harness Layer** – Converts model outputs into controlled actions through validation, tool execution, and safeguards.
3. **Orchestration Layer** – Manages state, recovery, scheduling, and information flow across sessions.

The distinction between deterministic controls and prompt-based guidance became clear during testing. Requirements such as termination checks, permission boundaries, and safety constraints are enforced in code. Judgment-oriented tasks remain the responsibility of the model. When prompt guidance failed, deterministic safeguards still prevented invalid completion states.

Systems 2 and 4 address context management from different perspectives. System 2 reduces context within a single conversation by summarizing completed segments while preserving active information. System 4 manages context across multiple sessions through tiered storage and selective retrieval. Both approaches share the same objective: provide the model with the minimum information required for the next decision while preserving accuracy where it matters most.

Testing also highlighted the importance of scenarios that rarely occur during normal execution. Recovery logic, atomic writes, and boundary conditions may not be visible in a successful run, yet they are essential for long-term reliability. Automated tests provide confidence that these paths behave correctly when failures eventually occur.

The overall impact of failures is intentionally constrained. Misclassification or incorrect routing can affect workflow quality, but multiple safeguards limit broader consequences. Budget limits, validation checks, exception handling, and completion requirements collectively reduce operational risk.

---

## Part 3 – Honest Assessment

### Challenges Encountered

The first obstacle involved dependency incompatibility between installed package versions. Resolving the issue required aligning library versions and rerunning validation tests. A second problem occurred because an invalid API key had been configured, preventing successful model access. After correcting the environment configuration, execution proceeded normally.

Even after resolving infrastructure issues, several runs still produced incomplete outcomes because the model occasionally ended conversations without invoking mandatory terminal tools. Rather than rerunning until obtaining ideal results, these outcomes were retained as evidence because they reflected actual system behavior.

### Improvements for Future Iterations

One improvement would be enhanced trace logging whenever the model finishes without performing a required terminal action. Capturing the final reasoning would simplify debugging and help identify why completion criteria were not satisfied.

Another enhancement would be introducing a deterministic reminder step that requests a terminal action before marking a task as incomplete. Such a mechanism could recover a portion of incomplete cases while preserving the tool-driven architecture and avoiding reliance on text parsing.
