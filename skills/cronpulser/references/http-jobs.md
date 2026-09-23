# HTTP jobs: endpoint design and job settings

An HTTP job makes CronPulser send a request to the customer's endpoint on
schedule. The customer's code does the work; CronPulser records status,
duration, and optionally the response body, then evaluates alert rules.

## Request CronPulser sends

- Method: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, or `HEAD` as configured.
- Header `X-CronPulser-Secret: <job secret>` on every request. The secret is generated per job and shown on the job detail page.
- Custom headers from the job's Headers JSON. Recognized sensitive headers such as `Authorization` and `X-API-Key` are encrypted at rest.
- Body from the job's Payload JSON, sent as `application/json` for methods with a body.
- 30-second timeout including reading the response.
- Destination must be a public hostname resolving to a public IP. Private, loopback, and link-local addresses are refused. Only same-origin redirects are followed.
- Optional retry: when enabled, a timeout, transport error, or 5xx response queues one more attempt after at least five seconds. Off by default. Enable only if the endpoint is idempotent.

## Endpoint rules

1. Verify `X-CronPulser-Secret` with a constant-time comparison before doing anything. Return 401 on mismatch.
2. Return 2xx only when the work is done or safely enqueued. Return 5xx on failure so the run is marked failed and alerts can fire. Do not return 200 with an error in the body.
3. Finish within 30 seconds. For longer work, enqueue and return 202; monitor the background job separately, or use a Runner.
4. Make it idempotent: a key derived from the schedule slot, a database lock, or "skip if already done in this window". CronPulser can legitimately call twice when retry is on.
5. Keep the response body short and non-sensitive. It may be stored in execution history, and only the first 4 KB is read, so a JSON summary must fit inside that.
6. Store the secret in configuration such as `CRONPULSER_JOB_SECRET`, never in code or the URL. One secret per job.

## Verification snippets

Node.js and Express:

```js
import { timingSafeEqual } from 'node:crypto'

const expected = process.env.CRONPULSER_JOB_SECRET
if (!expected) throw new Error('CRONPULSER_JOB_SECRET is not configured')

function verifyCronPulser(req, res, next) {
  const received = req.get('X-CronPulser-Secret') || ''
  const a = Buffer.from(received), b = Buffer.from(expected)
  if (a.length !== b.length || !timingSafeEqual(a, b)) return res.status(401).end()
  next()
}

app.post('/internal/jobs/nightly-report', verifyCronPulser, async (req, res) => {
  const result = await runNightlyReport()          // idempotent inside
  res.status(200).json({ ok: true, rows: result.rows })
})
```

Python and FastAPI:

```python
import hmac, os
from fastapi import FastAPI, Header, HTTPException

EXPECTED = os.environ["CRONPULSER_JOB_SECRET"]
app = FastAPI()

@app.post("/internal/jobs/nightly-report")
async def nightly_report(x_cronpulser_secret: str = Header(default="")):
    if not hmac.compare_digest(x_cronpulser_secret, EXPECTED):
        raise HTTPException(status_code=401)
    result = await run_nightly_report()   # idempotent inside
    return {"ok": True, "rows": result.rows}
```

Complete examples for ASP.NET Core, Go, and Laravel are in
`https://cronpulser.com/guides/authenticate-cronpulser-requests/`.

## Job settings in the app

| Field | Guidance |
|---|---|
| Method | `POST` for work, `GET` for health checks. `HEAD` for reachability only. |
| URL | Full `https://` URL. Path can carry a job name. Never put secrets in the query string. |
| Headers (JSON) | `{"Authorization": "Bearer …"}` when the endpoint also needs app auth. |
| Payload (JSON) | Small, static parameters such as `{"report": "daily"}`. |
| Retry | Off unless the endpoint is idempotent. |
| Response Logging | `on_failure` by default. `always` when the body is the audit trail you want to read in history. `never` for sensitive responses. Alert rules see the body in every mode. |

## Testing

1. Save the job, open its detail page, copy the Secret Key into the endpoint's configuration.
2. **Run Now**. Open the execution: status, duration, response status, and body.
3. Add alerts: Execution Failure, Status Code `gte 400`, and No Execution with a grace of 10 minutes.

## Common outcomes

| History shows | Meaning |
|---|---|
| 401 | Secret mismatch or header not read. Compare with the job page. |
| 404 | Wrong URL or route not deployed. |
| 5xx | Endpoint threw. Body may contain the error if logging is on. |
| timeout | Work exceeded 30 s. Move to enqueue-and-202 or a Runner. |
| connection error | DNS, TLS, or firewall. Confirm the host is public and the certificate valid. |
