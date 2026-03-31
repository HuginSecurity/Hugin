# Findings

Findings is Hugin's centralized vulnerability repository. Every security issue discovered during a test -- whether by the built-in scanner, a Synaps module, a passive check, or manual analysis -- ends up here. The Findings tab provides a unified view for triaging, annotating, exporting, and tracking vulnerabilities across an engagement.

## Creating Findings

Findings enter the repository through three paths:

**Scanner-generated** -- The built-in scanner and Synaps modules create findings automatically when a check detects a vulnerability. These include the exact probe request/response, the insertion point, the payload used, and the CWE classification.

**Send to Findings** -- Right-click any flow in the HTTP History view and choose "Send to Findings". A dialog opens with the target URL pre-populated from the flow. Fill in the title, severity, and description, then save.

**Manual creation** -- Open the Findings tab and click "New Finding" to create a finding from scratch. This is useful for issues discovered through manual testing that do not correspond to a single captured flow.

## Finding Fields

Every finding has the following fields:

- **title** -- short description of the vulnerability
- **target_url** -- the affected URL; auto-populated from the flow for scanner findings, manual input for hunter findings
- **severity** -- Critical, High, Medium, Low, or Info
- **confidence** -- Certain, Firm, or Tentative; indicates how reliable the detection is
- **status** -- current state in the triage workflow (see below)
- **source** -- Scanner, Manual, Plugin, or Passive; indicates how the finding was created
- **cwe** -- CWE identifier (e.g., CWE-89 for SQL Injection); auto-set by scanner checks
- **check_id** -- internal identifier of the check that produced the finding
- **insertion_point** -- where the payload was injected (query parameter, header, cookie, body field, URL path)
- **evidence** -- free-text field for notes, screenshots, or exploit details
- **remediation** -- recommended fix
- **references** -- links to relevant advisories, CVEs, or documentation
- **notes** -- internal notes for the tester (not included in exports by default)
- **tags** -- arbitrary labels for organization and filtering

## Evidence Flows

Findings can have multiple HTTP flows attached as evidence. Each attached flow shows the full request and response inline, so you can review the exact traffic that demonstrates the vulnerability without switching tabs.

Scanner-generated findings automatically attach the probe flow -- the modified request that triggered the detection -- alongside the original baseline flow. You can attach additional flows manually to build a complete evidence chain (e.g., a setup request, the exploit, and the impact confirmation).

## Probe Data

For scanner-generated findings, the probe data section shows:

- The exact request sent by the scanner (with the injected payload highlighted)
- The response received
- The detection rationale (e.g., "time-based: baseline 45ms, payload response 5,230ms" or "error pattern matched: `You have an error in your SQL syntax`")
- The insertion point and original parameter value

This data is preserved with the finding and survives project export/import.

## Status Workflow

Findings follow a 9-state workflow:

```
Open --> Triaged --> Confirmed --> Reported --> Accepted --> Resolved
  |                     |
  +--> False Positive   +--> Duplicate
  |                     |
  +--> Won't Fix -------+
```

- **Open** -- newly created, not yet reviewed
- **Triaged** -- reviewed by the tester, awaiting confirmation
- **Confirmed** -- vulnerability verified, ready for reporting
- **Reported** -- sent to the program or client
- **Accepted** -- acknowledged by the receiving party
- **Resolved** -- fix deployed and verified
- **Duplicate** -- same issue already reported
- **Won't Fix** -- accepted risk, will not be remediated
- **False Positive** -- detection was incorrect

Additionally, two boolean toggles provide quick classification:

- **False positive** -- marks the finding as a false detection without changing the status
- **Verified** -- confirms the finding has been manually verified

## Filtering and Search

The Findings tab provides multiple filtering mechanisms:

- **Severity pills** -- click Critical, High, Medium, Low, or Info to filter by severity
- **Status pills** -- filter by workflow state (Open, Confirmed, Reported, etc.)
- **Source pills** -- filter by origin (Scanner, Manual, Plugin, Passive)
- **Search** -- free-text search across title, URL, evidence, and notes
- **Tags** -- filter by one or more tags

Filters combine with AND logic. Clicking a severity pill while a status filter is active shows only findings matching both criteria.

## Tags

Add tags to findings for custom organization. Common patterns include tagging by target component (`api`, `auth`, `admin`), by engagement phase (`recon`, `exploitation`), or by report section (`critical-path`, `appendix`).

Tags are managed per-finding:

- Add a tag by typing in the tag input and pressing Enter
- Remove a tag by clicking the X on the tag pill
- Filter the findings list by clicking any tag in the sidebar

## Export

Export all findings (or a filtered subset) as JSON for reporting, archival, or import into other tools.

```
findings action:"export"
```

The export includes all finding fields, attached flow IDs, probe data, tags, and status history. Evidence flows are referenced by ID; the flows themselves are part of the project export.

## MCP Tool

The `findings` MCP tool provides full programmatic access to the findings repository.

```
findings action:"list"
```

Key actions:

- `list` -- list all findings with optional filters (severity, status, source, search)
- `get` -- get full details for a specific finding by ID
- `create` -- create a new finding manually
- `update` -- modify finding fields (title, severity, status, evidence, etc.)
- `delete` -- remove a finding
- `toggle_false_positive` -- toggle the false positive flag
- `toggle_verified` -- toggle the verified flag
- `add_tag` -- add a tag to a finding
- `remove_tag` -- remove a tag from a finding
- `export` -- export findings as JSON
- `attach_flow` -- attach an evidence flow to a finding
- `detach_flow` -- remove an evidence flow from a finding

Example -- list all High and Critical findings that are confirmed:

```
findings action:"list" severity:"high,critical" status:"confirmed"
```

Example -- create a manual finding:

```
findings action:"create" title:"Reflected XSS in search parameter" target_url:"https://example.com/search?q=test" severity:"high" confidence:"certain" cwe:"CWE-79" evidence:"The q parameter reflects unencoded input in the response body."
```

Example -- update status to Reported:

```
findings action:"update" id:"<uuid>" status:"reported"
```

## API

```
GET    /api/findings                    List findings (query params: severity, status, source, search, limit, offset)
GET    /api/findings/:id                Get finding details
POST   /api/findings                    Create finding
PUT    /api/findings/:id                Update finding
DELETE /api/findings/:id                Delete finding
POST   /api/findings/:id/flows          Attach evidence flow
DELETE /api/findings/:id/flows/:flow_id Detach evidence flow
POST   /api/findings/export             Export findings as JSON
```
