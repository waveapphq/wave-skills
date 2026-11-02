---
name: wave-mcp
description: Use when working against a WAVE server through its MCP tools (get_recipe_guide, validate_recipe, create_test_from_recipe, amend_test_from_recipe, start_execution, get_execution, get_failure_report, reveal_secret, ...) - authoring or amending tests via recipes, organizing suites and regressions, managing profiles, variables, secrets, or a test's data files (list_testdata, write_testdata, ...), running executions, or diagnosing failed runs - or when connecting an MCP client to a WAVE server.
---

# Using the WAVE MCP

## Overview

WAVE embeds an MCP server that makes an AI assistant an ordinary test-engineering client. There is no privileged path: every call flows through the same services, validation, and secret masking as the web UI, and everything you do is attributed to your client and permanent until a human removes it.

Core principle: **never guess what the server will tell you.** The recipe format, the block catalog, and dry-run validation are all served by WAVE itself; fetch, then author, then validate, then persist.

Tool names below are bare. Clients usually prefix them with the registered server name (Claude Code: `mcp__wave__get_recipe_guide`, loaded on demand via ToolSearch when schemas are deferred).

Context note: when only a verdict or summary is needed from a large read-only payload (`get_recipe_guide`, `list_blocks`, `get_run_report`), fetch it via a subagent that returns the verdict, not inline. Authoring loops that consume the payload directly stay inline.

## Connecting a client

The endpoint is streamable HTTP at `http://<host>:<port>/mcp`, served by the WAVE backend. It is off by default: an administrator enables it in the MCP Server section of System Config, which also holds the endpoint settings (port, bind scope, optional HTTP Basic auth with a write-only password) and shows the exact numeric addresses, connected clients, and a live feed of tool calls. The bind scope defaults to local-only; reaching the endpoint from another machine requires the administrator to deliberately widen it (auth is recommended alongside). The MCP surface itself cannot report its own host or port; connection details come from that System Config section.

Any MCP client needs exactly: transport streamable HTTP, the URL above, and the `Authorization: Basic <base64 user:pass>` header when auth is on. Claude Code registration:

```bash
claude mcp add --transport http wave http://<host>:<port>/mcp \
  --header "Authorization: Basic <base64 user:pass>"   # header only when auth is on
```

Troubleshooting by symptom: connection refused = endpoint disabled or wrong port; 401 = auth is on and the header is missing or wrong; 404 = wrong path (the endpoint is exactly `/mcp`); 406 on a plain browser GET is normal (it is an MCP endpoint, not a web page). The connected-clients list in System Config confirms a client got through.

## Authoring a test

Tests are authored ONLY through recipes: one YAML document of intent (blocks, settings, control flow, variables, functions). Wiring and layout are always generated - never author node ids, positions, or edges.

1. `get_recipe_guide` first, every session. It returns the canonical format guide plus worked examples, versioned with the server, so it cannot drift from behavior. Do not author from memory or from this skill.
2. Discover what the recipe will reference: `get_block` for real setting schemas and output names (guide examples are illustrations, not schemas); `list_functions` + `get_function` for library Functions (a Function Block of a library Function requires an exact `version`); `list_environments`, `list_profiles`, `list_variables` for what exists; `list_tests` to avoid name collisions; `list_testdata(test_ref)` for the files an existing test's `testdata://` settings can name. Call `list_blocks` only to find out what exists: it returns the entire catalog in one large payload, so skip it when you already know the block id.
3. `validate_recipe` until it returns `valid: true`. Findings come back as data rows `{step, code, severity, problem, fix}` with recipe paths like `flow[1].if.then[0]`; fix and re-validate. Validation persists nothing.
4. Persist with `create_test_from_recipe` (the default: compiles and saves in one shot) or `build_test_from_recipe` (holds the shared workbench session and reveals the build block by block for a person watching the workbench; offer it only when someone wants to watch, and a takeover abandons the build). Both refuse duplicate names with a `NAME_CONFLICT` finding: amend the existing test instead of minting a variant name. Invalid recipes are never persisted. Create and amend responses also carry `warnings` (compile warnings only; platform warnings, such as a Function input no block uses, are read with `get_test`) and `repairs` (anything the save pipeline deleted or normalized on its own); a non-empty `repairs` means the saved test differs from your recipe, so re-read it.

## Amending a test

