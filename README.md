# SalesOrderSync_REST_to_IDoc — SAP Cloud Integration Portfolio Project

## The Big Picture — Why This Project Exists

Enterprise IT constantly has to solve the same problem: getting two systems that speak
different "languages" to talk to each other.

- Modern web apps, e-commerce platforms, and CRMs speak **JSON over REST** — the
  common language of cloud-native software.
- SAP's backend systems (ECC, S/4HANA) speak **IDoc XML** — SAP's decades-old
  structured format for business documents like sales orders, invoices, and deliveries.

Nobody rewrites SAP's core to accept JSON directly. Instead, companies put an
**integration platform** in the middle — in this case, **SAP Cloud Integration (CI)**
on SAP BTP — to receive the JSON, transform it into the structure SAP expects, and
(in a real production landscape) forward it into SAP over an IDoc adapter or RFC
connection.

This project is a small, self-contained proof of that pattern: a deployed, working
iFlow that receives a JSON sales order and transforms it into IDoc-style XML — plus
a documented, end-to-end test of that flow running live on SAP BTP.

**Why it matters for my profile:** as an SAP ABAP / techno-functional developer, this
demonstrates a skill set beyond custom ABAP coding — **integration engineering** on
SAP's cloud-native middleware, which is increasingly where SAP-to-everything-else
connectivity is being built. It also demonstrates something arguably more valuable
than the build itself: the ability to **deploy, secure, and debug** an integration
like a real system, not just get a tutorial demo working once.

---

## Architecture

```
Sender (HTTPS) → Start → JSON to XML Converter → Extract Order Fields (Content Modifier) → Build_IDoc_XML (Content Modifier) → End
```

| Step | Purpose |
|---|---|
| **HTTPS Sender** | Receives the inbound JSON payload via POST |
| **JSON to XML Converter** | Converts the raw JSON body into an XML structure so downstream steps can use XPath |
| **Extract Order Fields** (Content Modifier) | Reads `orderId`, `customerId`, and `orderValue` from the XML body using XPath expressions and stores them as Exchange Properties |
| **Build_IDoc_XML** (Content Modifier) | Constructs the final IDoc-style XML using the extracted properties |

![Content Modifier extracting order fields via XPath into Exchange Properties](images/04-content-modifier-exchange-properties.png)
*The Content Modifier step reading `orderId`, `customerId`, and `orderValue` out of the converted XML body via XPath, and storing them as Exchange Properties for the next step to build the IDoc XML from.*

---

## Design Decisions & Trade-offs

- **JSON to XML Converter over Groovy scripting:** initially built with a Groovy
  script step to parse the JSON body manually, but pivoted to SAP's built-in
  JSON to XML Converter step combined with XPath-based Content Modifiers. This
  keeps the flow entirely low-code/no-script — more maintainable and easier for
  other developers to read.
- **XPath over inline JSON expressions:** SAP CI's Content Modifier uses Camel
  Simple expression syntax, which does not support a `${json.x}` style accessor
  directly on a JSON body. Converting to XML first and using XPath is the
  standard, documented pattern for this kind of extraction.
- **Content Modifier for transformation, not scripting:** given the fields
  involved were simple 1:1 mappings, a script step would have been unnecessary
  complexity. Content Modifiers keep the transformation logic visible directly
  in the flow designer.

## Known Trial-Tenant Limitations

Built and tested on a free SAP BTP trial tenant, which has some constraints:

- Only a limited number of iFlows can be deployed concurrently.
- No live backend S/4HANA system is connected on the trial tier, so the actual
  IDoc is not posted into a real SAP system — the flow demonstrates the
  *transformation and orchestration logic*, stopping at generating the IDoc XML.
- In a production scenario, the final step would route to an **IDoc adapter**
  or **RFC destination** instead of ending at a simple End event.

---

## Setting Up API Access — Step by Step

To call a deployed iFlow's HTTPS endpoint securely (rather than just viewing it
in the designer), you need OAuth2 client credentials from BTP. Here's the exact
path I followed.

### Step 1 — Confirm the trial subaccount and existing instances

Under **BTP Cockpit → Instances and Subscriptions**, I could see the ABAP
environment and Destination service instances already in place, but nothing yet
for calling Cloud Integration's runtime API directly.

![BTP Cockpit Instances and Subscriptions overview](images/01-btp-instances-subscriptions.png)
*Starting point — existing instances in the subaccount, before creating dedicated API access for Cloud Integration.*

### Step 2 — Create a Process Integration Runtime instance

From **Service Marketplace → SAP Process Integration Runtime**, I created a new
instance using the **`integration-flow`** plan — this is the plan that exposes
API access for calling and managing deployed iFlows (as opposed to other plans
that expose different Cloud Integration APIs), scoped to the **Cloud Foundry /
dev** space.

![Creating the SAP Process Integration Runtime instance with the integration-flow plan](images/02-create-pi-runtime-instance.png)
*Instance creation form — Service: SAP Process Integration Runtime, Plan: integration-flow, Runtime Environment: Cloud Foundry, Space: dev.*

### Step 3 — Instance created

The instance (`cpi-api-access`) was created successfully under Cloud Foundry,
with the technical service name `it-rt`.

![The cpi-api-access instance shown as Created, with Bound Applications and Service Keys tabs](images/03-pi-runtime-instance-created.png)
*Confirmed instance with Status: Created — next step is generating a service key from here to get OAuth2 credentials.*

### Step 4 — Generate a service key

From the instance's **Service Keys** tab, I created a new key (`cpi-api-key`).
This returns a JSON object containing:

