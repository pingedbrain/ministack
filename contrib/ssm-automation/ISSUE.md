# SSM Documents + Automation Executions

## The gap

`ssm` is registered and listed as `available` in `/_ministack/health`, but
`services/ssm.py` implements only Parameter Store and SendCommand. Every
Documents and Automation call — the whole runbook surface — answers:

```json
{"__type": "InvalidAction", "message": "Unknown action: CreateDocument"}
```

The describe-then-run pattern (verify a document name exists before
starting an execution, then poll it to a terminal status) is the standard
consumer contract for runbook-driven automation. It cannot run locally
today.

## What was implemented

Branch `feat/ssm-documents-automation` on `pingedbrain/ministack` —
`services/ssm.py`, +375 lines, no new dependencies:

**Documents** — account+region scoped via `AccountRegionScopedDict` (the
same store the parameters use):

- `CreateDocument` — stores name/content/type/format verbatim;
  `ValidationException` on missing Name/Content,
  `InvalidDocumentType` on a type outside the modeled enum,
  `DocumentAlreadyExists` on a duplicate name
- `DescribeDocument` — `InvalidDocument` ("Document with name X does not
  exist.") when absent; full `Document` description otherwise
- `GetDocument` — name + version + `Content`
- `DeleteDocument` — `InvalidDocument` when absent
- `ListDocuments` — `DocumentIdentifiers` across the account+region

**Automation executions** — the same record→poll lifecycle pattern
`_send_command` already uses:

- `StartAutomationExecution` — `ValidationException` on missing
  DocumentName; `AutomationDefinitionNotFoundException` when the document
  is not seeded locally (the emulator holds no hosted AWS catalogue —
  callers create the runbooks they depend on). Returns
  `AutomationExecutionId`; a `spawn_background` finisher moves the record
  `InProgress` → `Success` after ~0.5s
- `GetAutomationExecution` — `AutomationExecutionNotFoundException` when
  absent; the full `AutomationExecution` shape otherwise
- `DescribeAutomationExecutions` — `AutomationExecutionMetadataList`
- `StopAutomationExecution` — `Cancelled` on a running execution

## How real AWS behaves (what the implementation follows)

From the botocore `ssm-2014-11-06` service model — the wire contract boto3
parses against:

- `StartAutomationExecution` error surface:
  `AutomationDefinitionNotFoundException`,
  `AutomationExecutionLimitExceededException`,
  `InvalidAutomationExecutionParametersException`,
  `AutomationDefinitionVersionNotFoundException`, `InvalidTarget`,
  `IdempotentParameterMismatch`, `InternalServerError`
- `DescribeDocument` errors: `InvalidDocument`, `InvalidDocumentVersion`
- `GetAutomationExecution` errors:
  `AutomationExecutionNotFoundException`
- `CreateDocument` errors: `DocumentAlreadyExists`,
  `InvalidDocumentContent`, `DocumentLimitExceeded`,
  `InvalidDocumentSchemaVersion`, `MaxDocumentSizeExceeded`,
  `TooManyUpdates`, `InternalServerError`
- `AutomationExecutionStatus` lifecycle: `Pending` → `InProgress` →
  `Waiting` → `Success` / `TimedOut` / `Cancelling` → `Cancelled` /
  `Failed` (19 values total in the enum)
- `DocumentStatus`: `Creating` → `Active` → `Updating` → `Deleting` /
  `Failed`
- Timestamp fields are **omitted** from responses until set — an empty
  string breaks the boto3 parser (`Invalid timestamp ""`)

The implementation honors each of these: error names match the model
verbatim, timestamps are only emitted once real values exist, and the
status vocabulary maps to the modeled enum values (`InProgress`, `Success`,
`Cancelled`).

## Verified behavior (the test run)

Against the fork on `:4567`, boto3 client, real calls:

```
create:  Demo-EnableBucketLogging  Active
describe: Demo-EnableBucketLogging Automation
describe missing → InvalidDocument
start missing → AutomationDefinitionNotFoundException
start:       050460be-42e6-471f-a512-6794984d7f1e
get poll:    InProgress → Success
list:        ['Demo-EnableBucketLogging']
stop:        Cancelled
```

End-to-end consumer flow — a real remediation executor (boto3, no
mocks) driving describe → start → poll:

```
[PASS] describe->start->poll->Success —
       {"execution_type": "ssm", "status": "success",
        "execution_id": "9f7af780-..."}
```

Fault-injection check — a proxy returning the modeled
`AutomationExecutionLimitExceededException` (429) between the caller and
the emulator:

```
StartAutomationExecution → AutomationExecutionLimitExceededException (429)
SDK behavior: "reached max retries: 4" — real retry/backoff exercised
Executor records: {"status": "error", "error": "AutomationExecutionLimitExceededException..."}
```

## Deliberately out of scope

- Executing a runbook's `mainSteps` against emulated resources — the
  emulation is state-level: `Success` means "accepted and completed",
  which is the contract consumers poll for
- `UpdateDocument`, `ListDocumentVersions` (versioning beyond `1`),
  `DescribeDocumentPermission`, PatchManager, Maintenance Windows,
  OpsItems — separate contributions if wanted

## Evidence basis (per CONTRIBUTING.md)

- **botocore service model** (`ssm-2014-11-06`) for request/response
  shapes, error codes, enum values, and `httpStatusCode` per error
- **Existing MiniStack code** for patterns: `AccountRegionScopedDict`,
  `spawn_background`, `error_response_json`, SendCommand's record→poll
  lifecycle
- **boto3 client behavior** verified live against the implementation —
  parser rejects empty timestamps, retries fire on injected
  `AutomationExecutionLimitExceededException`
- **NOT validated against real AWS** — no AWS account was used. If the
  maintainers want observed-AWS evidence before code review, this issue is
  the artifact and the branch can stay in the fork meanwhile
