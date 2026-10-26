## Appendix A: Example JSON `properties`

This appendix gives synthetic examples of claim `properties` at the payload sizes discussed in [Bytes per row](#bytes-per-row). They illustrate the payload size; they are not captured plugin output.

Each example is stored as a JSON fixture alongside this document. Fixtures end with one newline byte; subtract it from the file size to get JSON text bytes. For example, on macOS, `wc -c < rfc/rfc-011-appendix-example-json/baseline-470B.json` prints the file size in bytes. JSON text size is not PostgreSQL `jsonb` storage.

### Summary

| Variant | Claim | Properties | Example (JSON bytes) | Fixture |
| --- | --- | --- | --- | --- |
| Baseline | node | 10 | **470 B** | [`baseline-470B.json`](baseline-470B.json) |
| Baseline | relation | 1 | **146 B** | [`baseline-146B.json`](baseline-146B.json) |
| JSON x3 | node | 12 | **1,452 B** | [`json-x3-1452B.json`](json-x3-1452B.json) |
| JSON x3 | relation | 1 | **437 B** | [`json-x3-437B.json`](json-x3-437B.json) |
| JSON x9 | node | 9 | **4,479 B** | [`json-x9-4479B.json`](json-x9-4479B.json) |
| JSON x9 | relation | 1 | **1,314 B** | [`json-x9-1314B.json`](json-x9-1314B.json) |

JSON x50 is omitted because it represents only a boundary-case estimate.

### Example JSON size. 
For a flat object, count each property and the JSON punctuation separately:

```text
string property     = key bytes + unquoted value bytes + 5 B
                      (quotes around key and value, plus colon)
non-string property = key bytes + JSON value bytes + 3 B
                      (quotes around key, plus colon)
JSON text bytes     = sum(property bytes) + commas + 2 B for the braces
```

String `key` and `value` lengths exclude their surrounding quotes. For non-string values, count the bytes of the JSON representation (for example, `true` is 4 B).

---

### Baseline

#### Fixture: [`baseline-470B.json`](baseline-470B.json), 10 properties, 470 B

| Property | Key B | Value B | Syntax B | Property subtotal B |
|---|---:|---:|---:|---:|
| `project` | 7 | 41 | 5 | 53 |
| `sync_status` | 11 | 9 | 5 | 25 |
| `health_status` | 13 | 11 | 5 | 29 |
| `destination_server` | 18 | 73 | 5 | 96 |
| `destination_namespace` | 21 | 36 | 5 | 62 |
| `source_type` | 11 | 9 | 5 | 25 |
| `operation_in_progress` (`true`) | 21 | 4 | 3 | 28 |
| `last_deployed_at` | 16 | 29 | 5 | 50 |
| `last_operation_phase` | 20 | 7 | 5 | 32 |
| `last_operation_started_at` | 25 | 29 | 5 | 59 |
| **Total** | **163** | **248** | **48** | **459** |

`459 B` property bytes + `9 B` commas + `2 B` braces = **470 B** JSON text.

#### Fixture: [`baseline-146B.json`](baseline-146B.json), 1 property, 146 B

| Property | Key B | Value B | Syntax B | Property subtotal B |
|---|---:|---:|---:|---:|
| `paths` | 5 | 134 | 5 | 144 |

`144 B` property bytes + `0 B` commas + `2 B` braces = **146 B** JSON text.

---

