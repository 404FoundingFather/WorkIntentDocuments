**WID**

Work Intent Documents

*Rethinking Software Development Workflow for the Agent Era*

The Problem

Traditional project management tools like Jira were designed around
**human development velocity**. Tickets assume a developer needs
structured context, acceptance criteria, and status ceremony spread
across hours or days of work. This model is breaking down.

The rise of AI coding agents (Claude Code, Cursor, Copilot Workspace,
etc.) has fundamentally shifted the economics of software development.
Effective cycle time per unit of work has collapsed from hours to
minutes. The result: **the project management layer itself has become
the primary bottleneck**.

Key Friction Points

-   **Lossy compression of intent.** Engineers have rich design
    conversations with AI assistants, then must compress that semantic
    richness into a flat Jira ticket. Context, reasoning, trade-off
    discussions, and architectural rationale are lost.

-   **Granularity mismatch.** A well-scoped story for a human developer
    represents multiple discrete agent invocations. Decomposing further
    into agent-sized tickets creates more overhead than the work itself.

-   **Status ceremony as bottleneck.** Manually transitioning tickets
    through workflow states, writing comments, and linking PRs takes
    longer than the agent takes to produce the implementation.

-   **Redundant intermediary.** The ticket becomes a translation layer
    between the engineer's mental model and the agent's input---adding
    latency with no value.

The Vision: Work Intent Documents

WID is a development workflow system built from the ground up for the AI
agent era. Instead of tickets, the atomic unit of work is a semantically
rich, living document---the Work Intent Document---that captures the
full context of what needs to be done, why, and how decisions were made.

Core Principles

1\. The WID Is the Source of Truth

A WID is a structured, version-controlled document (Markdown-native,
Git-backed) that preserves the complete reasoning chain: problem
statement, constraints, architectural context, edge cases, trade-off
discussions, and acceptance criteria. No lossy compression. The same
artifact a human reads for context is what an agent consumes for
execution.

2\. Living Documents, Not Frozen Tickets

WIDs evolve over time. Mutations are tracked with semantic changelogs
that capture not just what changed but why and what initiated the change
(human edit, agent discovery, test failure). Changes propagate
downstream to executing agents, dependent WIDs, and derived views.

3\. Conversation-First Authoring

You don't fill out forms---you think through the problem with an AI
assistant, and the system structures the output into a WID. The design
conversation is the authoring interface.

4\. Derived Views, Not Manual Updates

Kanban boards, sprint summaries, status dashboards, burndown
charts---these are projections of WID state, generated automatically
from Git activity, CI results, and agent execution logs. No one manually
drags cards or updates status fields.

5\. Agent-Native by Default

A WID is structured so that an autonomous agent can pick it up cold and
execute without human hand-holding. The system supports 24/7 agent
pipelines where work is continuously picked up, executed, validated, and
completed.

System Architecture

The WID platform consists of five core components:

  ----------------------- -------------------------------------------------
  **Component**           **Description**

  **WID Engine**          Storage, versioning, and schema for WIDs.
                          Markdown-native with structured frontmatter for
                          machine-readable metadata. Git-backed for full
                          history and branching.

  **Conversation-to-WID   Transforms AI design sessions into structured
  Pipeline**              WIDs. Engineers think through problems
                          conversationally; the system formalizes the
                          output.

  **Execution Runtime**   Agent orchestration layer that watches for
                          actionable WIDs, assigns execution, monitors
                          progress, and writes results back. Supports
                          interactive and autonomous 24/7 operation.

  **Projection Layer**    Generates stakeholder-facing views: kanban
                          boards, burndown charts, status reports, PR
                          summaries. All derived from WID state and CI/CD
                          activity. Zero manual status updates.

  **Mutation Bus**        Reactive event layer handling WID changes and
                          propagating effects: notify executing agents,
                          update projections, re-evaluate dependent WIDs,
                          trigger downstream workflows.
  ----------------------- -------------------------------------------------

The WID Lifecycle

A WID progresses through four phases, each of which can be driven by
humans, agents, or both:

1.  **Ideation & Design.** Conversational session with an AI assistant
    to explore the problem space. The conversation itself is the first
    draft of the WID.

2.  **Refinement.** The WID is structured, validated, and linked to
    codebase context (files, modules, APIs). Acceptance criteria and
    constraints are formalized.

3.  **Execution.** An agent (or human) picks up the WID and works it.
    Execution logs, commits, and test results attach back to the WID
    automatically.

4.  **Completion.** Acceptance criteria are validated (potentially by a
    review agent). The WID transitions to done. Derived views update.
    Knowledge is captured for future reference.

Market Opportunity

Every engineering organization adopting AI agents is about to hit the
same wall: their project management tooling was not designed for this
paradigm. The market is moving from AI-assisted coding to AI-driven
development, and the workflow layer has not kept pace.

Adoption Strategy

WID is designed for dual-mode adoption:

-   **Full platform replacement.** Organizations can adopt the complete
    WID system as their primary workflow tool, replacing Jira and
    similar tools entirely.

-   **Incremental integration.** Organizations can layer WID alongside
    existing tools (Jira, Linear, etc.), using bidirectional sync to
    gradually migrate. WIDs become the rich source of truth while legacy
    tools receive projected views for organizational continuity.

This incremental path is critical for enterprise adoption, where ripping
out Jira on day one is rarely feasible.

Differentiation from Existing Tools

  ----------------- ----------------- ----------------- -----------------------
  **Dimension**     **Jira / Linear** **GitHub Issues** **WID**

  Primary artifact  Flat ticket with  Issue + markdown  **Semantically rich
                    fields            body              living document**

  Agent readiness   Bolt-on           API-accessible    **Agent-native;
                    integrations      but not native    designed as agent
                                                        input**

  Context           Lossy; reasoning  Partial; limited  **Full reasoning chain
  preservation      lost              to issue body     preserved**

  Status tracking   Manual state      Manual            **Inferred from
                    transitions       labels/close      activity (zero
                                                        manual)**

  Authoring model   Form-based        Form + markdown   **Conversation-first,
                                                        AI-assisted**

  Mutation model    Edit fields; no   Edit body; Git    **Semantic changelog
                    semantic history  history           with propagation**
  ----------------- ----------------- ----------------- -----------------------

Next Steps

5.  **Define the WID schema.** Formalize the Markdown + frontmatter
    structure that serves as both human-readable documentation and
    agent-executable specification.

6.  **Build a prototype WID Engine.** Git-backed storage with CLI
    tooling for creating, querying, and mutating WIDs.

7.  **Implement Conversation-to-WID pipeline.** Integration with Claude
    Code to formalize design sessions into WIDs directly from the
    terminal.

8.  **Prototype the Projection Layer.** Build a minimal dashboard that
    derives board views from WID state and Git activity.

9.  **Validate with internal use.** Dogfood at OscarPro to refine the
    model before external launch.

*The shift from AI-assisted coding to AI-driven development demands a
new workflow primitive. WID replaces the lossy, ceremony-heavy ticket
with a semantically rich, agent-native, living document that preserves
full context from ideation through execution. It's the workflow system
the agent era requires.*
