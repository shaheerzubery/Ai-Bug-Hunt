# Plan: Automatically catching an AgentCollect-specific UX bug from PostHog session replays

## 1. Reframe the target

A "bug" here is rarely a crash — it's a button that does nothing, a control users click five times in frustration, or a page they abandon. For AgentCollect, the two highest-stakes surfaces are:

- **Debtor-facing payment portal** — setting up a payment plan or paying via the AI agent's link. A stuck flow here directly costs recovered revenue.
- **Client (creditor) dashboard** — where finance teams review collection status. Confusion here erodes trust in the product even if no money is at stake.

I'd start with whichever flow the team says is bleeding the most right now rather than guessing.

## 2. Clarifying questions I'd ask before building anything

- Which flow matters most right now — debtor payment setup, or the client-facing collections dashboard?
- Is PostHog autocapture already enabled for rage-click / dead-click / exception events, or only replay recording?
- What's an acceptable false-positive rate — is a human triaging flagged sessions, or does this need to auto-page someone?
- Are there elements that legitimately get multiple clicks by design (e.g., a "refresh status" button) that would look like rage clicks but aren't?


## 3. Detection approach, in order of effort

**Cheap, immediate:** Use PostHog's built-in `$rageclick`, `$dead_click`, and `$exception` autocapture events directly — no ML required. A scheduled HogQL query groups these by page + element and flags spikes.

**Funnel drop-off:** Define the critical flow as a PostHog funnel (e.g., "opened payment link → entered amount → submitted → confirmed"). Alert when step-to-step conversion drops abnormally week-over-week (simple % threshold vs. trailing average). This catches silent breakages that don't throw errors or rage clicks — e.g., a submit button that does nothing because of a JS error PostHog didn't tag as `$exception`.

**Session-level heuristics:** Flag sessions with high replay duration but low distinct-page-count on a single flow (thrashing), or repeated identical event sequences within a short window.

## 4. Alerting loop

Scheduled job (PostHog API / HogQL query on a cron) → dedupe/aggregate → post to Slack with a direct session-replay link, the flow name, and why it was flagged — so a human can watch the clip and confirm or dismiss in under a minute.

## 5. Validation before trusting it

Manually review a batch of flagged sessions against known non-bugs first (e.g., double-click habits, legitimate repeated actions), then tune thresholds before treating alerts as reliable.

## 6. What I'd explicitly not build in step 1

Anomaly-detection ML on replay video or DOM diffing — overkill for a first pass. Rule-based detection on existing autocapture events gets most of the value for a fraction of the effort, and is something a human can verify and trust immediately.
