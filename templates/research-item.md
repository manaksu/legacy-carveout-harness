### R-NN · <one-line title>

| Field | Value |
|---|---|
| Layer | L0–L6 |
| Kind | auto · api · human |
| Owner | Harness, or a named role |
| Status | open · waiting · resolved · waived |
| Holds | units this item blocks, e.g. `ELGB120 1100-FETCH-SPANS` |
| Opened | YYYY-MM-DD |
| Closed | YYYY-MM-DD, or blank |

**Why it matters**
What the harness cannot do until this is resolved, and how many golden cases or members it affects.

**Action plan**
1. Who does what first.
2. What the harness does once that is done.
3. How the held units move forward.

**API call** (kind `api` only)
```
GET /<service>/v1/<resource>?<params>
  read-only, masked source: <replica or unload name>
  returns: { <fields> }
```

**Resolution**
What was decided or done, by whom, and the ledger rule or decision it produced.
