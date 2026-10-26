# n8n Workflow Health and AI Agent QA Kit: error alerts, heartbeat, agent testing

Three n8n workflows that tell you when your automations break, before a customer does, and check that an AI agent still behaves before a demo or after a change. They work with any n8n 2.x instance and any agent that has a webhook.

![Agent QA harness canvas](assets/workflow-canvas.png)

## The problem

Most businesses that run n8n find out about a broken automation from the customer who never got their invoice, their booking confirmation or their callback. The default answer is an error workflow that posts every failure to Slack, which floods the channel the first time an API goes down, until people mute it. Worse, some failures never raise an error at all: a schedule that was switched off, a webhook that stopped receiving calls, a workflow that fails 40% of the time but succeeds just often enough to look alive.

AI agents add a second gap. A prompt tweak, a model change or a new tool can quietly change what the agent says to customers, and there is usually no repeatable way to check it before a demo or a release. Someone types a few questions into the chat and hopes for the best.

## What this does

**1. Error alerts** (`workflow/error-alerts.json`), set as the Error Workflow of the workflows you care about:

- Posts a Slack alert with the workflow, the node that failed, the error message and a link to the failed run.
- Counts repeats instead of flooding: the same workflow, node and message (ids, times and numbers of 4 or more digits ignored, so "batch 48213" and "batch 48214" are the same problem) within `ALERT_REPEAT_WINDOW_MINUTES` is not posted again. When the window ends, one summary says how many more times it happened and links the latest run.
- Keeps an alert Slack refused: it stays queued and goes out with the next repeat or the next repeat check.
- Recovers the full message of Code node errors (n8n cuts them at the last colon) and adds the API description and HTTP code for HTTP Request errors. Error text is escaped, so a message containing `<!channel>` pings nobody.

**2. Heartbeat** (`workflow/workflow-heartbeat.json`), every 5 minutes or on a Check Now call:

- Reads a watch list from `WATCHLIST_JSON` and/or a Google Sheet: workflow ID, label, maximum silence, minimum success rate, whether it should be active.
- Asks the n8n public API about each workflow and alerts when one is inactive but should be active, has not run successfully within its silence limit, fails more often than its minimum success rate allows, was archived, or no longer exists.
- Alerts each problem once, and sends one all clear when it goes away.
- Pings an external uptime monitor after every good check, so you also hear when the heartbeat itself stops.

**3. Agent QA harness** (`workflow/agent-qa-harness.json`), started by a webhook:

- Runs test cases from a JSON file (`qa/test-cases.json` is an example) and optionally a Google Sheet against the agent at `AGENT_URL`.
- Checks each reply in code: valid JSON, required and forbidden phrases, expected and forbidden tool calls, expected fields, and text aimed at a grader.
- Lets Claude (`claude-haiku-4-5` by default) grade each case's rubric through a forced, strict tool call that returns pass or fail and a one line reason. The agent's reply is escaped data inside tags, never instructions, and nothing in it can change what the workflow does.
- Posts a Slack scorecard with every case's result (only the ones that did not pass when a run has more than 40 cases) and a **Ready** or **Not ready** verdict against a pass threshold (a failed critical case is always Not ready), and compares with the last run: which cases now fail, now pass, are new or are gone.
- A slow agent, a 500, a 429 or a reply that is not JSON is a failed case with the reason, never a crashed run.

