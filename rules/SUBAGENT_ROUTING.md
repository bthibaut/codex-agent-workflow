# Subagent Routing

Default to cost-aware delegation for repository work. The user has explicitly authorized automatic subagent use to conserve the main agent's allowance. Do not delegate trivial conversation, a single bounded read, a tiny answer, or work where spawn/context overhead is likely to cost more than doing it directly.

Keep the main context lean: delegate raw discovery and bounded implementation, request concise reports, and avoid sending the full conversation when a task-local prompt is sufficient.

Resolve custom-agent configurations from `$CODEX_HOME/agents` when `CODEX_HOME` is set; otherwise use `~/.codex/agents`. Select agents by their exact `name`:

- Repository discovery, broad searches, contract tracing, and read-only investigation -> `code-explorer`
- Advice on complex architectural decisions with significant tradeoffs -> `architecture-advisor`
- Evidence-based advice on a bounded technical impasse -> `technical-advisor`
- Mechanical, well-specified changes limited to one or two files -> `quick-implementer`
- Feature or bug-fix implementation with appropriate tests and validation -> `implementer`
- Review of completed code changes -> `code-reviewer`
- Commit and push of completed changes, only when the user explicitly requests both -> `commit-pusher`

Do not substitute the built-in `default`, `worker`, or `explorer` agent when the corresponding custom agent is available. If a requested custom agent cannot be selected, state that clearly before using a generic subagent.

Agent identity and display labels:

- Select a custom agent by its exact agent `name`, not by inventing a task alias.
- A task/thread label or generated nickname is presentation metadata; it does not prove which custom role, model, or reasoning effort was used.
- Before every delegation, announce the selected custom agent name, model, reasoning effort, and whether the agent is being created or reused in a user-visible progress message.
- For a new agent, use: `Delegating to custom <agent> — <model>, <reasoning> reasoning (new agent).`
- For an existing agent, use: `Delegating to custom <agent> — <model>, <reasoning> reasoning (reusing existing agent).` This applies to follow-up tasks, resumed work, and messages assigning additional work, even when the agent is already running. Routine status requests and coordination messages that assign no work do not require a delegation announcement.
- Read model and reasoning values from the active custom-agent configuration or runtime metadata; do not infer them from the task label or from a requested override that the custom role may ignore.
- If the agent's runtime metadata differs from the delegation announcement, correct the user-visible record immediately.
- In the final task summary, list each custom agent that materially contributed, whether newly created or reused, with its actual model and reasoning effort.
- If runtime metadata shows no custom role (for example, `agent_role` is null), describe it as a generic subagent rather than attributing a custom-agent personality or configuration to it.

Agent lifecycle and delegation briefs:

- For follow-up work on the same task and role, prefer reusing an existing agent while its context remains relevant.
- For a new or unrelated task, or when the existing context is stale or overloaded, create a fresh agent with a concise task-local brief.
- Reuse is a continuity choice; do not assume it saves tokens or guarantees lower cost.
- Every delegation, whether creating or reusing an agent, must announce the selected custom agent, model, reasoning effort, and lifecycle state using the formats above.
- Give each work assignment a concise brief covering the objective, scope, files or symbols, constraints, expected validation, and relevant prior findings. Omit full conversation history when the brief is sufficient.
- Avoid frequent status polling. Follow runtime progress and notification constraints, and poll only when a meaningful handoff or decision requires it.

Default workflow for coding changes:

1. Handle trivial reads, answers, obvious edits, and targeted file-location searches directly when delegation overhead would dominate.
2. Use `code-explorer` only when the task needs broad discovery; skip it when target files are already known.
3. Use `quick-implementer` for explicit one- or two-file mechanical changes.
4. Use `implementer` for multi-file behavior changes, debugging, or work needing substantial tests.
5. Use `code-reviewer` after non-trivial or risk-bearing implementation. Skip review-agent spawning for purely mechanical documentation/config edits that the coordinator can validate cheaply.
6. Use `commit-pusher` only after the applicable review or direct validation, and only on an explicit request to commit and push.

## Mandatory explorer boundary

Use `code-explorer` when investigation requires cross-module discovery or tracing callers, data flow, contracts, conventions, or test coverage across the codebase.

An unknown file alone does not require delegation. The coordinator may use targeted searches to locate it and inspect known files directly. Delegate when those searches reveal a need for broad investigation; do not use a fixed command or file count as the trigger.

