# 7 ways n8n workflows fail silently – and how to catch them

## 1. Catch schedule triggers that are "active" but never fire (no error is raised)

The Error Trigger only fires when an execution *fails*. If a schedule trigger stops registering, there is no execution, so nothing fails and nobody gets alerted.

A simple heartbeat check:
1. Add a last step to the scheduled workflow: an HTTP Request that pings an external heartbeat URL (e.g. a self-hosted Uptime Kuma push monitor or healthchecks.io).
2. Set the monitor's expected interval slightly longer than the schedule (15-min schedule → alert after 20 min).
3. The alert lives **outside** n8n, so it still fires if n8n itself is down.

Bonus: put the ping at the *end*, not the start. Then a run that crashes halfway is also caught.

---

## 2. Detect successful executions that processed 0 items

An execution can be marked successful while the important node received 0 items (empty API response, filter too strict, skipped branch).

Pattern: after the critical node, add an **IF** node: `{{ $input.all().length }}` equals 0 → route to a **Stop and Error** node with a clear message ("Order sync returned 0 orders, expected > 0"). Now it becomes a real failure and your Error Workflow sees it.

Only do this where 0 items is actually abnormal, otherwise you create noise.

---

## 3. Don't monitor n8n only from inside n8n

Error workflows and watchdog workflows run inside the same instance. If the main process OOMs or the container restarts in a loop, they go quiet together with everything else.

Minimum outside check: an uptime monitor on `/healthz` (and on `/healthz/readiness` if you run queue mode), plus one heartbeat per business-critical workflow (see the schedule tip). It takes 10 minutes and covers the "everything stopped" case.

---

## 4. OAuth credentials that expire quietly (Google "Testing" mode)

If a Google OAuth app is in **Testing** publishing status, refresh tokens expire after 7 days. The trigger still looks active and simply stops receiving data.

Fixes:
- Move the OAuth consent screen to **In production** (internal apps can usually do this without review).
- For credentials you can't change, add a weekly "canary" workflow that calls the API once and errors if the call fails, so you hear about it before the real workflow breaks.

---

## 5. Find executions stuck in "running" after a crash

After a worker or main-process crash, some executions can stay "running" forever, and no error is raised.

Check regularly via the public API (`GET /api/v1/executions?status=running`) and alert when an execution has been running longer than your longest normal run (e.g. 30 min). Run this check from a cron job or monitor outside the instance (see Tipp 3).

---

## 6. Your execution history might be hiding the failure you're looking for

If "Save successful executions" is off or pruning is aggressive, an empty execution list can mean "never ran" *or* "the run was deleted". For critical workflows:
- keep successful executions on (at least for those workflows),
- set pruning (`EXECUTIONS_DATA_MAX_AGE`) long enough to cover your slowest cadence (monthly workflow → keep > 31 days).

---

## 7. Turn "Workflow X failed" into an alert someone can act on

A raw "workflow failed" message gets muted fast. In your Error Workflow, build the message from `$json.execution` and `$json.workflow`:
- **What broke:** workflow name + failing node + error message
- **Impact:** which business object is affected (pass an ID like order number into the error message with Stop and Error)
- **Owner:** map workflow tags → team (e.g. tag `owner:sales-ops`)
- **Next step:** a link to the execution (`$json.execution.url`)

Send it to wherever work gets tracked (an issue in Jira/Linear/GitHub), not only to a chat channel. Group repeats so one problem becomes one item.

