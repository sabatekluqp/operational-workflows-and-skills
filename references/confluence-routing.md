# Confluence Routing

Use these mappings when an analysis page should be routed by owning PagerDuty escalation policy.

## PagerDuty Escalation Policy to Confluence Folder

- `Shopping BE`
  - Escalation policy id: `PL6YO4G`
  - Space: `QD`
  - Folder id: `5289574404`
  - URL: `https://quadpay.atlassian.net/wiki/spaces/QD/folder/5289574404`

- `Decisioning Escalation`
  - Escalation policy id: `PKSYG3Y`
  - Space: `QD`
  - Folder id: `5289279502`
  - URL: `https://quadpay.atlassian.net/wiki/spaces/QD/folder/5289279502`

## Folder Handling

- The mapped ids above are Confluence folder ids, not page ids.
- When publishing programmatically, use the mapped folder id as the page `parentId` and expect the resulting parent type to be `folder`.
- Do not try to validate a folder id with page lookup APIs such as `getConfluencePage`; those calls return `404` for valid folders.

## Naming Rule

Prefix new analysis pages with the current timestamp in the user's timezone so pages sort by recency.

Recommended format:

- `<YYYY-MM-DD HH:mm z> - <topic>`

Example:

- `2026-03-25 14:37 EDT - Incident Q2YLVYYF9DVCJK - customers`
