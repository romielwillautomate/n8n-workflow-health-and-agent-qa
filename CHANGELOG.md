# Changelog

## v1 (October 26, 2026)

First build: an n8n reliability kit of three workflows (error alerts with repeat suppression, a heartbeat for silent and failing workflows, and a QA harness that grades an AI agent with code checks and Claude), with an end to end test that runs the n8n side against a real n8n 2.19.5 and mocks Slack, Claude, the agent and Google Sheets (68 checks across 31 scenarios).

### Error alerts

- An Error Trigger workflow that posts the workflow, the failed node, the message and a link to the failed run to Slack.
- Repeats of the same workflow, node and message (ids, times and numbers of 4 or more digits ignored) within `ALERT_REPEAT_WINDOW_MINUTES` are counted, not posted. A repeat check every 5 minutes posts one summary per finished window with the count and the latest run, and forgets the group only after Slack accepted the message.
- An alert is marked as sent only after Slack accepted it. If Slack is down, the next repeat or the repeat check posts it with the count.
- Handles the Error Trigger's trigger failure shape (no execution) with a link to the workflow when `N8N_EDITOR_URL` is set.

### Heartbeat

- Every 5 minutes, or on a Check Now webhook (`Authorization: Bearer <HEARTBEAT_SECRET>`), it reads the watch list from `WATCHLIST_JSON` and/or a Google Sheet and checks each workflow through the n8n public API: inactive while it should be active, archived, not found, no successful run within `max_silence_minutes`, success rate below `min_success_rate` over the recent runs. Manual editor runs are left out.
- New problems are posted once, resolved ones get one all clear. The open problems are saved only after Slack accepted the message.
- Optional ping to an external uptime monitor after every good check.

### Agent QA harness

- A webhook (`Authorization: Bearer <QA_RUN_SECRET>`) that takes the cases file in its body, or loads it from `QA_CASES_URL`, plus cases from a Google Sheet. Every setting and case is checked before the first message is sent.
- Each case goes to `AGENT_URL` with a request template and its own session id. Code checks the JSON, required and forbidden phrases, expected and forbidden tool calls, expected fields and text aimed at a grader. Claude (`claude-haiku-4-5` by default) grades the rubric with a forced strict tool call, the reply escaped inside tags.
- A Slack scorecard with a Ready or Not ready verdict (pass threshold, every case graded, no failed critical case), the comparison with the last saved run of the suite, and an answer with the whole result when the request asks to `wait`.
- A run is saved as the next baseline only when it is complete and its scorecard reached Slack. An incomplete run or a refused scorecard ends the run as failed.
- n8n API, agent, Claude and Google Sheets calls are retried with exponential backoff that honors Retry-After, from Code nodes, because the HTTP Request node's built in retry waits a fixed time.

### Built from what n8n 2.19.5 actually does

These were checked against the running n8n while building, before the test was written:

- **An inactive error workflow never runs.** n8n's Error Trigger page says a workflow with an Error Trigger does not have to be published, but on 2.19.5 a failed run produced no error run at all while the error workflow was off. The setup says to activate it, and the test proves the behavior.
- **Code node errors arrive cut at the last colon** (`"Customer lookup failed ... at 2026-10-26T08:00:00Z"` arrives as `"00Z [line 1]"`). The full text is taken from the first line of the stack.
- **HTTP Request errors carry a generic message** ("The service was not able to process your request"). The alert adds the description and the HTTP code, and never the request context, which can hold headers.
- **A custom secret header is saved with every run in plain text, while `Authorization` is hidden.** Both webhooks take their secret as `Authorization: Bearer`.
- **`GET /executions` answers 200 for any workflow ID, `GET /workflows/{id}` answers 404 for an unknown one.** The heartbeat asks for the runs first, so a wrong `N8N_API_URL` is reported as a URL problem instead of every workflow being reported as deleted.
- **A Respond to Webhook node in a scheduled run, or after the webhook already got its answer, just passes the data on.** The heartbeat shares its response nodes between the schedule and Check Now.
- **A Code node task may run 300 seconds.** One QA case runs per Code node run, and `AGENT_TIMEOUT_SECONDS` (60 at most) times `AGENT_MAX_TRIES` (3 at most) stays inside that.