`get_test_recipe` -> edit the returned YAML -> `validate_recipe(recipe, test_ref)` -> `amend_test_from_recipe`. The amendment applies the whole recipe, metadata included (name, description, tags), bumps the minor version (1.0.0 -> 1.1.0), and untouched blocks keep their canvas positions.

- The expressibility verdict is completeness-checked: every block reachable in the test's graph is either expressed in the recipe or named in the `inexpressible` list, and amendment is refused (`TARGET_NOT_RECIPE_EXPRESSIBLE`) whenever `expressible` is false. Treat a refused test as workbench-only and say so.
- Readbacks never contain secret values. Re-provide every secret write (inline, `value_from_file`, or declare-empty) in the amended recipe.
- `expressible: false` with an `inexpressible` list means hand-built wiring the format cannot express; amendment is refused (`TARGET_NOT_RECIPE_EXPRESSIBLE`) and that test is edited in the workbench by a person.
- The amend response is only a version bump; confirm the structure with `get_test` (block inventory) or `get_test_recipe`.
- A readback carries `test.version`, and amendment is refused (`STALE_RECIPE_VERSION`) when the test has moved on since you read it. Keep that line intact, and on refusal read the test back again rather than re-sending. Amend once per readback.

## Test data

Each test owns a folder of data files that its blocks read through `testdata://<path>` settings (a fixture CSV, a workbook, a chart archive). The folder travels with the test (clone, export, import, backup) and reaches every run of that test as its single `testdata://` root.

- `list_testdata(test_ref)` - the test's files as a flat list of entries (`path`, `kind` file or dir, `size_bytes`, `modified_at`); an empty list when the test has none yet.
- `write_testdata(test_ref, path, content | content_base64, replace=false)` - stores one file at a relative path; its folder must already exist. Exactly one of `content` (UTF-8 text) or `content_base64` (any bytes) is required. An existing file is refused with a `File already exists` finding unless `replace` is true.
- `create_testdata_folder(test_ref, path)` - one folder under an existing parent.
- `rename_testdata(test_ref, path, new_path)` - rename or move within the test.
- `delete_testdata(test_ref, path)` - removes a file, or a folder with everything inside it.

The four write tools answer `{written | created | renamed | deleted: bool, findings}`: a refusal (`File already exists`, `Folder not found`, `Test data entry not found`, the running-test refusal below) is a finding, not an error; only an unknown `test_ref` or a bad argument raises. Blocks never change test data: a block that writes, moves, or deletes a `testdata://` path fails at run time, and run outputs go to `artifact://`. While any execution of the test is in progress (a tracked run, or a person running it in the workbench) every write tool answers the finding `Test '<name>' is running; test data cannot be changed until it finishes`: wait for a terminal status, then retry.

`validate_recipe(recipe, test_ref)` and `amend_test_from_recipe` warn `TESTDATA_NOT_FOUND` for a `testdata://` literal that names no file in that test's folder. A new test has no folder yet, so `create_test_from_recipe` and `validate_recipe` without a `test_ref` warn for every literal: create the test, `write_testdata` the files, then validate again with the `test_ref` to confirm every reference resolves.

## Running

`start_execution(entity_type, ref, environment, profile?, variables?)`, then poll `get_execution` until the status is terminal. Runs are real, tracked, attributed executions; a test run may also mirror live on the workbench for watchers (the `mirrored` flag in the response - nothing for you to manage; suite and regression runs are never mirrored).

`variables` sets per-run values for that one run without touching the stored test. Pass either `{name: value}` or `{name: {type, value}}`; each name must already be declared by the test, and the declaration's type and secret flag are preserved. `{name: null}` sets null for that run. A map is the typed form only when its `type` is the declared type; any other map is the value, so a `map` variable takes a bare map as its value.

Poll once immediately (a smoke test finishes in well under a second), then back off to a few-second cadence. There is no wait-for-terminal option. Execution rows report `duration` in whole seconds (0 means sub-second, not "did nothing") and attribute runs as `triggered_by: mcp:<client-name>`.

## Diagnosing failures

Cheapest-first triage order:

