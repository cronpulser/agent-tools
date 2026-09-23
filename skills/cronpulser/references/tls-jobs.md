# TLS certificate jobs

Executor **TLS Certificate**. Field: a public DNS hostname such as
`shop.example.com`. No scheme, path, or port; port 443 is implied.
International names are entered as-is and checked as punycode.

## What a check does

On each scheduled run, and on **Run Now**, CronPulser performs a TLS handshake
with SNI, 15-second timeout, and validates the chain against system roots, the
hostname, and validity dates. It records check time, expiry, issuer, and a
SHA-256 fingerprint. No HTTP request is sent; no headers or secrets are involved.
Expired or invalid certificates are recorded as failed executions with the reason.

Limits: one resolved endpoint per hostname, no redirects, no revocation or OCSP
check, public destinations only, port 443 only.

## Schedule

Any cron schedule works. Daily (`0 6 * * *`) is the common choice. A TLS job
counts as one active job like any other.

## Alerts

Creating a TLS job creates no alert. Add a **TLS Certificate** rule from the
job's Alerts section:

- `days_before_expiry`: notify when the certificate expires within this many days (default 10).
- `include_failures`: also notify when the check itself fails (handshake error, invalid chain, wrong hostname).

Select recipients explicitly; new rules preselect nobody. The rule is evaluated
after every scheduled check and re-notifies once the cooldown expires while the
condition still matches. There is no automatic recovery notification.

## Editing

The hostname can be changed. Results from the previous hostname are hidden
until the next check completes.
