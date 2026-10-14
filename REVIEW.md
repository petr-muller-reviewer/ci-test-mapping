---
pr: openshift-eng/ci-test-mapping#885
title: "TRT-2949: Allow JIRA functionality to use SA OAuth auth"
head_sha: 98ba9eec98c2825243a201ca8c718387a8226c19
base: main
reviewed_at: 2026-10-13T11:47:43Z
verdict: approve
---
# Review

## Verdict

Approve. The opt-in service-account OAuth path covers all Jira consumers in this repository, and the existing Bearer, Basic, and unauthenticated mapping behaviors remain selected when the flag is off. I found no spec mismatch or behavior defect; one low-priority duplicated request-reading path is noted below.

## What this PR does

- Adds `--jira-service-account` to `map`, `jira-create`, and `jira-verify`.
- Shares Jira request construction and service-account OAuth configuration across those commands.
- Keeps legacy Bearer and Basic credentials, plus unauthenticated Jira reads from `map`, when OAuth is not selected.
- Binds API calls, retry waits, and OAuth token renewal to command cancellation.
- Documents the Jira reads, credential setup, permissions, and rollout and rollback steps.

## Findings

### [nit] Share Jira metadata response handling
- where: `cmd/ci-test-mapping/jira_create.go:68-76,124-132`
- concern: The two metadata readers each send the request, close its body, and read it into memory. A shared response-reading method on the Jira client would keep this lifecycle in one place; this is a low-priority cleanup, not a behavior defect.
- excerpt: |
    resp, err := client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)

    resp, err := client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)

## Standards

No repository-specific standards document was present. No hard standards violations were identified. The duplicated request-reading lifecycle above is a possible code smell, not a standards breach.

## Spec

No spec findings. The PR description and linked Sippy PR #4095 call for explicit opt-in OAuth while preserving the legacy path when the flag is absent. All Jira HTTP consumers found in this repository are covered by the new flag and shared client.

## Checked

- Confirmed the worktree is at PR head `98ba9eec98c2825243a201ca8c718387a8226c19`, based on `main` at `7b06b90c499a53be163fd04985691614b63248d4`.
- Inspected the Jira request sites, authentication selection, command wiring, tests, and documentation. No tests were run.
- Confirmed the Atlassian token endpoint and Jira cloud gateway URL match [Atlassian's service-account OAuth guidance](https://support.atlassian.com/user-management/docs/create-oauth-2-0-credential-for-service-accounts/).

## Open questions

- Could you add the TRT-2949 acceptance criteria to the PR description or link an accessible issue reference? The full ticket text was unavailable in this worktree, so this review used the PR description and linked Sippy PRs.
