# Handoff: grilling round 1 (paused 2026-11-11)

Read this first when you resume. It replaces the chat history from the session that produced it.

## What this project is

An AI-assisted **fail verification** component. It covers only the analysis logic and rules that decide why an automated test failed. The user, a software QA engineer, owns the overall system architecture separately, so don't design that. The user asked: **don't assume anything; ask.**

- **Stack under test:** Python, Robot Framework with SeleniumLibrary (web, PC). Appium (mobile, phone) comes later.
- **Scale:** about 17,000 test cases run daily, with more than 100 failures per day. Manual triage takes too long.
- **Inputs per failure:**
  - stack trace or error message
  - screenshot (sometimes)
  - page source (sometimes)
  - the full test framework code
  - Jira and Confluence: active and backlog bugs; requirements are still being compiled
- **Categories (user's):** env issue, real bug, flaky test, maintenance period, script issue, data issue.
- **Output per verdict:** category, confidence, evidence, next action (rerun, file bug, quarantine, ignore).
- **Approach:** an explicit ruleset first, then a human-like diagnostic path that reaches the root cause.

## Done so far

1. **Research:** [docs/research/failure-triage-initial-findings.md](../research/failure-triage-initial-findings.md). Every claim is cited and labelled verified, inference, proposal or unverified. It includes a draft 10-step diagnostic path, 34 draft rules and 14 open questions.
2. **Skills setup:** issues are tracked as local markdown under `.scratch/`, with the default triage labels and single-context domain docs. See [CLAUDE.md](../../CLAUDE.md) and `docs/agents/`.
3. **Glossary started:** [CONTEXT.md](../../CONTEXT.md) defines Failure group, Candidate and Platform.
4. **Grilling started** (`/grill-with-docs`): round 1 has been asked and nothing has been answered yet.

## Facts the user gave mid-round (settled)

- Failed tests are **grouped before analysis**, in this order: feature → platform (PC or phone) → **full** error message. Grouping is implemented for Selenium only; Appium grouping hasn't started.
- All tests in a group are assumed to share one root cause.
- **One candidate per group** is analysed; the fail analysis designed here runs on candidates.

## Where we are in the flow

`/research` ✅ → `/setup-matt-pocock-skills` ✅ → **`/grill-with-docs` ⏸ (round 1 open)** → (`/to-questionnaire` for questions other people must answer) → (`/prototype` on past failures if needed) → `/to-spec` → `/to-tickets` → `/implement`.

**To resume:** run `/grill-with-docs` and say "resume from docs/handoff/2026-11-11-grilling-round-1.md". Re-ask the open questions below, or take the user's answers to them, then compute round 2.

## Open questions (round 1), with recommendations

### Grouping-related (asked after the user explained grouping)

- **Q1a, exact or normalised message match:** messages contain IDs, numbers, session IDs and timestamps that differ between tests, so exact matching splits one root cause into many groups. Is the grouping key in scope for us?
  ➡️ In scope as a recommendation. Group on a **normalised message**, masking volatile values and keeping the locator pattern. Keep the full original message as evidence.
- **Q1b, group members with different causes:** example: same message, but 10 tests failed at 02:10 on node A (service down) and 2 at 05:40 on node B (script bug).
  ➡️ Run a **group consistency check** before copying the verdict. Split off any member whose failing keyword or step, failure time, node, device or environment, or known-flaky status differs from the candidate's.
- **Q1c, looking across groups:** a run-wide outage splits into one group per feature, which hides the "everything broke at 02:10" signal.
  ➡️ Add a **run-level pass**: the same normalised message, time window, or node or device across features is a suspected **incident**, which is strong env evidence.
- **Q1d, choosing the candidate:** how is it chosen today?
  ➡️ Choose the member with the **richest evidence**: screenshot plus page source, failure in the test body rather than setup or teardown, then failure time.
- **Q1e, what carries over to the group:**
  ➡️ Copy category, root cause, evidence and the linked Jira issue. Decide **per test** from each test's own history: quarantine, flake rate, rerun eligibility.
- **Q1f, mobile scope:**
  ➡️ **Selenium web only for v1**. Keep categories and the diagnostic path platform-neutral, with separate rule tables per platform.

### Categories and boundaries

- **Q2, category set:** is there a seventh outcome? How are setup failures handled? Is a known bug its own category?
  ➡️ Six categories plus **Undetermined** (needs a human). A setup failure is **inherited** from the setup, with its origin recorded. A known bug and a new bug are both **real bug**; known or new is a property of the verdict, not a category.
- **Q3, maintenance period:** where is the schedule kept? Whole environment or a single dependency? Grace period around the window? A maintenance banner with nothing on record?
  ➡️ A planned unavailability recorded in an **authoritative schedule** before the run, for the environment or a named dependency. A failure counts if it falls inside the window, or within a small grace period around it. Unplanned outage = **env issue** (flag "unregistered maintenance?").
- **Q4, flaky vs env:** scenario: the Selenium node dropped and the rerun on another node passed.
  ➡️ **Flaky** = the test or product timing varies under healthy conditions. **Env issue** = infrastructure or a dependency was unhealthy. A rerun pass only shows the cause was temporary, so the scenario is **env (transient)**.
- **Q5, data vs env:** scenarios:
  - (i) the user account is locked by a parallel test
  - (ii) the product is out of stock in the shared environment
  - (iii) a config or master-data table is wrong after a refresh
  - (iv) a data-setup API call failed because the backend was down

  ➡️ **Data issue** = records, accounts or state the test consumes are missing, wrong, locked or used up: (i) and (ii). **Env issue** = the system is down, misconfigured or not deployed: (iii) and (iv). **Still needs from the user:** is test data created per test, shared in the environment, or a mix?
- **Q6, script vs real bug when the UI changed:** a button id was renamed in the new build. Requirements aren't available yet.
  ➡️ A distinct finding, **UI drift**: verdict is script issue at low confidence, routed to a human. When a Jira story or requirement shows the change was intended, it becomes script issue at high confidence.
- **Q7, next action vocabulary:**
  ➡️ **rerun** (same node, fresh node, or after the window), **link to known issue**, **file product bug**, **file automation task** (script or data), **notify env owner**, **quarantine**, **ignore**, **send to human**. One primary action plus optional secondary ones.
- **Q8, facts only the user knows** (no recommendation):
  - how much run history is kept, and whether test IDs stay stable
  - auto-reruns: on the same or a different node
  - run metadata: app build, test commit, environment, browser or driver version, device
  - parallelism (pabot) and shared accounts
  - whether past manual triage verdicts are recorded anywhere

## Round 2 (depends on the round 1 answers, not yet asked)

- Precedence when one test shows several causes (research proposed real bug > data > script > env > flaky > maintenance).
- How confidence is expressed and calibrated (H/M/L vs numeric; shadow mode first).
- Which actions may run without a human, and what precision each needs.
- Known-bug matching against Jira and Confluence (which projects and issue types; whether the agent may create issues).
- The remaining research open questions not covered above (#10–#14 in the research file): artefact capture on every failure, environment health monitoring, external LLM vs on-premises.