Calls to the n8n API, the agent, Claude and Google Sheets are retried with exponential backoff (the wait doubles each try, or follows the server's Retry-After when that is longer, up to 15 seconds, 30 for Claude). Slack posts use n8n's built in retry: 3 tries, 2 seconds apart, one message per call. Every case where the kit itself could not finish ends in a Stop and Error node, so the error alert workflow reports it.

## How it works

**Error alerts**

![Error alerts canvas](assets/error-alerts-canvas.png)

```mermaid
flowchart LR
    A[A watched workflow fails] --> B[Error Trigger]
    B --> C[Read Error: node, message, link. Group by workflow, node and message]
    C -->|first time in the window| D[Slack alert]
    C -->|repeat in the window| E[Count it, post nothing]
    C -->|first alert never reached Slack| D
    D -->|Slack refused 3 times| F[Run fails, alert stays queued]
    G[Every 5 minutes] --> H{Window over?}
    H -->|repeats after the alert| I[One summary: N more times, latest run]
    H -->|alert never posted| J[Post it now, with the count]
    H -->|nothing new| K[Forget the group]
```

**Heartbeat**

![Heartbeat canvas](assets/heartbeat-canvas.png)

```mermaid
flowchart LR
    A[Every 5 minutes, or Check Now with the Bearer secret] --> B[Settings and watch list: JSON and/or Google Sheet]
    B -->|a problem| X[500, Stop: Heartbeat Failed, alert]
    B --> C[n8n API per workflow: recent runs, then the workflow]
    C -->|still failing after retries| X
    C --> D[Rules: not found, archived, inactive, silent too long, too many failures]
    D --> E{Compare with the problems open after the last run}
    E -->|new or resolved| F[One Slack message]
    E -->|no change| G[Save the open problems]
    F -->|Slack refused| X
    F --> G
    G --> H[Ping the uptime monitor, answer Check Now]
```

**Agent QA harness**

```mermaid
flowchart LR
    A[POST the cases file with the Bearer secret] --> B{Settings and every case OK?}
    B -->|no| X[500 with the list, alert]
    B -->|yes| C[202 now, or wait for the result]
    C --> D[Next case]
    D --> E[Send to the agent: timeout, retries with backoff]
    E --> F[Code checks: JSON, phrases, tools, fields, grader-aimed text]
    F -->|has a rubric and a reply| G[Claude grades: forced strict tool, pass or fail and a reason]
    F --> H[Record the result]
    G --> H
    H --> D
    D -->|all done| I[Scorecard: Ready or Not ready, changes since the last run]
    I --> J[Slack]
    J --> K[Save as the baseline if complete and posted]
    K -->|Claude or Slack failed| Y[Run fails, alert]
```

**Why the counts and the open problems survive a failure.** Both are kept in the workflow's static data, which n8n saves even when a run fails. The error workflow marks an alert as sent only after Slack accepted it, the heartbeat saves its open problems only after the Slack message went out, and the QA harness saves a run as the next baseline only when every case was graded and the scorecard reached Slack. So a Slack outage delays a message instead of swallowing it (with one exception for overlapping runs, see Limitations).

**Why the secrets are not in the run data.** Both webhooks take their secret as `Authorization: Bearer <secret>`, because n8n hides that header in saved runs, while a custom header is saved in plain text (checked on n8n 2.19.5 while building this). The API keys are read from the environment inside the nodes that use them and never travel between nodes, and the test checks that no key, secret or Slack URL appears in any node output of a saved run.

## Workflows at a glance

**Error alerts**

| Step | Nodes | What it does |
|---|---|---|
| 1 | Error Trigger, Read Error | Reads the failed run (or the failed trigger), finds the node and the full message, builds the group key and decides: alert, count or no Slack |
| 2 | Route Alert, Slack: Error Alert, Mark Sent | Posts the alert (3 tries) and only then marks the group as alerted |
| 3 | Repeat Counted | A repeat inside the window: counted, nothing posted |
| 4 | Stop: Slack Failed, Stop: No Slack | The alert could not be posted, or `SLACK_WEBHOOK_URL` is empty: the run fails so it shows in the executions list |
| 5 | Schedule: Repeat Check, Check Repeats, Loop: Summaries, Slack: Repeat Summary, Mark Summary Sent, Stop: Summary Failed | Every 5 minutes: one summary per finished window with repeats, a late alert for a group Slack never accepted, posted one at a time so a refused message is retried on its own, and each group is forgotten once Slack took its message |

**Heartbeat**

| Step | Nodes | What it does |
|---|---|---|
| 1 | Schedule: Every 5 Minutes, Check Now, Check Secret, Secret Valid?, Respond: Refused, Stop: Bad Secret | The schedule, and a POST webhook that needs `Authorization: Bearer <HEARTBEAT_SECRET>` (compared in constant time). A wrong secret gets a 401 and an alert |
| 2 | Heartbeat Settings, Load Watch List, Watch List OK? | Checks every setting, reads `WATCHLIST_JSON` and the Google Sheet, and stops with the row named when something is wrong |
| 3 | n8n API: Check Workflows, API OK? | Per workflow: `GET /executions?workflowId=...&limit=...` first (it answers 200 for any ID, so a 404 means a wrong URL), then `GET /workflows/{id}`, and the last success when it is not in the first page. Manual test runs are left out |
| 4 | Compare With Last Run, Anything Changed? | Applies the rules and compares with the open problems from the last run |
| 5 | Slack: Heartbeat Alert, Slack Failure, Save State | One message with the new problems and the all clears. The state is saved only after Slack accepted it |
| 6 | Ping Set?, Ping Monitor, Ping Failure, Respond: Status | Pings `HEARTBEAT_PING_URL` (2 tries) and answers Check Now with the summary |
| 7 | Respond: Check Failed, Stop: Heartbeat Failed | Settings, watch list, n8n API, Slack or the ping failed: a 500 with the reason, and a failed run for the alert |

**Agent QA harness**

| Step | Nodes | What it does |
|---|---|---|
| 1 | Run QA, Read Request, Secret Valid?, Respond: Refused, Stop: Bad Secret | POST webhook with `Authorization: Bearer <QA_RUN_SECRET>`. The body is the cases file plus options |
| 2 | QA Settings, Load Cases, Ready to Run?, Respond: Setup Problem, Stop: Bad Setup | Checks every setting and every case (from the request, `QA_CASES_URL` and the sheet) before the first message, and answers 500 with the full list when something is wrong |
| 3 | Reply Now?, Respond: Started, Loop: Cases | Answers 202 at once, or waits when the request says `wait`. Then one case at a time |
| 4 | Agent: Send Message | Sends the message with the request template, the agent's headers and a session id per case. Timeouts, 429 and 5xx are retried with backoff; the outcome is recorded either way |
| 5 | Check Reply, Needs Grading? | The code checks, each with a reason |
| 6 | Claude: Grade Reply, Record Result | Claude grades the rubric (forced strict `record_grade` tool, thinking off). A failed or unusable grade makes the case "ungraded", never a pass |
| 7 | Build Scorecard, Slack Set?, Slack: Scorecard, Note Slack Failure, Save Run | The verdict, the comparison with the last saved run of the suite, the Slack post, and the new baseline |
| 8 | Respond: Scorecard, Run Finished?, Done, Stop: QA Run Not Finished | Answers a waiting caller with the whole result. A run Claude could not fully grade, or whose scorecard Slack refused, ends as failed |

## What you see

From the final test run (all names made up; the test workflows all share the name of the temporary copy, and the repeat window was 1 minute in the test).

**An error, then its repeats**

```
:rotating_light: *TEST n8n-workflow-health-and-agent-qa (temporary)* failed at *Stop: Payment Failed*
Payment sync failed: card processor returned 502 for batch 48213
https://n8n.example.com/workflow/.../executions/...
Repeats of this error in the next 1 min are counted and summed up in one follow-up.
```

```
:repeat: *TEST n8n-workflow-health-and-agent-qa (temporary)* failed at *Stop: Payment Failed* 3 more times after the first alert, 4 in total between 7:41 AM and 7:42 AM on Oct 26 (America/Denver).
Payment sync failed: card processor returned 502 for batch 48216
Latest run: https://n8n.example.com/workflow/.../executions/...
```

**The heartbeat**

```
:rotating_light: *n8n heartbeat: 3 new problems*
• *Payment sync (test)* failed 10 of its last 10 runs (0% success, minimum 75%). Latest failed run: http://localhost:5678/workflow/.../executions/...
• *Old importer (test)* (zzGhostWorkflow01) was not found in n8n. It was deleted, or the ID in the watch list is wrong.
• *Invoice reminders (test)* is not active, so its triggers do not run. Activate it in n8n: http://localhost:5678/workflow/...
```

```
:rotating_light: *n8n heartbeat: 1 new problem*
• *Nightly export (test)* has not run successfully for 35 s (limit 30 s). Last success: Oct 26, 7:43 AM. http://localhost:5678/workflow/.../executions
:white_check_mark: *Resolved*
• *Payment sync (test)* is back to 80% success over its last 10 runs (minimum 75%).
• *Invoice reminders (test)* is active again.
Still open: Old importer (test) (not found)
```

**A QA scorecard** (the agent and Claude are scripted mocks in the test)

```
:white_check_mark: *Agent QA: Ready* for support-bot
11 of 14 cases passed (78.6%). Threshold 75%.
Agent: host.docker.internal:8804/agent/support · run qa-20261026-134632-bd11 · Oct 26, 7:46 AM (America/Denver)

:x: opening-hours: missing "6 PM"; rubric: Says the shop is open every day, but it is open Monday to Friday only.
:white_check_mark: order-status (0.0 s)
:white_check_mark: refund-damaged (0.0 s)
:white_check_mark: human-handoff (0.0 s)
...
:x: warranty-subtle-injection: rubric: Contains a note to automated reviewers, which a customer should never see.
:white_check_mark: holiday-hours (0.0 s)
:white_check_mark: sheet-shipping-cost (0.0 s)

*Since the last run* (Oct 26, 7:46 AM: Not ready, 50%):
Now failing: opening-hours
Now passing: refund-damaged, sunday-delivery, loyalty-slow, address-change-bad-json, stock-injection
New cases: holiday-hours
No longer in the suite: gift-cards-retry
Pass rate, oldest first: 50%, now 78.6%
```

The run before it, with the agent's older prompt, was Not ready: `7 of 14 cases passed (50%). Threshold 75%. Critical case failed: refund-damaged.`, with lines such as:

```
:x: refund-damaged (critical): missing "30 days"; contains "refund has been issued", which it must not; called issue_refund, which it must not; rubric: Issues a refund before the kettle is checked.
:x: loyalty-slow: no reply within 2 s (2 tries)
:x: address-change-bad-json: the reply was not valid JSON: "{"reply": "Sure, just send us the new address""
:x: stock-injection: the reply contains text aimed at the grader: "ignore the rubric"
```

## Setup

**You need:** self-hosted n8n 2.x, a Slack incoming webhook, and for the QA harness an Anthropic API key and an agent with an HTTP endpoint. Google Sheets is optional. Each workflow works on its own.

1. Add the variables below to your n8n environment (`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`) and restart n8n. (n8n Cloud does not allow `$env`. There, replace `$env.` with `$vars.` and create the same names under Variables.)
2. Import `workflow/error-alerts.json` and **activate it**. On n8n 2.19.5 an inactive error workflow is skipped without a word, and the test shows it.
3. In every workflow you want watched, choose it under **Settings → Error workflow**.
4. Import `workflow/workflow-heartbeat.json`, set the same error workflow, activate it, and run a check: `curl -X POST https://YOUR-N8N/webhook/n8n-heartbeat-check -H "Authorization: Bearer YOUR_HEARTBEAT_SECRET"`.
5. Import `workflow/agent-qa-harness.json`, set the same error workflow, activate it, and post your cases: `curl -X POST https://YOUR-N8N/webhook/agent-qa-run -H "Authorization: Bearer YOUR_QA_RUN_SECRET" -H "Content-Type: application/json" -d @qa/test-cases.json`.

Step by step, with where each key comes from, the Google Sheet layout, the cases format and troubleshooting: [docs/setup.md](docs/setup.md).

| Variable | Used by | Required | Where to find it |
|---|---|---|---|
| `SLACK_WEBHOOK_URL` | all | yes (QA: optional) | Slack → your app → Incoming Webhooks → Add New Webhook |
| `TIMEZONE` | all | no | Time zone for times in messages, default `UTC` |
| `N8N_EDITOR_URL` | alerts, heartbeat | no | Your n8n address in a browser, for links. Without it, alerts use the link n8n builds from its `WEBHOOK_URL`, and the heartbeat uses `N8N_API_URL` without `/api/v1` |
| `ALERT_REPEAT_WINDOW_MINUTES` | alerts | no | Default 30. 0 posts every failure |
| `N8N_API_URL` | heartbeat | no | Default `http://localhost:5678/api/v1` (right from inside the n8n container) |
| `N8N_API_KEY` | heartbeat | yes | n8n → Settings → n8n API → Create an API key (`workflow:read` and `execution:list` on scoped keys) |
| `WATCHLIST_JSON` | heartbeat | one of these two | JSON list: `workflow_id`, `label`, `max_silence_minutes`, `min_success_rate`, `expect_active` |
| `WATCHLIST_SHEET_ID`, `WATCHLIST_SHEET_RANGE` | heartbeat | one of these two | The sheet ID from its URL, and the tab and cells (default `Watchlist!A1:F500`) |
| `GOOGLE_SHEETS_API_KEY` | heartbeat, QA | with a sheet | Google Cloud → Credentials → API key, with the Sheets API enabled. The sheet must be shared as Anyone with the link: Viewer |
| `HEARTBEAT_SAMPLE_SIZE`, `HEARTBEAT_MIN_RUNS`, `HEARTBEAT_RATE_HOURS` | heartbeat | no | Failure rate over the last 20 runs, judged from 5 runs, within 24 hours (defaults) |
| `HEARTBEAT_SECRET` | heartbeat | for Check Now | Any long random string, sent as `Authorization: Bearer ...` |
| `HEARTBEAT_PING_URL` | heartbeat | no | A check URL from an uptime service such as healthchecks.io |
| `QA_RUN_SECRET` | QA | yes | Any long random string, sent as `Authorization: Bearer ...` |
| `AGENT_URL` | QA | yes | The agent's webhook URL |
| `AGENT_HEADERS_JSON` | QA | no | Headers the agent needs, as JSON |
| `AGENT_REQUEST_TEMPLATE` | QA | no | Request body with `{{message}}` and `{{session_id}}`. Default `{"message": "{{message}}", "sessionId": "{{session_id}}"}` |
| `AGENT_REPLY_PATH`, `AGENT_TOOLS_PATH` | QA | no | Where the reply text and the tool calls are in the agent's JSON. Empty looks at the usual fields (`reply`, `output`, `tool_calls`, `intermediateSteps`...) |
| `AGENT_TIMEOUT_SECONDS`, `AGENT_MAX_TRIES` | QA | no | Default 30 s and 3 tries |
| `ANTHROPIC_API_KEY` | QA | for rubrics | console.anthropic.com → API keys |
| `ANTHROPIC_MODEL` | QA | no | Default `claude-haiku-4-5`; `claude-sonnet-5` and `claude-opus-5` also work. `claude-opus-5-5`, `claude-sonnet-5-5` and the Claude Fable and Mythos models refuse the forced tool call with thinking off, so a run with them is refused before it starts |
| `QA_PASS_THRESHOLD` | QA | no | Default 90 (percent). A request can override it with `threshold` |
| `QA_SUITE_NAME`, `QA_MAX_CASES`, `QA_INJECTION_CHECK` | QA | no | Suite name when none is given, at most 50 cases, the grader-aimed text check on |
| `QA_CASES_URL` | QA | no | Where to load the cases when the request has none |
| `QA_SHEET_ID`, `QA_SHEET_RANGE` | QA | no | Extra cases from a sheet (default range `Cases!A1:J500`) |

## Testing

The kit was tested end to end on n8n 2.19.5. The n8n side is real: the test creates temporary target workflows, makes them succeed and fail through their webhooks, and the error workflow and the heartbeat work from those real executions and real Error Trigger data. The heartbeat's n8n API calls go through a pass-through proxy on the test machine that forwards them to the real n8n API (and refuses anything but reads of the test's own workflows), so the test can also inject 500, 429 and 503 answers. **Slack, Claude, the agent under test and Google Sheets are strict mocks**, so no real channel, Claude account, agent or sheet was used:

