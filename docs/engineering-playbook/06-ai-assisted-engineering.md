# Module 6: AI-Assisted Engineering & Agentic Workflows

AI assistants can accelerate exploration, test design, scaffolding,
documentation, refactoring, and defect analysis. They do not replace
requirements ownership, security review, system understanding, or validation.
Treat generated output as an untrusted draft until it is checked against source
code, repository rules, tests, and operational constraints.

## 1. Context injection and repository rules

Generic prompts produce generic code. Enterprise repositories need durable
context that explains architecture, conventions, safety boundaries, and
validation expectations.

Rule mechanisms may include `.cursorrules`, `.cursor/rules/`, repository
instruction files, and approved assistant configuration. Follow the syntax and
scope rules of the selected tool; do not assume one tool will read another
tool's configuration automatically.

### Good repository rules

Rules should be specific, testable, and scoped to the files they govern. Useful
rules may require that an assistant:

- inspect existing patterns before adding a helper;
- preserve the Page Object Model directory structure;
- use API-driven fixtures for heavy data setup;
- prefer accessible roles, labels, and stable test IDs;
- avoid unconstrained terminal loops and arbitrary sleeps;
- preserve typed API contracts and explicit error handling;
- validate state-machine transitions and RBAC negative paths;
- avoid secrets, unbounded destructive commands, and broad rewrites;
- run the smallest relevant lint, type-check, build, and test commands;
- report assumptions, changed files, and validation evidence.

### Rule design principles

- State the reason for a constraint when it prevents a tempting shortcut.
- Prefer a small number of high-value rules over contradictory instructions.
- Separate universal rules from frontend, backend, test, or documentation rules.
- Define what to do when evidence is missing: inspect, ask, or report uncertainty.
- Version and review rule files like production engineering documentation.
- Do not use rules as a substitute for CI enforcement or server-side security.

## 2. Guardrails against hallucination and infinite loops

AI systems may invent APIs, infer behavior from names, repeat failed commands,
or make a local symptom disappear without fixing the cause. Establish explicit
stopping and evidence rules.

### Evidence-first behavior

Require the assistant to:

1. identify the relevant files and current behavior;
2. cite the source lines or test output supporting its conclusion;
3. distinguish observed facts from hypotheses;
4. make the smallest complete change;
5. run a bounded validation command;
6. stop and report when validation contradicts the hypothesis.

The assistant must not claim tests passed, APIs exist, or deployments succeeded
without actual evidence.

### Loop prevention

Set a bounded number of attempts for a failing command. After repeated failures,
the workflow should switch to diagnosis and request a decision rather than
repeating the same edit. Preserve the error, command, changed files, and
attempt history so another engineer can continue.

Never allow an agent to:

- run an unbounded test or retry loop;
- repeatedly weaken assertions until a test passes;
- delete broad directories or reset unrelated work;
- suppress errors with empty catches or success-shaped fallbacks;
- rewrite unrelated files to resolve a local failure.

### State-machine alignment

Before changing workflow code or tests, ask the assistant to map entity states,
events, guards, roles, persistence, and invalid transitions. Check that the
proposed change preserves:

- valid lifecycle transitions;
- forbidden transitions;
- retry and duplicate-event behavior;
- concurrent update handling;
- persisted state and emitted events;
- cleanup and rollback behavior.

### RBAC boundary checks

Require negative authorization cases, not only happy paths. For each protected
operation, verify that:

- unauthenticated callers are rejected;
- authenticated low-privilege roles cannot mutate protected resources;
- tenant or organization boundaries are enforced;
- UI restrictions are backed by API authorization;
- direct API calls cannot bypass hidden or disabled controls.

### Locator guardrails

Prevent generated tests from using absolute XPath, generated CSS classes, or
positional selectors unless there is a documented reason. Prefer accessible
roles, labels, semantic attributes, and stable `data-testid` values. If a
stable locator does not exist, the assistant should recommend a product or
testability change instead of inventing a brittle selector.

## 3. Agentic QA workflow

An agentic workflow is a sequence of bounded tasks with explicit handoffs, not
an autonomous replacement for engineering judgment.

```text
context -> inspect -> plan -> implement -> validate
        -> review evidence -> document -> hand off
```

Give each stage a clear input, output, and stop condition. Keep implementation,
security review, and final approval separate when risk warrants it.

### Recommended phases

1. **Reconnaissance:** inspect repository structure, build scripts, rules, and
   related tests without changing files.
2. **Requirement extraction:** identify acceptance criteria, risk, affected
   boundaries, and unresolved questions.
3. **Plan:** list files, behavior changes, dependencies, and validation commands.
4. **Implementation:** make focused edits using existing helpers and patterns.
5. **Validation:** run targeted tests, lint, type checks, builds, and contract
   checks as appropriate.
6. **Review:** inspect the diff for scope, security, regressions, and evidence.
7. **Handoff:** summarize changed files, results, assumptions, and remaining risk.

Do not let an agent silently skip reconnaissance or validation because a change
appears simple.

## 4. Prompt engineering for QA

Precise prompts include scope, constraints, evidence, and a definition of done.
Ask for execution and artifacts, not vague advice.

