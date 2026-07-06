# Quality Control API Frontend Usage

This document describes the QCS Bruno collection for frontend work. All endpoints use the Bruno `{{base_url}}` environment variable and inherit the collection auth.

Most list responses in the current Bruno scripts are treated as:

```json
{
  "data": []
}
```

When consuming list endpoints, expect records under `response.data`. Detail/create/update responses may return either a single object or a wrapped `data` value depending on backend serializer behavior, so keep the frontend API client tolerant while the contract is finalized.

## Main Concepts

| Term | Meaning | UI usage |
| --- | --- | --- |
| FMEA | Failure Mode and Effects Analysis document for a part or sub-part. | Risk table, process/failure/effect/cause scoring. |
| RPN template | Reusable risk template filtered by action priority (`ap`). | Suggested defaults for FMEA entries. |
| FMEA entry | One row/risk item inside an FMEA document. | Editable row with S/O/D scores and recommended action. |
| Corrective action | Follow-up action attached to an FMEA entry. | Task list for high-risk entries. |
| Control Plan | Quality control document linked to a part and optional FMEA document. | Defines inspection/control details for production. |
| Control Plan entry | One inspection/control row generated from worksteps or manually edited. | Editable inspection method row. |
| Inspection plan | Sampling plan for one control-plan entry and inquiry part. | Defines AQL and active inspection setup. |
| Inspection batch | A production batch being inspected against an inspection plan. | Batch status and inspection progress. |
| Inspection record | One sampled piece result inside a batch. | Pass/fail/rework input rows. |

## Important Keywords And Values

| Keyword | Type | Used in | Notes |
| --- | --- | --- | --- |
| `part` | number | FMEA documents, Control Plan documents, FMEA entries | Backend part id. Example: `5732`. |
| `sub_part` | number | FMEA sub-part scope | Optional sub-part id. Example: `54748`. |
| `inquiry_part` | number | Inspection plan | Inquiry part id. Example: `28018`. |
| `document` | number | FMEA entries, Control Plan entries filter | FMEA document id for FMEA entries; Control Plan document id for CP entries. |
| `fmea_document` | number | Control Plan document | Links Control Plan to its FMEA. |
| `control_plan_entry` | number | Inspection plan | The CP entry being inspected. |
| `plan` | number | Inspection batch filter | Inspection plan id. |
| `plan_id` | number | Create inspection batch | Request body field used by the Bruno request. |
| `batch` | number | Inspection records | Inspection batch id. |
| `batch_quantity` | number | Create inspection batch | Backend field for the total pieces in this production run. |
| `sequence_step` | number | Populate endpoints | Step size/order increment when generating entries from worksteps. Example: `10`. |
| `sequence_no` | number | Manual FMEA entry | Display/order number for a manual row. |
| `piece_sequence_no` | number | Inspection record | Sampled piece position in the batch. First-piece request uses `1`. |
| `ap` | `H`, `M`, `L` | RPN templates, FMEA entries | Action priority filter: High, Medium, Low. |
| `status` | string | Actions, batches | Examples: `In Progress`, `Completed`, `Open`. |
| `result` | `Pass`, `Fail`, `Rework` | Inspection records | Failed/rework records should include `defect_description` when available. |
| `is_active` | boolean | Inspection plans, templates | Query filter, sent as `true`/`false`. |
| `is_customized` | boolean | FMEA entries | `false` means generated/not manually reviewed yet. |
| `severity` | number | FMEA entry | S score. |
| `occurrence` | number | FMEA entry | O score. |
| `detection` | number | FMEA entry | D score. |
| `severity_new` | number | Corrective action update | Re-rated S score after action completion. |
| `occurrence_new` | number | Corrective action update | Re-rated O score after action completion. |
| `detection_new` | number | Corrective action update | Re-rated D score after action completion. |
| `aql_level` | number | Inspection plan | Sampling/AQL level. Example: `2.5`. |
| `revision` | number | FMEA/Control Plan headers | Document revision number. |
| `changed_index` | number | Control Plan header update | Revision/change index. |
| `date_first_release` | date | Control Plan document | ISO date string. |
| `date` | date | FMEA document update | ISO date string. |
| `due_date` | date | Corrective action | ISO date string. |
| `completion_date` | date | Corrective action update | ISO date string. |

## Recommended Frontend Workflow

1. Get or create the FMEA document for a part.
2. Populate FMEA entries from backend worksteps.
3. Let QC review entries, edit S/O/D scores, and create corrective actions for high-risk rows.
4. Create a Control Plan document linked to the FMEA document.
5. Populate Control Plan entries from worksteps, then let QC fill inspection method details.
6. Create an Inspection Plan for a Control Plan entry and inquiry part.
7. Create an Inspection Batch for the plan.
8. Submit inspection records, close the batch, then re-fetch related FMEA entries if RPN values are recalculated by backend logic.

## FMEA Templates

### List active RPN templates