Do not use `code-explorer` merely to re-read a file already identified by the user or a previous agent report. Do not duplicate the explorer's searches after it returns; trust its cited report and inspect only the exact excerpts needed for the next decision.

## Cost and escalation policy

The configured coordinator default is Sol low. Sol medium can be selected for harder framing, or Astra for exceptionally complex coordination. These are explicit session/configuration choices, not automatic model changes implied by routing. Keep the chosen setting stable during a task where practical.

- Select the role whose scope fits the task and read its actual model and reasoning settings before delegation. Both implementation roles use Luna high; their distinction is scope and instructions, not a guaranteed cost or capability difference.
- Do not spawn multiple agents to solve the same problem unless independent review is justified.
- Prefer sequential handoffs with concise artifacts over parallel duplication.
- Escalate from `quick-implementer` to `implementer` when a design decision is required, the assigned scope expands, or a failure requires substantive diagnosis.
- Reserve `code-reviewer` (Astra low) for behavior changes, correctness, security, architecture, or other meaningful risks. Validate mechanical documentation/configuration edits directly when appropriate.
- Keep outputs concise and stop once evidence is sufficient; preserve the configured reasoning effort.

## Optional architectural consultation

`architecture-advisor` uses Astra medium and is consultative, read-only, and optional. Consultation is occasional, not a required phase of the default workflow.

Consult it only when a concrete complex decision involves significant tradeoffs between credible solutions, module boundaries or shared contracts, a data migration, or a hard-to-reverse structural choice. A task spanning multiple files, routine design work, a difficult bug, or an implementer failure alone does not trigger consultation.

Provide the decision to resolve, goals, constraints, existing contracts, and the explorer's summary when available. Request a justified recommendation, credible alternatives, consequences, risks, and an appropriate migration or validation strategy. Do not invoke an explorer solely to prepare a consultation if the necessary context is already known.

The coordinator retains the decision and passes the chosen approach and rationale to the implementer. The advisor does not implement changes or replace independent code review. Reconsult only when new evidence or a changed constraint materially affects the decision.

## Optional technical consultation

`technical-advisor` uses Astra low and is optional. The coordinator may consult it when a targeted diagnosis fails to make progress, observations contradict the current hypotheses, or a technically difficult behavior remains unexplained. A routine doubt, syntax error, or missing dependency alone does not justify consultation.

The implementer reports the objective, expected and observed behavior, evidence, attempts and outcomes, and a precise question to the coordinator. The coordinator decides whether consultation is useful and sends a concise brief; implementers do not automatically contact advisors. Do not require repeated blind attempts or wait for two correction cycles when a concrete impasse is already established.

Ask for a justified hypothesis and a discriminating check with expected outcomes. The coordinator evaluates the advice and gives the implementer a precise next step; the implementer validates it. Reconsult only with new evidence or a materially changed question. Medium reasoning is an explicit option for a harder diagnosis, not an automatic escalation or a second default advisor.

If the diagnosis reveals a complex architectural choice, the coordinator decides whether the architectural advisor adds value; never invoke both advisors automatically. Exploration may precede architectural advice when existing contracts are unknown. Keep each advisor's context separate from the independent reviewer, even when they use the same model.

## Correction loop

Before the first review, the coordinator provides the baseline ref/commit or pre-existing state, target version (working tree, index, or commit), intended files/hunks, acceptance criteria, and available validation results. Account for applicable staged, unstaged, untracked, and committed changes without including unrelated user work. Record the reviewed target and findings so a later correction delta can be identified; if the delta cannot be reconstructed reliably, disclose that limitation and review the relevant scope.

1. The reviewer reports concrete findings with locations, impact, and the required outcome.
2. The implementer receives those findings and relevant context, makes scoped corrections, and reruns the relevant checks.
3. The coordinator supplies the previous findings, initial baseline, previously reviewed target, and updated target. The reviewer verifies the correction delta and affected behavior before issuing an updated verdict, widening the review when scope or risk materially changes. Keep the reviewer distinct from the implementer.
4. If the same difficulty remains after two correction attempts, the coordinator takes over diagnosis and chooses a different approach. A model change must be explicit and supported by the available configuration; switching implementation roles alone does not change the model.

Avoid parallel write-heavy delegation unless the work is divided into non-overlapping files or the user explicitly requests it.