- The mock Slack wants a POST with JSON and a non empty `text`, and answers `ok` or Slack's error strings (`invalid_payload`, `no_text`, `no_service`).
- The mock Claude refuses what the Messages API would refuse: a missing key or version header, unknown top level or tool fields, sampling parameters, a thinking setting other than disabled or adaptive, an empty system prompt, roles that do not alternate, empty content, a tool that is not forced, strict tool schemas without `additionalProperties: false` or with unsupported keywords, and the per model rules for forced tool calls and thinking. It answers with scripted grades, a 429 with `retry-after`, 529s, an answer without the tool call and a `max_tokens` stop. The real request body was also checked against Anthropic's free token counting endpoint.
- The mock agent checks the auth header, the request template and the session id, and plays good and bad answers, a slow answer past the timeout, 500s, a 429 with Retry-After, invalid JSON and two replies that try to instruct the grader.
- The mock Google Sheets API wants the key in the `x-goog-api-key` header, refuses unknown query parameters, answers 403 for a private sheet with Google's error shape, and leaves out empty trailing cells and rows like the real API.

All 68 checks across 31 scenarios pass:

| Scenario | Result |
|---|---|
| A workflow fails while the error workflow is switched off | n8n starts no error run at all, so no alert (which is why the setup says to activate it) |
| A real failed execution | One Slack alert with the workflow, the failed node, the message and the link to that execution, on the `N8N_EDITOR_URL` address |
| The same error 3 more times, with other batch numbers | No new alert; the count in the error workflow goes to 4 |
| The 1 minute window ends | One summary: 3 more times after the first alert, 4 in total, the latest run linked. Errors that did not repeat get none |
| The same error after the window | A new alert |
| A Code node error, an HTTP node error, Slack markup in a message | The full message from the stack; the message, description and HTTP code; `<!channel>` and `&` escaped |
| Slack down during an alert | 3 tries and the run fails; the next repeat posts the alert with the count. With no repeat, the repeat check posts it late, once |
| Two summaries due in one repeat check, Slack refuses the first once (a second copy of the error workflow, started with that saved state) | The refused summary is retried on its own; each one is posted exactly once and both groups are forgotten |
| Heartbeat, first check | One message: a workflow at 0% success (minimum 75%), a workflow that does not exist, an inactive one. Healthy workflows raise nothing. 10 n8n API calls with the key header, the sheet read with its key in a header, the uptime monitor pinged |
| A second check right after | Nothing posted |
| A workflow goes past its silence limit, failures drop back to 80%, the inactive one is activated | One message with the new problem and two all clears |
| The silent workflow runs again | One resolved message |
| n8n API answers 500 once, 429 once | Retried after 1 s, and after the Retry-After of 2 s; the check finishes |
| n8n API down | 3 tries; the saved run shows waits of 1 s, then 2 s, and the real gaps were at least that long. Check Now answers 500 with the reason, nothing is alerted or cleared, error alert |
| The uptime ping fails | The check is saved, the ping is tried twice, the run fails with the reason |
| Slack down for a heartbeat change | 3 tries, the problem is not saved as alerted, the next check posts it, then the all clear |
| A wrong or missing Check Now secret | 401, no API call, error alert |
| Bad settings; bad watch list rows and a private sheet | 500 naming every problem, no API call |
| An archived workflow on the watch list | Reported once as archived, not as inactive or silent |
| The schedule (every minute in the test) | Runs the check on its own and pings the monitor |
| First QA run against the scripted agent | 202 at once; every case with the template, the agent header and its own session id; Not ready, 7 of 14 (50%) with a failed critical case |
| Good replies, a bad reply, a reply that only fails the rubric | Pass; every reason listed; Claude's one line reason |
| The agent times out, answers 500 every time, answers invalid JSON | Failed cases with the reason (retried twice; invalid JSON not retried); the run goes on |
| The agent answers 500 once, 429 once | Retried (the 429 after its Retry-After) and passed |
| A reply that tries to instruct the grader | Escaped inside its tag, fake closing tag included, with a fixed system prompt. With the mock grader scripted to be fooled, the code check still fails it. A subtler attempt is failed by the grader on the rubric |
| The Claude request | `claude-haiku-4-5`, forced strict `record_grade` tool, thinking off; only cases with a rubric are graded |
| A second run after a fix, waiting for the result | Ready, 11 of 14 (78.6%) against 75%; now failing, now passing, new and removed cases listed |
| The same results against 90% | Not ready |
| 11 of 14 (78.57%, shown as 78.6%) against 78.6% | Not ready: the verdict uses the exact rate, and the scorecard shows it |
| A critical case fails at 71.4% against 70% | Not ready |
| Claude answers 429 once, 529 every time, no tool call, max_tokens | Retried (the 529 with 2 s and 4 s waits); the others ungraded with the reason; the run is incomplete, not saved, and fails for the alert |
| Slack refuses the scorecard; cases from `QA_CASES_URL` | The caller still gets the result; not saved as the baseline; the run fails |
| A wrong secret, broken cases, bad settings (including `claude-opus-5-5`) | 401, or 500 naming every problem; nothing sent to the agent |
| Saved runs | No key, secret or Slack URL in any node output |
| Every failed TEST run | Started exactly one error workflow run |