```http
GET /qcs/fmea/rpn-templates/?is_active=true
```

Query params:

| Param | Value |
| --- | --- |
| `is_active` | `true` |

### Filter templates by action priority

```http
GET /qcs/fmea/rpn-templates/?ap=H
```

Use `ap=H`, `ap=M`, or `ap=L`.

### Get a template by id

```http
GET /qcs/fmea/rpn-templates/{rpn_template_id}/
```

## FMEA Documents

### List or pull FMEA document for part

```http
GET /qcs/fmea/documents/?part={part_id}
```

The Bruno collection stores the first returned document id as `fmea_doc_id` and its part id as `part_id`.

### Get full FMEA document detail

```http
GET /qcs/fmea/documents/{fmea_doc_id}/
```

Use this for the document detail screen. The request name indicates the response includes nested entries and subpart groups.

### Update FMEA document header

```http
PUT /qcs/fmea/documents/{fmea_doc_id}/?revision=1&date=2026-06-30
```

Query params:

| Param | Value |
| --- | --- |
| `revision` | New revision number |
| `date` | Header/revision date |

### Populate FMEA document from worksteps

```http
POST /qcs/fmea/documents/{fmea_doc_id}/populate/
```

Body:

```json
{
  "sequence_step": 10
}
```

Use this when a new FMEA document needs generated entries, or after new worksteps are added to the part.

### Sub-part scoped FMEA

The collection includes a "Create Subpart Scope" request with:

```http
GET /qcs/fmea/documents/
```

Body shown in Bruno:

```json
{
  "part": 5732,
  "sub_part": 54748,
  "revision": 0,
  "date": "2026-07-28"
}
```

Note: GET requests normally do not create records or carry JSON bodies. Confirm whether the backend expects this to be `POST /qcs/fmea/documents/` before implementing the frontend.

## FMEA Entries

### List entries for a document

```http
GET /qcs/fmea/entries/?document={fmea_doc_id}
```

### Filter entries by part

```http
GET /qcs/fmea/entries/?part={part_id}
```

### Filter entries by action priority

```http
GET /qcs/fmea/entries/?document={fmea_doc_id}&ap=H
```

Use `ap=H`, `ap=M`, or `ap=L`.

### Filter generated entries not yet manually reviewed

```http
GET /qcs/fmea/entries/?document={fmea_doc_id}&is_customized=false
```

### Get/update a single entry

```http
GET /qcs/fmea/entries/{entry_id}/
PUT /qcs/fmea/entries/{entry_id}/
```

The Bruno request named "get single entry" currently calls the list URL with `document`; use the detail URL above if the backend exposes standard detail routing.

Update body example:

```json
{
  "severity": 8,
  "occurrence": 5,
  "detection": 4,
  "potential_effects": "Batch scrap and production halt",
  "recommended_actions": "100% inspection required before next process"
}
```

### Create a manual FMEA entry

```http
POST /qcs/fmea/entries/
```

Body:

```json
{
  "document": 1,
  "sequence_no": 997,
  "process_name": "Raw Material Incoming",
  "potential_effects": "Batch scrap / Affect parts capability",
  "potential_failure": "Raw Material Error",
  "potential_causes": "Purchase error/ Mixed material",
  "current_controls": "IQC check material Certificate",
  "recommended_actions": "PD, Store, QC overlapping control",
  "severity": 8,
  "occurrence": 3,
  "detection": 4
}
```

## Corrective Actions

### List actions for an entry

```http
GET /qcs/fmea/actions/?entry={entry_id}
```

### Filter actions by status

```http
GET /qcs/fmea/actions/?entry={entry_id}&status=In%20Progress
```

Known statuses from the Bruno collection:

| Status | Usage |
| --- | --- |
| `In Progress` | Action is open/being worked. |
| `Completed` | Action is done and may include re-rated scores. |

### Create corrective action

```http
POST /qcs/fmea/actions/
```

Body:

```json
{
  "entry": 7,
  "action_taken": "Added 100% incoming inspection. Update work instruction WS-04.",
  "status": "In Progress",
  "due_date": "2026-07-15"
}
```

### Complete/update corrective action

```http
PUT /qcs/fmea/actions/{action_id}/
```

Body:

```json
{
  "status": "Completed",
  "completion_date": "2026-07-10",
  "severity_new": 10,
  "occurrence_new": 2,
  "detection_new": 3
}
```

## Control Plan Documents

### List Control Plans for a part

```http
GET /qcs/control-plan/documents/?part={part_id}
```

### Create Control Plan document

```http
POST /qcs/control-plan/documents/?part={part_id}
```

Body:

```json
{
  "part": 5732,
  "fmea_document": 1,
  "revision": 0,
  "date_first_release": "2026-06-29"
}
```

### Retrieve Control Plan document

```http
GET /qcs/control-plan/documents/{control_plan_document_id}/
```

### Update Control Plan header

