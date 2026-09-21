# Submission

## What did you investigate first, and why?

I first ran the repository’s documented setup and verification commands to establish a clean baseline. After confirming that typechecking, the existing test suite, report generation, and a successful validation command all worked, I inspected the CLI, MCP adapter, Git helper, validation runner, and report generation paths.

I prioritized the CLI/MCP boundary and validation execution because they are central to the tool’s advertised behavior and are places where small contract mismatches can affect reliability. That led me to identify three issues: the MCP handler read `repoPath` even though the schema exposed `repo_path`, the CLI truncated repository paths containing spaces, and failed validation commands aborted the whole review instead of producing a failed `ValidationResult`.

## What did you choose to implement or fix?

I implemented three targeted fixes:

- Fixed the MCP repository-path mapping so the `repo_path` field defined in the MCP schema is correctly passed to `reviewRepository()`.
- Fixed CLI handling for repository paths containing spaces by removing logic that truncated the path at the first space.
- Changed failed validation commands to return a `ValidationResult` with `status: "failed"` instead of rejecting and aborting the entire review. I also updated the Markdown report to display each validation result’s status.

I chose these because they were concrete correctness and reliability issues with scoped, verifiable fixes and direct impact on the tool’s advertised interfaces.

## What did you intentionally not do?

I intentionally kept the scope narrow instead of trying to address every potential issue in the repository. I did not redesign validation execution, add command timeouts, change output buffering limits, or broadly refactor the Git/reporting code.

I left the existing validation-command execution model unchanged and focused instead on the concrete correctness and reliability issues I could verify within the assessment window.

Given the time limit, I prioritized three concrete issues that were reproducible, scoped, and verifiable.

## Interface decision

- Decision: hybrid

- Primary user and execution environment:
  The tool supports both developers running reviews directly from a local CLI and AI/tooling clients invoking the same review functionality through MCP. Both operate against a local Git repository.

- Trust boundary and allowed capabilities:
  Both interfaces ultimately call the same review core. Repository paths and validation commands are caller-provided, and validation commands are intentionally executed on the local machine. Because of that, MCP clients that can supply validation commands cross a local command-execution trust boundary and should only be given that capability in a trusted environment.

- Reliability, discoverability, latency/context, and output tradeoffs:
  The CLI is easier to discover, debug, and run manually, while MCP makes the same functionality available to AI clients. Sharing the same core implementation reduces behavioral drift between the two interfaces. I prioritized correctness issues in the supported interfaces and a shared reliability issue in validation behavior.

- How supported interfaces remain consistent:
  Both the CLI and MCP adapters call `reviewRepository()`, so shared repository inspection and validation behavior stays centralized. I also fixed interface-specific inconsistencies, including the MCP `repo_path` mapping and CLI handling of repository paths containing spaces.

- Evidence that would change this decision:
  If usage showed that nearly all users interact with the tool manually from a terminal, I would favor CLI-first. If most usage came from AI-agent workflows, I would favor MCP-first and place more emphasis on structured responses and stronger capability controls.

## How did you use an AI coding agent?

I used AI assistance primarily to help inspect the repository, identify possible correctness and reliability issues, and reason through targeted fixes. I did not ask the AI to broadly rewrite the repository. I checked the surrounding code before making changes, ran typechecking and tests after each fix, reviewed each Git diff, and committed the fixes separately.

## Where did you check, correct, or reject an AI suggestion? (required)

The AI initially identified failed validation commands as a possible bug because `runValidation()` rejected when a command exited unsuccessfully. I did not change it immediately because that behavior could have been intentional. I checked `types.ts`, `core.ts`, and `report.ts` first. `ValidationResult` explicitly supports both `"passed"` and `"failed"`, while the implementation could only return `"passed"`. After confirming that mismatch, I treated it as a real contract and reliability bug.

## Commands used to verify the result, with outcomes

- `npm run typecheck` — passed after each implemented fix.
- `npm test` — passed; the existing test suite completed successfully.
- `npm run inspector -- review --repo . --format markdown` — successfully generated `review-report.md`.
- `npm run inspector -- review --repo . --validate "npm test"` — successfully generated a report with a passing validation command.
- `npm run inspector -- review --repo . --validate "node -e ""process.exit(1)"""` — successfully generated a report instead of aborting, and the validation was shown as `(failed)`.
- `git diff` — used to verify that each change was limited to the intended files before committing.
- `git status` — confirmed the working tree was clean after the final verification.

## A blocker you hit and how you approached it

The repository had very limited automated test coverage, so passing the existing test suite was not enough to verify the behaviors I changed. I used typechecking and the existing test as regression checks, then manually exercised the CLI happy path and a deliberately failing validation command to verify the behavior end to end.

## Known limitations and the next three things you would do

Known limitations include validation commands having no explicit timeout, validation output potentially becoming large enough to hit buffering or report-size limits, and limited automated test coverage around the CLI, MCP, and validation paths.

Next, I would:
1. Add timeout handling for validation commands.
2. Add explicit output-size or truncation handling with clear reporting.
3. Add focused automated tests for CLI path parsing, MCP input mapping, and failed validation behavior.

## Approximate focused-work time

- Start: 12:29 PM
- Finish: 1:41 PM