1. `list_executions(status="failed")` - newest first; also filters by `entity_type` and `limit` (default 20).
2. `get_failure_report(execution_id)` - THE triage tool: overall outcome, each failed test's failing blocks with their error messages, and the warning-and-above log excerpt around them. Refuses non-terminal executions.
3. `get_run_report(execution_id, format)` - the full evidence surface (variables, per-block timeline, complete logs). Large payload, generated on demand by a report job (the first fetch after a run can take a moment); fetch it when you need proof, not for a pass/fail check.
4. `get_execution` - status, progress, per-test rows, counts. Its `workflow_snapshot` is raw canvas JSON full of layout noise; to understand a test's logic, read `get_test_recipe` instead.

## Traps that will bite you

These are verified current behaviors, not theory:

- **A bare `assert` does not fail a run.** A failing `assert: {that: [...]}` logs a warning, paints the block `warning`, and the run still ends `passed`. To gate a run, add `fail_current_scope: true`: without `on_pass`/`on_fail` a failure then stops the flow like any failed block (a `try` catches it); with them the flow continues down `on_fail` and the function or test ends failed. Or use `on_fail` with a `halt`. Author every gating assertion that way and say so when you hand a test over.
- **`start_execution(variables=...)` overrides only what the test already declares.** Each name must be a variable of the test; an unknown name is refused rather than injected. The declaration keeps its type and its secrecy, so overriding a secret stays masked.
- **Suite and regression run reports carry per-test rows, not per-block rows.** That is by design: for block-level evidence diagnose the failed test's own run, or use `get_failure_report`, which reports failing blocks for suite runs too.
- **Masking hides exact values, not derived ones.** A slice, concatenation, or transformation of a secret is not recognized as secret and reaches logs and reports in cleartext, unless it lands in a block setting the block declares secret, a Function input declared secret, or a variable declared `secret: true`, which mask the whole value. Never build a value out of a secret and then log it.
- **A map or list element keeps its own type, and quoted digits stay text.** In a recipe, `"02134"` inside a map is a string and `16` an integer; declare an element's type with `element_types` beside the value (a variable declaration, a `set:` step, or a block step for a `typed_values` setting such as the SQL block's `driver_options`). A declared `integer` or `float` may be written as exact text to keep every digit. A `null` item or map value is allowed under any element type. Readbacks write `element_types` in full, so keep it when you amend.
- **A variable with no value holds null, and a block setting given null fails the run.** A variable declared with a type alone, or created by a capture, holds null until something writes it. A block setting that reads it first fails the run and names the setting (`'session' is null`); a For `count` and a For Each `collection` instead run zero times. Give a start value to anything a setting or an expression condition reads before its capture: a Wait Until whose expression condition reads a counter that a later capture fills declares it `{type: integer, value: 0}`. A typed check (`{variable, operator}`) needs none: it reads null by the holds-nothing rule (`is_null`, `not_equals`, `does_not_contain`, `does_not_have_key`, `is_empty` and `length_equals 0` are true, every other operator is false).
- **A free-entry setting left out takes its default when the Block runs.** A readback lists only the settings the Block holds, so a setting missing from it is running on its default.
- **A choice-list or true / false setting left out is written with its default when the Block is made.** A readback lists it like any setting the Block holds; a Function step's choice-list and true / false inputs are written from the Function's defaults the same way.
- **A null map value or list row reaches the Block as null.** Each Block's setting description says what it does with one: use it, skip it, or fail naming the setting and the entry.
- **An enum Function input is declared by its value type, not as an array.** Give it `type` `string`, `integer` or `float`, list its allowed values in `enumValues` (each once, never an empty text), and add a `default` that is one of them; it is always `required: true`. A `required` input may carry a default too, but only a run of the Function on its own uses it: a Function Block always gives the value.
- **A run is refused while another tracked run is active,** including another client's. Wait for the running execution to reach a terminal status and start yours again; there is no stop tool on this surface.

The format is strict on purpose, so these are caught before anything persists: an undefined variable inside a `{{ template }}` or an expression, a literal that contradicts its declared type (`null` fits every type), a capture target that is not a valid variable name, a capture of an output the block does not have (a Shell Script or Python script variable must first be listed in the block's `script_variables`), a key the format does not define (a typo'd `captur:`), and a profile field the bound profile does not have.

## Exact vocabularies (do not guess these)

