# Overview

- Service:
- Exact date range:
- Environments:
- Conclusion:

# Entity Mapping

- eastus dev entities:
- eastus prod entities:
- Mapping caveats:

# Tier Legend

- Very high:
- High:
- Medium:
- Low:
- Very low:
- None observed:

# Endpoint Inventory

- Include one row per canonical endpoint with:
  - HTTP verb
  - canonical path
  - eastus dev tier and approximate count
  - eastus prod tier and approximate count
  - notes

# Operational / Non-Business Observations

- Health checks, Swagger, malformed paths, and other excluded noise:

# Recommendation

- Safest Deprecation Candidates First:
- Next Wave After Caller Validation:
- Clearly Not Cleanup Targets:

# Ambiguities / Caveats

- `None observed` means no matching canonical endpoint series was observed in Dynatrace, not proof that usage is impossible.
- Additional caveats:

# Dynatrace Data Used

- Include the exact DQL or metric query shape used to support the page.