```json
{
  "oauth": {
    "clientid": "sb-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx!bxxxxxx|it-rt-xxxxxxxxtrial!bxxxxx",
    "clientsecret": "<redacted>",
    "tokenurl": "https://<subdomain>.authentication.us10.hana.ondemand.com/oauth/token",
    "url": "https://<subdomain>.it-cpitrial05-rt.cfapps.us10-001.hana.ondemand.com"
  }
}
```

> **Security note:** the `clientid`/`clientsecret` pair here are live credentials
> for the trial tenant. They are never committed to this repo or shown in
> screenshots — only the field *shape* is documented above. Keys used during
> testing were rotated (deleted and regenerated) after this write-up was completed.

### Step 5 — Find the real deployed endpoint

Rather than assuming the endpoint path from the design-time adapter config, I
confirmed the actual resolved URL from **Monitor → Manage Integration Content**,
under the iFlow's **Endpoints** tab.

![Manage Integration Content showing the deployed iFlow's resolved endpoint URL](images/05-deployed-iflow-endpoint.png)
*`SalesOrderSync_REST_to_IDoc` — Status: Started. Resolved endpoint: `https://<subdomain>.it-cpitrial05-rt.cfapps.us10-001.hana.ondemand.com/http/salesordersync2026`*

This caught a real mismatch: the endpoint path configured on the adapter
(`/salesordersync2026`) resolved differently than the placeholder path used
in early testing (`/http/salesorder`) — see the Troubleshooting Journey below.

---

## Testing the Deployed Flow

### 1. Get an OAuth2 access token

```bash
curl -X POST "https://<subdomain>.authentication.us10.hana.ondemand.com/oauth/token" \
  -d "grant_type=client_credentials" \
  -d 'client_id=<clientid>' \
  -d 'client_secret=<clientsecret>'
```

Returns a bearer token:

```json
{ "access_token": "eyJhbGciOi...", "token_type": "bearer", "expires_in": 43199 }
```

### 2. Call the deployed iFlow

```bash
curl -X POST "https://<subdomain>.it-cpitrial05-rt.cfapps.us10-001.hana.ondemand.com/http/salesordersync2026" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <access_token>" \
  -d '{
    "orderId": "SO1001",
    "customerId": "CUST500",
    "orderValue": "2500"
  }'
```

### 3. Result — successful transformation

```xml
<IDOC>
  <E1EDK01>
    <BELNR>SO1001</BELNR>
    <KUNNR>CUST500</KUNNR>
    <NETWR>2500</NETWR>
  </E1EDK01>
</IDOC>
```

Confirmed end to end: OAuth token → HTTPS POST with JSON → JSON-to-XML
conversion → XPath extraction → IDoc XML construction → response returned.

---

## Troubleshooting Journey (worth knowing for interviews)

Getting this flow to a fully working, externally-callable state involved two
rounds of systematic debugging.

**Round 1 — Getting the flow to deploy (`Started` status):**

1. Started with a generic `CAMEL_CONTEXT_NOT_STARTED` deployment error with no
   detailed stack trace available.
2. Used **bisection**: temporarily stripped the flow down to just
   `Start → End`, redeployed, and confirmed the sender adapter and tenant were
   healthy.
3. Reintroduced steps one at a time, isolating the failure to the Content
   Modifier's expression syntax.
4. Identified that `${json.x}` is not valid Camel Simple syntax in SAP CI, and
   replaced it with a JSON-to-XML conversion + XPath approach.
5. Used the **Problems tab** to catch remaining validation issues (missing XML
   namespace, unset Data Type fields, an orphaned Receiver participant) before
   final deployment.

**Round 2 — Getting an external OAuth-authenticated call to succeed:**

1. First call returned **404 Not Found** — the endpoint path assumed from the
   README (`/http/salesorder`) didn't match the actual deployed path. Fixed by
   checking the real resolved URL under **Monitor → Manage Integration Content
   → Endpoints**, which showed `/http/salesordersync2026`.
2. Second call, now against the correct URL, returned **403 Forbidden** —
   despite a valid OAuth token and a correctly configured `ESBMessaging.send`
   User Role on the adapter.
3. Traced the cause to the **CSRF Protected** checkbox being enabled on the
   HTTPS sender adapter's Connection tab. CSRF protection expects a browser-style
   session flow (fetch a CSRF token via GET, then include it on the POST) —
   it isn't meant for direct OAuth2 machine-to-machine calls like this one.
4. Disabled CSRF Protected on the adapter (appropriate here since this is a
   REST-triggered, non-browser client scenario), redeployed, and the call
   succeeded.

![HTTPS adapter Connection tab showing the User Role and CSRF Protected setting](images/06-https-adapter-csrf-setting.png)
*The setting that caused the 403 — CSRF Protected was checked by default, which blocks direct machine-to-machine POST calls without a prior CSRF-token handshake.*

This kind of methodical isolation — narrowing a vague error down to its exact
root cause, whether at deploy time or at call time — is the same approach I'd
bring to debugging any integration issue in a production landscape.

---

## Tech Stack

- SAP Integration Suite (Cloud Integration), trial tenant
- SAP BTP — Process Integration Runtime service (OAuth2 client credentials)
- Camel Simple expressions, XPath
- HTTPS adapter, JSON to XML Converter, Content Modifier steps
- curl (bash) for endpoint testing

## Author

Sharfunisa Shajahan — SAP ABAP / techno-functional developer, based in KAEC, Saudi Arabia.
# SAP-cpi-retry-deadletter-pattern
