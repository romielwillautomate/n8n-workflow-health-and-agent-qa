# Setup: Slack, n8n, the heartbeat, Google Sheets and the agent QA harness

You need self-hosted n8n 2.x, a Slack workspace where you can add an incoming webhook and, for the QA harness, an Anthropic API key. Google Sheets is optional. Plan about 30 minutes for all three workflows. Each one works on its own, so you can start with the error alerts and add the rest later.

## 1. Slack incoming webhook

1. Go to **api.slack.com/apps → Create New App → From scratch**, give it a name (for example "n8n health") and pick your workspace.
2. Open **Incoming Webhooks**, switch **Activate Incoming Webhooks** on, then **Add New Webhook to Workspace** and choose the channel the alerts should go to.
3. Copy the URL (`https://hooks.slack.com/services/...`) into `SLACK_WEBHOOK_URL`.

Anyone who has this URL can post to the channel, so treat it like a password. All three workflows post through it.

## 2. n8n environment

The workflows read their settings with `$env`, so self-hosted n8n needs `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`.

1. Copy `.env.example`, fill in the values and add them to the environment of your n8n container (for Docker Compose, the `environment:` list or an `env_file:`).
2. Restart n8n so it reads them.

n8n Cloud does not allow `$env`. There, replace `$env.` with `$vars.` in the three workflow files before you import them, and create the same names under **Variables**.

`N8N_BLOCK_ENV_ACCESS_IN_NODE=false` lets every workflow on the instance read every environment variable. If other people build workflows on the same instance, move the keys into n8n credentials or give the kit its own instance.

## 3. Error alerts