### Found and fixed while writing and running the test

- **Every QA run was refused in the first test run.** The settings check used `new URL()`, which does not exist in the Code node sandbox, so every `AGENT_URL` was reported as invalid. The URL is now read with a pattern.
- **A 30 second silence limit was shown as "1 min"**, because durations were rounded to minutes. Durations under a minute are now shown in seconds, and longer ones are rounded down.
- **The run id could be shorter than planned** when the random part had few digits. It is now always 4 characters.
- **The test itself raced the static data.** Failures fired back to back could start two error workflow runs at the same moment, and n8n saves static data at the end of each run, so one run's count could overwrite the other's. The test now waits for each error run before the next failure, and the limit is documented in the README.

### Found and fixed in review

- **A failed uptime ping went unnoticed.** The ping used "continue on error", so a wrong `HEARTBEAT_PING_URL` would only show up as the uptime monitor reporting a dead heartbeat. The check is now saved first, then the run fails with the reason.
- **The late alert path was not tested.** An alert Slack refused whose error never repeats is posted by the repeat check; the test now covers it.
- **A silence problem resolved by removing its limit** was reported as "ran successfully again". It now says the limit was removed.
- **A repeat summary within one day** printed the date twice ("between Oct 26, 2:39 AM and Oct 26, 2:39 AM"). It now prints it once.

### Found and fixed in the second review (October 26, 2026)

- **Repeat summaries could be posted twice, or tried only once.** n8n retries a node that has an error output only when the first item of its output failed, and then runs the whole node again for every item (checked in the n8n 2.19.5 source). With two or more summaries in one repeat check, a refused first summary re-posted the ones Slack had already accepted, and a refused later summary got one try instead of three. The summaries now go through a loop (Loop: Summaries), one per Slack call. A new scenario starts a second copy of the error workflow with two finished windows in its saved state and has Slack refuse the first summary once: before the fix the other summary arrived twice, now each arrives once.
- **The flaky backoff check was the check, not the backoff.** "n8n API down: 3 tries with growing waits" failed once with gaps of 2.9 s and 2.0 s while four test suites shared the machine. The heartbeat waited 1 s and then 2 s as designed (those 503 answers carry no Retry-After); the first gap was the 1 s wait plus delay from the busy machine, and the check had an upper limit of 1.9 s. The backoff helper now returns the waits it used, the heartbeat keeps them with the failed call in the saved run, and the check reads them exactly (1000 ms, then 2000 ms) and checks the real gaps only from below, which load cannot break.
- **Alert links ignored `N8N_EDITOR_URL`.** The README says the setting is for links, but error alerts always used n8n's own execution link, which n8n builds from `WEBHOOK_URL` and which is wrong when webhooks have their own address. Alerts now use `N8N_EDITOR_URL` when it is set. The test sets it to an address different from n8n's own and checks every link.
- **Ready at a rate that only rounds up to the threshold.** The verdict compared the pass rate after rounding to one decimal, so 32 of 33 (96.97%) counted as reaching a 97% threshold. It now uses the exact rate, and the scorecard shows the exact figure when the rounded one would look like a pass. Tested with 11 of 14 against 78.6%.
- **Wrong times in a carried over count.** When an error came back after its window ended but before the repeat check, the alert said the earlier failures happened "in the 30 min before this" (or "in the 1 s before this" with a window of 0), even when they were hours old. It now gives their real times.
- **Archived workflows were claimed but not tested.** Checked by hand that n8n 2.19.5 answers 200 with `isArchived: true` for an archived workflow, and added a scenario.
- **A test check that could fail on a busy machine.** When a repeat check of the error workflow itself fails (Slack down at that moment), n8n runs the same workflow as its own error workflow, because it holds an Error Trigger. The coverage check now expects that run instead of counting it as extra.
- **Docs corrected:** the static data limitation is about runs that overlap, not runs that finish at the same moment, and while Slack is down an overlapping run can drop a queued alert; the grouping ignores numbers of 4 or more digits, not all numbers; Retry-After is followed up to 15 seconds (30 for Claude); the scorecard lists only the cases that did not pass when a run has more than 40.