```http
PUT /qcs/control-plan/documents/{control_plan_document_id}/
```

Body:

```json
{
  "revision": 1,
  "changed_index": 1
}
```

### Populate Control Plan entries from worksteps

```http
POST /qcs/control-plan/documents/{control_plan_document_id}/populate/
```

Body:

```json
{
  "sequence_step": 10
}
```

Note: The Bruno request "Create matching Control Plan for the same Subpart scope" currently points to `/qcs/fmea/documents/2/populate/` while sending Control Plan fields. Confirm whether this should instead call the Control Plan document endpoint.

## Control Plan Entries

### List entries for a Control Plan document

```http
GET /qcs/control-plan/entries/?document={control_plan_document_id}
```

### Filter entries by part

```http
GET /qcs/control-plan/entries/?part={part_id}
```

### Get a Control Plan entry

```http
GET /qcs/control-plan/entries/{control_plan_entry_id}/
```

### Update inspection method details

```http
PUT /qcs/control-plan/entries/{control_plan_entry_id}/
```

Body:

```json
{
  "specification_tolerance": "Per drawing DIN ISO 2768-m",
  "evaluation_method": "Visual + Height gauge",
  "control_method": "Height gauge, Micrometer",
  "reaction_plan": "Stop production, tag defective parts, inform QC supervisor",
  "sample_frequency": "Each batch"
}
```

These fields are the main editable frontend fields for finalizing a Control Plan row.

## Inspection Plans

### List plans for a Control Plan entry

```http
GET /qcs/control-plan/inspection-plans/?control_plan_entry={control_plan_entry_id}
```

### Filter active plans

```http
GET /qcs/control-plan/inspection-plans/?is_active=true
```

### Create inspection plan

```http
POST /qcs/control-plan/inspection-plans/
```

Body:

```json
{
  "control_plan_entry": 1,
  "inquiry_part": 28018,
  "aql_level": 2.5
}
```

### Get plan detail

```http
GET /qcs/control-plan/inspection-plans/{inspection_plan_id}/
```

## Inspection Batches

### Create inspection batch

```http
POST /qcs/control-plan/inspection-batches/
```

Body:

```json
{
  "plan_id": 1,
  "batch_quantity": 100
}
```

If `batch_quantity` is omitted, the backend uses the inspection plan's `planned_batch_quantity`.

### Filter open batches

```http
GET /qcs/control-plan/inspection-batches/?plan={inspection_plan_id}&status=Open
```

Known batch status:

| Status | Usage |
| --- | --- |
| `Open` | Batch can still receive inspection records. |

### Close batch manually

```http
POST /qcs/control-plan/inspection-batches/{batch_id}/close/
```

After closing, re-fetch the affected FMEA entry if the UI shows updated risk/RPN values.

## Inspection Records

### List records for a batch

```http
GET /qcs/control-plan/inspection-records/?batch={batch_id}
```

### Filter failed records

```http
GET /qcs/control-plan/inspection-records/?batch={batch_id}&result=Fail
```

### Get a record

```http
GET /qcs/control-plan/inspection-records/{record_id}/
```

### Submit one inspection record

```http
POST /qcs/control-plan/inspection-records/
```

Body:

```json
{
  "batch": 1,
  "piece_sequence_no": 1,
  "result": "Pass",
  "notes": "First piece check OK"
}
```

### Bulk submit inspection records

```http
POST /qcs/control-plan/inspection-records/bulk/
```

Body:

```json
{
  "batch": 1,
  "records": [
    {
      "piece_sequence_no": 15,
      "result": "Pass"
    },
    {
      "piece_sequence_no": 30,
      "result": "Fail",
      "defect_description": "Surface crack depth 0.3"
    },
    {
      "piece_sequence_no": 70,
      "result": "Rework",
      "defect_description": "Minor burr on edge"
    }
  ]
}
```

Allowed `result` values seen in the collection:

| Result | UI meaning |
| --- | --- |
| `Pass` | Piece passed inspection. |
| `Fail` | Piece failed inspection; show defect description input. |
| `Rework` | Piece requires rework; show defect/rework notes input. |

## Frontend Implementation Notes

- Use id placeholders instead of hard-coded Bruno sample ids (`5732`, `1`, `7`, `28018`) in app code.
- Encode query values with spaces, for example `status=In%20Progress`.
- Dates should be sent as ISO `YYYY-MM-DD` strings.
- Scores are numeric. The collection examples use integer S/O/D values.
- For high-priority/risk screens, use `ap=H` and the corrective actions endpoints together.
- For QC review queues, `is_customized=false` is useful for generated FMEA rows that still need human review.
- Re-fetch parent/detail resources after populate, bulk submit, close batch, and corrective-action completion because backend logic may generate rows or recalculate risk values.
- Confirm the noted collection quirks before freezing the frontend API client: sub-part create using `GET` with a body, and the sub-part Control Plan request pointing at an FMEA populate endpoint.