| Parameter | Accepted values |
|-----------|-----------------|
| any `ref` / `*_ref` / `environment` / `profile` | entity id or unique name; id is tried first, and an ambiguous name errors asking for the id |
| `start_execution` entity_type | `test`, `suite`, `regression` |
| execution status (`get_execution`, `list_executions` filter) | `pending`, `running`, `passed`, `failed`, `stopped`; terminal = passed / failed / stopped |
| `reveal_secret` entity_type | `system`, `global`, `suite`, `regression`, `test`, `profile`, `function` (a library Function's own secret variable or output default, latest version, under the test-variables reveal) |
| `list_variables` / `update_variables` scope | `system`, `global` (global aliases the shared store) |
| `get_run_report` format | `json` (structured, default), `html` (rendered document for a person) |
| per-block status in run reports | `passed`, `failed`, `warning`, `pending` |
| file references in block settings | `testdata://<path>` (the test's own data files, read-only to blocks), `artifact://<path>` (the run's outputs), or a plain path (absolute or relative) where the block runs; inside a Remote Execution body WAVE reads either scheme and sends the file along for the Python Script File, the Netcat payload File and the SQL File and Load table Table; every other remotable block fails at run time on either scheme, so give it a path on the remote host |
| `write_testdata` content | exactly one of `content` (UTF-8 text) or `content_base64` (base64 of the bytes) |
| test data `path` / `new_path` | relative to the test's folder, `/`-separated, no leading slash, no `..` |
| `sql-client` `operation` | `run_sql`, `call_procedure`, `load_table`; `database_type` is `postgresql`, `mysql`, `oracle`, `mssql` or `sqlite` |
| `opensearch` `mode` | `search`, `aggregate`; `query_language` is `dql` or `query_dsl`; `order` (`search`) is `desc` or `asc`; `group_by_time` (`aggregate`) is `none`, `10s`, `30s`, `1m`, `5m`, `15m` or `1h` |
| `sftp` `operation` | `upload`, `download`, `list`, `stats`, `delete`, `move`, `create_folder`; `if_exists` (`upload`, `download`, `move`) is `overwrite` or `fail` |
| `sql-client` settings | connection: `host`, `port`, `database`, `username`, `password` (network types) or `file` (SQLite), then `tls`, `verify_ssl`, `ca_cert`, `client_cert`, `client_key`, `driver_options`, `timeout`, `connect_timeout`; `run_sql`: `script_type`, `sql` (`sql: {expr: ...}` gives the script text from an expression; `:name` parameters still bind) or `sql_file`, `parameters`, `stop_on_error`, `max_rows`; `call_procedure`: `procedure`, `arguments`, `out_arguments`, `max_rows`; `load_table`: `table`, `target_table`, `mode` (`add`, `add_or_replace`, `overwrite`), `key_columns`, `commit_each_batch`, `stop_on_error` (only with `commit_each_batch`), `batch_size` |
| `opensearch` `stats` entry | a row `{value: {stat: ..., field: ..., percent: ..., json: ..., name: ...}}`; `stat` is `count`, `sum`, `average`, `min`, `max`, `percentile`, `unique_count`, `increase` or `custom`; `field` for every stat but `count` and `custom`, `percent` (0 to 100) for `percentile`, `json` (one aggregation) and an optional column `name` for `custom` |
| `database` profile fields (bind to `sql-client`) | `host`, `port`, `database`, `username`, `password`, `verify_ssl`, `ca_cert`, `client_cert`, `client_key`, `driver_options` |
| `opensearch` profile fields (bind to `opensearch`) | `url`, `username`, `password`, `api_key`, `bearer_token`, `verify_ssl`, `ca_cert`, `client_cert`, `client_key` |

Listing tools return one JSON object per row, not a wrapping array: a single object means exactly one row exists, and zero matches return no output at all (emptiness, not an error). `list_variables` rows carry the variable's `name` alongside its declaration. The one exception is `list_blocks`, which returns the whole catalog in one `{blocks: [...]}` object. `search` parameters are case-insensitive substring matches over name and description.

## Organizing and shared variables

`create_suite` / `update_suite`, `create_regression` / `update_regression`, `create_profile` / `update_profile`, `update_variables`. All updates are non-destructive: omitted fields and members stay untouched, removal happens only through explicit `remove_*` parameters, and attached tests or suites are existence-checked. Validation problems come back as findings, exactly like recipes.

Shapes worth knowing: suite and regression `set_variables` take typed objects (`{type, value, secret}`), the same form their listings return; `value` may be null, and a type alone holds null; an `array` or `map` may add `elementTypes` (one type per item, or key -> type) to type its elements. Profile credential values live under `environment_values` keyed by environment id, each field a typed object (`{type, secret, value}`, plus `elementTypes` for an `array` or `map`). `profile_type` is validated against a fixed registry, and an invalid type comes back as a finding that lists the valid types; that finding, or the types in use via `list_profiles`, is the only oracle. There is no `get_profile`: verify profile writes through `list_profiles`.

## Secrets

Three write modes wherever a secret is authored:

- provided inline (`value`) - sealed at rest, never echoed back, but the value passes through the conversation on its way in. Only for a value whose exposure nobody minds;
- `value_from_file` - the file is read on the WAVE **server**, from its secrets input directory. This is for deployments where secrets are already provisioned onto the host (a mounted secret, a vault agent, config management). It cannot reach anything on the user's machine, and the directory is off by default, so an unconfigured deployment refuses the write with `SECRETS_DIR_NOT_CONFIGURED`. Never invent file names hoping one exists;
- declare-empty - declares the secret with no value; the write succeeds everywhere and the authoring result announces what a person still owes (a secret a capture fills is not owed). Every authoring surface returns `action_items`: tests, suites, regressions and shared variables use code `NO_VALUE_VARIABLE`, profiles use `NO_VALUE_SECRET`, and a failed run's `get_failure_report` lists unfilled secrets under `pending_action_items`. A secret that an import or a backup restore could not open, or that was imported from a masked export (`********`), holds null and is listed the same way wherever it sits: a test's variables or block settings, a suite's or regression's variables, a System variable the run reads, a profile it binds, a library Function a Function Block runs. Relay action items to the user every time.

**When the user must not paste a secret to you** - a private key, a production credential - the answer is declare-empty, not `value_from_file`. Declare it, tell them which secret you left unfilled, and let them enter it in the web UI, where the field is sealed and write-only. Reach for `value_from_file` only when the user tells you the value already sits in a file on the WAVE server. Never ask a user to upload a secret to the server just so you can reference it: that is more exposure, not less.

Reads are masked everywhere: listings, readbacks, execution variables, logs, reports. `reveal_secret` works only when the deployment's reveal flag for that surface is already on AND you pass `consent: true`; obtain the user's explicit go-ahead in the conversation first, every time. When reveal is off, say so and present the masked form; the MCP never widens reveal policy.

Authoring an expression that reads a secret returns a `SECRET_DERIVED_VALUE` warning, because masking covers the secret's exact bytes and not values built from them. There is no warning when the expression fills a block setting the block declares secret, a Function input declared secret, or a variable declared `secret: true`: those mask whatever they receive. Elsewhere, mark the derived variable `secret: true` or keep it out of anything that gets logged.

## What this surface will not do

- Delete anything but a test's data files (`delete_testdata`). Removing tests, suites, regressions, profiles, and variables is a human act in the UI.
- Publish Functions. Promoting a local function to the shared library is human-only.
- Bypass validation. A definition the UI would reject is rejected here with the same errors.

## Common mistakes

| Temptation | Reality |
|------------|---------|
| Author a recipe from memory | `get_recipe_guide` first, every session; it is versioned with the server you are talking to |
| Copy block ids, setting keys, or output names from guide examples | Examples illustrate the format; `get_block` is the schema authority and `validate_recipe` the gate |
| `NAME_CONFLICT`, so create "My Test 2" | Amend the existing test through the readback loop |
| Diagnose from `get_execution`'s workflow_snapshot | `get_failure_report` first; logic from `get_test_recipe` |
| `get_run_report` for a quick pass/fail | `get_execution`'s status and counts already answer that |
| `build_test_from_recipe` because it seems richer | It borrows the human's live workbench session; default to `create_test_from_recipe` |
| Amend without restating secrets | Readback recipes are masked; re-provide each secret inline or via `value_from_file` |
| `reveal_secret` with `consent: true` preemptively | Consent belongs to the user, given in conversation, per retrieval |
| Point a block's write, move, or delete at `testdata://` | Test data is read-only to blocks and the block fails at run time; write run outputs to `artifact://`, and change test data only through `write_testdata`, `rename_testdata`, `delete_testdata` |
| Change test data while the test runs | Every write tool answers the running-test finding until the run reaches a terminal status; wait, then retry |