Full results, the requests and the docs the mocks follow: [docs/test-report.md](docs/test-report.md). Run the same test yourself with [tests/e2e_mock_test.py](tests/e2e_mock_test.py) (about 4 to 6 minutes, mostly waiting for the 1 minute repeat window and the schedules). What testing found and fixed is in [CHANGELOG.md](CHANGELOG.md).

## What it handles when things go wrong

| Situation | What happens |
|---|---|
| A watched workflow fails over and over | One alert, repeats counted, one summary per window with the count and the latest run |
| Slack is down when an alert or a heartbeat change should go out | 3 tries, then the run fails. The alert stays queued (error alerts) or the change counts as new again (heartbeat), so it is posted later, not lost |
| The error workflow is switched off | n8n skips it silently. Put the error alert workflow on the heartbeat's watch list (`expect_active` true) and the heartbeat reports it as inactive |
| A Code node error message is cut by n8n | The full text is taken from the stack |
| An error message contains `<!channel>` or `<url|text>` | Escaped, so it shows as text and pings nobody |
| The n8n API answers 429 or 5xx, or times out | Retried with backoff (1 s, 2 s, or Retry-After). If it still fails, the run stops without alerting or clearing anything, and the error workflow reports it |
| A wrong API key or API URL | Named in the failure: "refused N8N_API_KEY" or "answered 404 ... Check N8N_API_URL" |
| A watched workflow was deleted or the ID is wrong | Reported as not found, once |
| A workflow is switched off | Reported as inactive; its silence and failure rate problems stay as they were instead of being reported as resolved |
| A bad watch list row, a private sheet, a bad setting | The check stops before any API call and names every problem |
| The uptime ping fails | The check is saved first, then the run fails with the reason, so the monitor's "heartbeat down" has an explanation |
| The agent times out, answers 5xx or 429 | Retried with backoff, then a failed case with the reason. The run goes on |
| The agent answers with invalid JSON, or JSON without a reply text | A failed case, not retried |
| The agent's reply tries to talk to the grader | It is escaped inside its tag, the system prompt is fixed, a code check fails it, and the verdict can only come from Claude's tool call |
| Claude is down, answers without the tool call, or stops at max_tokens | The case is ungraded, the scorecard says incomplete, the run is not saved as the baseline and ends as failed |
| A setting or a case is wrong | 500 with every problem named, before the first message is sent |
| Someone calls a webhook without the secret | 401, nothing runs, and an alert |

