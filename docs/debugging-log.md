# Debugging Log

Five real issues hit while building this pattern, in the order they surfaced.
Each one was diagnosed from an actual Message Processing Log or error trace,
not guessed at.

---

## 1. Duplicate start events blocked deployment

**Symptom:** A warning triangle on the canvas; the iFlow wouldn't validate.

**Cause:** The default template's message Start Event was left in place
alongside a newly added Timer Start Event. CPI only allows one start event
per Integration Process.

**Fix:** Deleted the leftover default Start node and reconnected the Timer
directly into the process.

---

## 2. Timers set to fire once, not repeat

**Symptom:** Both flows looked "stuck" — the error store only ever held 1–2
entries no matter how long the flows ran, and nothing seemed to update
between checks.

**Cause:** Both Timer Start events had **Repeat: None** — meaning "run once
at deployment, then stop forever." Not a recurring poll at all.

**Fix:** Changed Repeat to `Minutes` with a short interval on both timers.
Once actually recurring, the store started growing and shrinking in real
time, as designed.

---

## 3. Exchange properties don't survive into the Exception Subprocess

**Symptom:** The payload written to `ErrorStore` was empty every time,
even though the original message clearly had content.

**Cause:** An Exchange Property set earlier in the main flow (meant to
preserve the original payload for the error handler) did not carry over into
the Exception Subprocess. This isn't obvious from the UI, and is only
lightly documented in SAP Community threads — properties set before an
exception aren't reliably available once the exception subprocess starts.

**Fix:** Switched from an Exchange Property to a **Message Header** instead
(headers survive the exception boundary more reliably), and as a simpler
fallback for this test scenario, hardcoded the known payload as a Constant
in the exception handler's Content Modifier.

---

## 4. The Data Store `Select` operation always expects XML — even for JSON

**Symptom:** `RetryFlow_OrderSync` failed with:
```
com.ctc.wstx.exc.WstxUnexpectedCharException: Unexpected character '{' (code 123)
in prolog; expected '<'
```

**Cause:** SAP's Data Store `Select` operation always wraps multiple
retrieved entries in an XML envelope internally — this isn't a togglable
setting anywhere in the UI. Storing raw JSON in the data store breaks this
silently, because the moment `Select` (paired with a General Splitter) tries
to process the batch, it expects everything to start with `<`, not `{`.

**Fix:** Changed the Exception Subprocess's Content Modifier to write the
payload wrapped in valid XML, with the original JSON preserved inside a
CDATA section:

```xml
<ErrorEntry>
  <Payload><![CDATA[{"orderId": "1001", "customer": "Acme", "amount": 250}]]></Payload>
</ErrorEntry>
```

Downstream, an XPath expression (`//Payload/text()`) extracts the original
JSON back out before resending.

---

## 5. One bad entry poisons the entire batch read

**Symptom:** The XML fix above was verified correct on a fresh entry, but
`RetryFlow_OrderSync` kept failing with the exact same XML parsing error.

**Cause:** The `Select` step pulls up to 10 entries per run. Old entries
written *before* the XML fix (still raw JSON) were still sitting in
`ErrorStore`. A single malformed entry in that batch was enough to fail the
entire `Select` operation — not just skip the bad one.

**Fix:** Cleared every entry from `ErrorStore` via Monitor → Manage Data
Stores, so only well-formed post-fix entries remained. The very next run
completed cleanly.

---

## Takeaway

Every one of these was invisible from the canvas — no red X, no obvious
misconfiguration. Each required reading the actual Message Processing Log,
matching the failing `ModelStepId` and `com.sap.esb.datastore.id` to what was
actually stored, and reasoning backward from the exception class name
(`WstxUnexpectedCharException` → an XML parser choking on `{` → the Select
operation must expect XML). That's the debugging muscle this project was
built to exercise.