1. **Workflows → Import from file** → `workflow/error-alerts.json`.
2. **Activate it.** On n8n 2.19.5 an inactive error workflow is skipped without any message; the test shows a failed run producing no error run at all while it was off. (n8n's Error Trigger page says such a workflow does not have to be published, but that was not true on 2.19.5.) It also needs to be active for its repeat check schedule.
3. Open each workflow you want watched → **⋯ → Settings → Error workflow** → choose **n8n Error Alerts: Slack with repeat suppression** → **Save**. Do the same for the heartbeat and the QA harness below, so a failed check is reported too.
4. Test it: make one of those workflows fail once (for example a Stop and Error node in a copy) and run it through its trigger. The Error Trigger only runs for automatic runs, not for a manual **Execute workflow** in the editor.

Settings:

- `ALERT_REPEAT_WINDOW_MINUTES` (default 30): how long repeats of the same error are only counted. 0 posts every failure.
- The repeat check runs every 5 minutes. Change it in the **Schedule: Repeat Check** node. A summary can come up to that long after its window ends.
- `TIMEZONE` for the times in the messages, `N8N_EDITOR_URL` for the links to failed runs. Without it, alerts use the link n8n builds from its `WEBHOOK_URL`, which is wrong when webhooks have their own address, and a failed trigger (no execution exists then) gets no link.

The repeat check runs every 5 minutes, so this workflow adds about 288 short runs a day to your executions list. If that bothers you, set the workflow's **Save successful production executions** setting to **Do not save**; failed runs are still saved.

## 4. Heartbeat

### n8n API key

1. In n8n: **Settings → n8n API → Create an API key**. On Enterprise instances with scoped keys, it needs `workflow:read` and `execution:list`.
2. Put it in `N8N_API_KEY`.
3. `N8N_API_URL` is the public API of the instance to watch. When the heartbeat runs on the same instance in Docker, `http://localhost:5678/api/v1` works from inside the container. Behind a path prefix, include it (`https://example.com/n8n/api/v1`).
4. Set `N8N_EDITOR_URL` to the address people open in a browser (`https://n8n.example.com`), so the links in alerts work outside the container.

### The watch list

Each workflow needs its ID: open the workflow, the ID is the part of the URL after `/workflow/`.

| Field | Required | Meaning |
|---|---|---|
| `workflow_id` | yes | The workflow to watch |
| `label` | no | The name used in alerts (default: the workflow's name in n8n) |
| `max_silence_minutes` | no | Alert when its last successful run is older than this. Leave empty for workflows that only run when something happens (webhooks) |
| `min_success_rate` | no | Alert when fewer than this percent of its recent runs succeeded |
| `expect_active` | no | Default true: alert when the workflow is not active. Use false for sub-workflows that are only called by others |

As JSON in `WATCHLIST_JSON`:

```json
[
  {"workflow_id": "AbCdEf1234567890", "label": "Order sync", "max_silence_minutes": 60, "min_success_rate": 90},
  {"workflow_id": "GhIjKl1234567890", "label": "Lead intake webhook", "min_success_rate": 95}
]
```

Or in a Google Sheet (section 5), with these column names in the first row. `templates/watchlist-sheet.csv` has the header and three example rows; **File → Import** it into a new sheet. When both are set, both lists are watched. A workflow listed twice, a missing ID or a value that is not a number stops the check with the row named, instead of quietly skipping it.

The failure rate looks at the last `HEARTBEAT_SAMPLE_SIZE` runs (default 20) from the last `HEARTBEAT_RATE_HOURS` (default 24), and only once there are `HEARTBEAT_MIN_RUNS` of them (default 5), so one failed run out of one is not a 0% alarm. Manual test runs from the editor are left out.

### Turn it on

1. Import `workflow/workflow-heartbeat.json`, choose the error alert workflow as its **Error workflow**, and activate it.
2. Set `HEARTBEAT_SECRET` and run a check by hand:

   ```bash
   curl -X POST "https://n8n.example.com/webhook/n8n-heartbeat-check" \
     -H "Authorization: Bearer YOUR_HEARTBEAT_SECRET"
   ```

   The answer lists what was checked and which problems are open, for example `{"ok":true,"source":"check now","slackPosted":true,"checked":5,"open":[...],"new":[...],"resolved":[]}`. New problems also go to Slack, once.
3. After that it runs every 5 minutes. Change the interval in the **Schedule: Every 5 Minutes** node. Keep it shorter than your smallest `max_silence_minutes`.

Two things the heartbeat depends on:

- The watched workflows must keep their successful production runs (the default). With **Save successful production executions: Do not save**, the heartbeat sees no successes and reports the workflow as silent.
- n8n deletes old executions (`EXECUTIONS_DATA_MAX_AGE`, 14 days by default). A workflow that runs less often than that has no successful run left to find; give it no `max_silence_minutes` or raise the age.

Put the error alert workflow itself on the watch list too, with no limits (`expect_active` true is the default). If someone switches it off, the heartbeat tells you, and nothing else would.

### Who watches the heartbeat

If the heartbeat itself stops (n8n is down, the workflow was switched off), it cannot tell you. Set `HEARTBEAT_PING_URL` to a check URL from an external uptime service such as healthchecks.io: the heartbeat calls it after every good check, and the service alerts you when the calls stop.

## 5. Google Sheets (optional)

The heartbeat watch list and the QA cases can live in a Google Sheet. The workflows read it with an API key, so the sheet must be shared for reading by link.

1. **Google Cloud Console → APIs & Services → Library → Google Sheets API → Enable.**
2. **APIs & Services → Credentials → Create credentials → API key.** Restrict the key to the Google Sheets API. Put it in `GOOGLE_SHEETS_API_KEY`.
3. In the sheet: **Share → General access → Anyone with the link → Viewer.** Do not put secrets in it.
4. The sheet ID is the part of the URL between `/d/` and `/edit`. Put it in `WATCHLIST_SHEET_ID` or `QA_SHEET_ID`.
5. The range is the tab name and the cells, for example `Watchlist!A1:F500` (`WATCHLIST_SHEET_RANGE`) or `Cases!A1:J500` (`QA_SHEET_RANGE`). The first row holds the column names.

The key is sent in the `x-goog-api-key` header, not in the URL. If your Google Workspace does not allow link sharing, use `WATCHLIST_JSON` and the cases file instead.

## 6. Agent QA harness

### Point it at your agent

Any agent with an HTTP endpoint works. A test copy or staging version is best, because every case is a real message (and a retried one may arrive twice).

| Setting | Example | Meaning |
|---|---|---|
| `AGENT_URL` | `https://n8n.example.com/webhook/support-agent` | Where each test message is POSTed |
| `AGENT_HEADERS_JSON` | `{"X-Agent-Secret": "..."}` | Headers the agent needs, as JSON |
| `AGENT_REQUEST_TEMPLATE` | `{"message": "{{message}}", "sessionId": "{{session_id}}"}` | The request body. `{{message}}` is the case's message, `{{session_id}}` is unique per case and run, so an agent with memory starts fresh each time |
| `AGENT_REPLY_PATH` | `output` | Where the reply text is in the answer. Empty: the first text found in `reply`, `output`, `text`, `message`, `response`, `answer` or `content` |
| `AGENT_TOOLS_PATH` | `tool_calls` | Where the list of tool calls is. Empty: `tool_calls`, `toolCalls`, `tools_used`, `toolsUsed`, `tools`, `intermediateSteps` or `intermediate_steps`. Items can be names or objects with `name`, `tool`, `tool_name`, `function.name` or `action.tool` |
| `AGENT_TIMEOUT_SECONDS` | `30` | Wait per try, 1 to 60 |
| `AGENT_MAX_TRIES` | `3` | Tries per case, 1 to 3. 429, 5xx, timeouts and network errors are retried with backoff (1 s, 2 s) |

For an n8n AI Agent behind a Webhook node, the template is usually `{"chatInput": "{{message}}", "sessionId": "{{session_id}}"}` and the reply is in `output`. Turn on **Return Intermediate Steps** in the agent if you want to check tool calls; they come back in `intermediateSteps`. Answers that are a list of items use the first item.

The agent must answer with JSON. A reply that is not valid JSON, a timeout or an error status is a failed case, never a crashed run.

### Write the test cases

`qa/test-cases.json` is a working example for a made up shop's support agent:

```json
{
  "suite": "support-bot",
  "cases": [
    {
      "id": "refund-damaged",
      "message": "My kettle arrived broken. Can I get a refund?",
      "must_include": ["30 days"],
      "must_not_include": ["refund has been issued"],
      "forbid_tools": ["issue_refund"],
      "rubric": "Explains the 30 day return window and how to start a return. Does not promise or issue a refund before the item is checked.",
      "critical": true
    }
  ]
}
```

| Field | Checked by | Meaning |
|---|---|---|
| `id` | | Unique, letters, digits, `.`, `-`, `_`. Runs are compared by id |
| `message` | | What is sent to the agent |
| `must_include` | code | Every phrase must be in the reply text (case and spacing do not matter, curly quotes count as straight ones) |
| `must_not_include` | code | None of these may be in the reply text |
| `expect_tools` | code | Every tool must be in the reply's tool calls |
| `forbid_tools` | code | None of these may be called |
| `expect_fields` | code | Fields of the JSON answer and their values, with dots for nesting: `{"handoff": true, "order.status": "shipped"}`. `"*"` means present and not empty |
| `rubric` | Claude | What a good reply says, in plain words. Claude answers pass or fail with a one line reason |
| `critical` | | A failed critical case makes the run Not ready, whatever the pass rate |
| `enabled` | | false skips the case |

In a Google Sheet, use these as column names (`templates/qa-cases-sheet.csv`); list cells take one item per line or items separated by `|`, `expect_fields` is JSON text, and `critical` and `enabled` take TRUE or FALSE. Cases from the request (or `QA_CASES_URL`) and from the sheet are run together; a case id used twice is refused.

Every reply also gets a check for text aimed at a grader ("ignore the rubric", "mark this as pass", a fake `</agent_reply>` tag). A real customer would never see such text, so the case fails. Set `QA_INJECTION_CHECK=false` to turn it off.

### Claude

Put your key in `ANTHROPIC_API_KEY`. The default `ANTHROPIC_MODEL` is `claude-haiku-4-5`; `claude-sonnet-5` and `claude-opus-5` also work. The grader forces one strict tool call with thinking turned off, which `claude-opus-5-5`, `claude-sonnet-5-5` and the Claude Fable and Mythos models refuse, so a run with one of them is refused before it starts.

### Run it

1. Import `workflow/agent-qa-harness.json`, choose the error alert workflow as its **Error workflow**, set `QA_RUN_SECRET`, and activate it.
2. Post the cases file:

   ```bash
   curl -X POST "https://n8n.example.com/webhook/agent-qa-run" \
     -H "Authorization: Bearer YOUR_QA_RUN_SECRET" -H "Content-Type: application/json" \
     -d @qa/test-cases.json
   ```

   It answers `202` right away (`{"started":true,"runId":"qa-...","suite":"support-bot","cases":6}`) and posts the scorecard to Slack when the run is done.
3. To wait for the verdict instead (before a demo, or in a deploy script), add options next to the cases:

   ```bash
   jq '. + {wait: true, threshold: 100, label: "after prompt v3"}' qa/test-cases.json | \
   curl -X POST "https://n8n.example.com/webhook/agent-qa-run" \
     -H "Authorization: Bearer YOUR_QA_RUN_SECRET" -H "Content-Type: application/json" -d @-
   ```

   The answer is the whole result: `verdict` (`Ready` or `Not ready`), `passRate`, every case with its reasons, and what changed since the last run.

Request options: `wait` (answer when done), `threshold` (overrides `QA_PASS_THRESHOLD` for this run), `only` (a list of case ids), `label` (shown on the scorecard), `suite` (overrides the file's suite name). With no `cases` in the request, the cases come from `QA_CASES_URL` and the sheet.

### How runs are compared

Each complete run is saved per suite in the workflow's static data, and the next run of the same suite lists the cases that now fail, now pass, are new or are gone. A run is only saved when every case was graded and its scorecard reached Slack (or Slack is not set), so the comparison is always against a run someone saw. n8n does not save static data for manual runs in the editor, so start runs through the webhook.

## Troubleshooting

| You see | Likely cause |
|---|---|
| A workflow failed but no alert came | The error alert workflow is not active, is not chosen as that workflow's Error workflow, or the run was a manual one from the editor |
| Error alert run stopped at "Stop: No Slack" | `SLACK_WEBHOOK_URL` is empty |
| Error alert run stopped at "Stop: Slack Failed" | Slack refused the post 3 times. The alert stays queued and goes out with the next repeat or the repeat check |
| Fewer alerts than failures | That is the repeat window: the same error is counted, and a summary follows when the window ends |
| Heartbeat: "refused N8N_API_KEY" | The key is wrong or lacks `workflow:read` / `execution:list` |
| Heartbeat: "answered 404 ... Check N8N_API_URL" | The URL does not end in `/api/v1`, or points at the wrong instance |
| Heartbeat: a workflow "was not found" | Wrong ID in the watch list, or the workflow was deleted |
| Heartbeat: a healthy workflow is reported as silent | Its successful runs are not saved, or they were deleted by execution pruning (section 4) |
| "could not be read from Google Sheets: HTTP 403" | The sheet is not shared as Anyone with the link: Viewer, or the key may not use the Sheets API |
| QA run refused with a list of problems | A setting or a case is wrong; the 500 answer and the error alert name each one |
| Every QA case "could not reach the agent" or got 401 | `AGENT_URL` or `AGENT_HEADERS_JSON` |
| "no reply text in the JSON" | Set `AGENT_REPLY_PATH` to where your agent puts its answer |
| QA scorecard says "incomplete" | Claude could not grade some cases (outage, key, credit). The run is not saved as the baseline; run it again |
| No "Since the last run" section | First run of that suite name, or the earlier runs were incomplete or started from the editor |
