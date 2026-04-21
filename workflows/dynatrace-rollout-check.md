# Dynatrace Rollout Check

Read `../workflows/dynatrace-investigation.md` first, then use this playbook when the question is whether a deployment or rollout caused degradation.

## Workflow

1. Define the rollout window.
- Anchor the analysis on the exact rollout start time if known.
- Default to a short comparison window on both sides of the rollout:
  - pre-window: `rollout start - 30 minutes` to `rollout start`
  - post-window: `rollout start` to `rollout start + 30 minutes`
- Expand the windows when the rollout was gradual, traffic is sparse, or degradation appears delayed.

2. Confirm the comparison is valid.
- Check whether traffic or processing volume is reasonably comparable between the pre-window and post-window.
- If the service was idle, heavily bursty, or partially rolled out, say the comparison is weak before drawing conclusions.
- Separate normal demand shifts from deployment-correlated regressions.

3. Start with high-signal regression checks.
- Run `list_problems` in the rollout window.
- Check service-level request or processing volume, failure rate, and latency or duration.
- Check deployment or change events if the environment exposes them.
- Note whether the first visible degradation starts before, during, or after the rollout.

4. Drill into the workload-specific failure surface.
- For HTTP services:
  - identify the endpoints or operations driving the regression
  - compare status code mix, exception types, and dependency failures
- For workers or consumers:
  - compare processing throughput, failure counts, retry noise, and backlog or lag symptoms
- For scheduled jobs:
  - compare run starts, completions, duration, skipped runs, and partial completions

5. Classify the conclusion explicitly.
- `correlated regression`
- `no regression observed`
- `inconclusive because comparison quality is weak`
- `degradation started before rollout`
- `degradation appears driven by downstream dependency or shared platform`
- `rollout correlated but code change likely unrelated`

6. Support the conclusion with direct evidence.
- Include the exact windows compared.
- Include the exact entities checked.
- Include the metrics, problems, logs, or events that make the conclusion defensible.