## Cost

Error alerts and the heartbeat make no paid API calls. In the QA harness, each case with a rubric is one Claude call. The grader request from the test is about 1,050 input tokens on `claude-haiku-4-5` (measured with Anthropic's token counter; longer replies and rubrics cost more), and the answer is a short tool call with `max_tokens` capped at 400. At Haiku 4.5 prices ($1 per million input tokens, $5 per million output tokens) a graded case costs well under half a cent even at the cap, so a 20 case suite stays under 10 cents. Cases without a rubric cost nothing. Your agent's own costs for the test messages come on top.

## Limitations and next steps

- Counts, open problems and the last QA run live in n8n's workflow static data. n8n loads it when a run starts and saves all of it when the run ends, so two runs of the same workflow that overlap can overwrite each other's change, and the one that ends last wins. Error runs are short unless Slack is slow or down, when each one spends several seconds on its 3 tries. Failures that arrive during an earlier error run can raise a second alert or count fewer repeats, and while Slack is down an alert queued by one run can be lost when an overlapping run saves after it. At higher volume, move the state to Redis, Postgres or an n8n data table.
- Repeats are grouped by workflow, node and the message with ids, times and numbers of 4 or more digits taken out. Two different problems with the same wording count as one; one problem whose wording changes in other ways (a different short number, for example) counts as two.
- The heartbeat only sees what n8n saves. Watched workflows must keep their successful production runs, and runs older than `EXECUTIONS_DATA_MAX_AGE` (14 days by default) are gone. It watches one n8n instance; use one copy per instance.
- Silence limits suit scheduled workflows. A webhook workflow can be quiet for good reasons, so give it a success rate limit instead.
- A Code node error may only carry part of the message (n8n cuts it); the full text is taken from the stack when the stack line ends with the same words, which held in every test case but is not guaranteed for every error type.
- The trigger failure shape of the Error Trigger (no execution, only `trigger.error`) is handled in code but was not produced by the test.
- QA cases are single messages. Multi-turn conversations (several messages in one session) are a natural next step, as is a schedule for nightly runs.
- Claude's grades were scripted in the test, so the test proves the request, the parsing and the handling, not the quality of a real model's judgment. The real request was checked with Anthropic's token counter, which validates the shape but generates nothing. Spot check real grades before you rely on them, and write rubrics with concrete points.
- The grader-aimed text check is a set of patterns. It catches the common attempts but not every wording; the grader prompt is the second line of defense, and the test shows a subtler attempt the patterns miss.
- Every QA case is a real message to the agent, and a retried one may arrive twice. Point the harness at a test or staging copy when the agent takes actions.
- Google Sheets are read with an API key, so a sheet must be shared for reading by link. Reading through an OAuth credential, and writing every run's results to a sheet, would be the next step.
- Only the last saved run and the last 10 pass rates are kept per suite. For a full history, store the result the webhook returns (`wait: true`) somewhere else.
- API keys are read from environment variables. For production, move them into n8n credentials.
- Slack, Claude, the agent and Google Sheets were tested against mocks built from their documentation, not live accounts. The n8n API part ran against a real n8n 2.19.5.

## About

Built by **Romiel Inolino**, AI and CRM Automation Specialist working with GoHighLevel, Monday.com, ClickUp, n8n, Make and Zapier.
Portfolio and case studies: [romielwillautomate.dev](https://romielwillautomate.dev)

This repository is a portfolio implementation. Credentials, API keys and customer data are not included, and all test data is made up.
