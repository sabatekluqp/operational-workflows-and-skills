# Dynatrace GUID and Data Validation Trace

Read `../workflows/dynatrace-investigation.md` first, then use this playbook when the user wants to prove that an identifier or BI event did or did not flow through the expected path.

## Workflow

1. Build the identifier set and transformation hypotheses.
- Record every exact identifier supplied by the user.
- Add only justified variants:
  - case variants
  - dashless GUID
  - derived correlation id or request id when there is evidence of that relationship
- Do not invent relaxed search variants without a reason.

2. Discover the telemetry objects and fields before concluding absence.
- If the expected event, record, or BI signal is not already known, use semantic dictionary or schema discovery first.
- Identify:
  - likely data object
  - exact field name
  - service or entity expected to emit it
- Do not conclude that an event is missing until the relevant object and field names have been validated.

3. Start from the earliest expected producer.
- Search the earliest service or event that should contain the identifier.
- Record the earliest confirmed observation with:
  - timestamp
  - entity
  - object type
  - field name
  - exact matched value

4. Walk the chain one hop at a time.
- Move from upstream producer to downstream consumer or BI emission.
- For each hop, record:
  - expected system
  - expected object
  - expected field
  - whether the identifier was found
  - the last confirmed timestamp
- Call out where the trail stops rather than jumping directly to the final missing artifact.

5. Be strict about `missing BI event` conclusions.
- Only claim a downstream or BI event is missing when:
  - upstream evidence shows the source event really happened
  - the searched downstream object and field are correct
  - the searched time window is wide enough for the expected delay
- Otherwise say the answer is partial and describe the strongest remaining uncertainty.