### API fixture generation template

```text
Inspect the existing typed API client and fixture patterns.
Create setup for [entities] using API calls, not UI navigation.
Preserve foreign-key order: [dependency order].
Register every created ID for reverse-order cleanup.
Use unique test data and explicit failure handling.
Do not invent endpoints; cite the existing route and response types.
Validate with: [bounded command].
Report changed files and setup/teardown evidence.
```

### Negative test matrix template

```text
For requirement [ID], build a matrix across:
- role and tenant;
- authentication state;
- resource state;
- valid and invalid input;
- duplicate and concurrent requests;
- expected status, error code, and persisted outcome.
Compare the matrix with existing tests and identify uncovered cases.
Do not implement until the current API contract and state model are verified.
```

### Defect and RCA formatting template

```text
Use only the supplied trace, console, network, and server evidence.
Separate observed facts, hypotheses, and confirmed cause.
Format: summary, environment, reproduction, expected/actual,
impact, evidence, five-whys RCA, likely owner, and regression test.
Do not claim a root cause that is not supported by evidence.
Redact tokens, credentials, and personal data.
```

### Review prompt template

```text
Review the diff for functional regressions, security issues, flaky
synchronization, API-contract drift, state-machine violations, and scope creep.
Return only actionable findings with file, line, severity, confidence,
reasoning, and a minimal remediation.
Verify each finding against the repository; do not speculate.
```

## 5. Human-in-the-loop validation and ownership

AI-generated tests, fixtures, architecture rules, and defect reports are drafts.
The engineer owning the subsystem remains accountable for:

- requirement interpretation;
- security and privacy decisions;
- compatibility and migration safety;
- test data and environment impact;
- code review and approval;
- production release decisions.

### Verification gates

Before accepting generated work:

1. Read the complete diff, including files the assistant added or modified.
2. Compare behavior with requirements and existing conventions.
3. Inspect authorization, validation, error, retry, and logging paths.
4. Run targeted tests and type checks.
5. Run broader regression where risk or shared infrastructure requires it.
6. Check for secrets, unsafe dependencies, generated licenses, and data leakage.
7. Review the final report against actual command output.

Do not approve code that the reviewer cannot explain or operate independently
of the assistant.

## 6. Safe handling of information and tools

Do not send credentials, private keys, access tokens, sensitive customer data,
or unnecessary proprietary content to an external model. Redact examples and
use approved enterprise tooling for confidential code.

Tool access should follow least privilege. Agents should use read-only
inspection by default and receive write, network, deployment, or destructive
permissions only when the task requires them. Protect:

- secret files and environment variables;
- production credentials and deployment commands;
- customer data and logs;
- destructive database or filesystem operations;
- package installation and supply-chain changes.

Review generated commands before execution. Never allow a prompt or code comment
to override repository security controls.

## 7. AI-assisted code review checklist

- Does the change implement the requirement rather than a proxy?
- Are claims supported by source or command evidence?
- Does it preserve behavior outside the requested scope?
- Are state transitions, RBAC boundaries, and tenant isolation covered?
- Are API contracts, error handling, retries, and idempotency correct?
- Are selectors resilient and synchronization explicit?
- Are tests meaningful, isolated, and non-flaky?
- Does the change introduce secrets, unsafe dependencies, or excessive access?
- Are generated files, licenses, and documentation appropriate?
- Does CI enforce the important guarantees independently?

## 8. Knowledge capture and durable decisions

Chat history is not a durable engineering record. Capture important decisions
in repository documentation, issues, architecture records, or test plans.
Record:

- decision and context;
- alternatives considered;
- trade-offs and risks;
- owner and review date;
- affected requirements and systems;
- validation evidence and rollback plan.

Store reusable prompts and agent workflows only after reviewing them for
security, scope, and compatibility. Keep examples sanitized and version them
with the rules or tooling they govern.

## 9. Measuring AI-assisted engineering

Measure outcomes, not generated code volume. Useful signals include:

- lead time for a validated change;
- first-pass test and build success;
- escaped defects and rework;
- review findings per change;
- flaky-test and false-positive rates;
- time spent correcting generated output;
- security or privacy incidents;
- documentation and traceability completeness.

Compare assisted and unassisted work carefully; task complexity and engineer
experience are confounding factors. Use findings to refine prompts, rules,
validation gates, and training rather than rewarding superficial speed.

## 10. Agent workflow checklist

- [ ] Repository rules and applicable conventions were inspected first.
- [ ] Scope, acceptance criteria, risks, and stop conditions are explicit.
- [ ] Existing APIs, state models, and test patterns were verified.
- [ ] API-driven setup, isolated data, and reverse-order cleanup are used where
      appropriate.
- [ ] Negative RBAC, invalid state, duplicate, and failure paths are considered.
- [ ] Generated locators are semantic, scoped, and stable.
- [ ] Commands and retries are bounded and non-destructive.
- [ ] Secrets and sensitive data are excluded or redacted.
- [ ] Validation evidence comes from actual tests, lint, type checks, or builds.
- [ ] A human owner reviewed and can explain the final change.
- [ ] Assumptions, limitations, and follow-up risks are documented.
