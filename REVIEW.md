---
pr: openshift-eng/ci-test-mapping#885
title: "TRT-2949: Allow JIRA functionality to use SA OAuth auth"
head_sha: 254aa1e7cc2f247a7b682524f1041efd250e5179
base: main
reviewed_at: 2026-10-14T13:26:35Z
verdict: approve
---
# Review

## Verdict

Approve. No actionable findings in the current revision.

The opt-in service-account OAuth path covers all three Jira commands while preserving legacy authentication selection. The blocking signal-handling regression reported against the earlier revision is addressed by removing global signal interception. The relevant package tests pass, including an uncached subprocess test confirming termination on the first SIGINT and SIGTERM while the real map-verify request is stalled. Live Atlassian access remains unverified.

## What this PR does

- Adds the default-false `--jira-service-account` flag to `map`, `jira-create`, and `jira-verify`.
- Shares Jira request construction and an OAuth client that obtains, caches, and replaces service-account access tokens.
- Preserves Bearer-then-Basic authentication and anonymous mapping reads when the flag is absent.
- Propagates command context cancellation through Jira requests, retry waits, and token grants while retaining default CLI signal termination.
- Documents credential configuration, scopes, project permissions, and rollout and rollback steps.

## Findings

No findings.

## Resolved

### [blocking] Restore process termination for existing commands

- where: `cmd/ci-test-mapping/ci-test-mapping.go:31-34`
- concern: The [external review comment](https://github.com/openshift-eng/ci-test-mapping/pull/885#issuecomment-6150649373) demonstrated that revision `98ba9eec98c2825243a201ca8c718387a8226c19` intercepted SIGINT and SIGTERM globally while existing HTTP, BigQuery, and interactive OAuth paths could ignore the canceled context. Current revision `254aa1e7cc2f247a7b682524f1041efd250e5179` removes `signal.NotifyContext` and calls `rootCmd.Execute()`, restoring default process termination for those paths. `TestCLITerminationSignals` invokes the real `Execute` and `map-verify` implementations with a stalled transport and verifies termination by the first signal; both signal cases passed uncached.
- excerpt: |
    err := rootCmd.Execute()
    if err != nil {
        log.WithError(err).Fatal("could not execute root command")
    }

### [nit] Share Jira metadata response handling

- where: `cmd/ci-test-mapping/jira_create.go:68-76`
- concern: The earlier saved review suggested sharing the duplicated metadata response-reading lifecycle. This suggestion is withdrawn as a review finding: the current defect-first review established no concrete regression or meaningful maintainability defect from this duplication. The code remains unchanged in this respect.
- excerpt: |
    resp, err := client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)

## Checked

- Reviewed the complete PR diff from merge base `7b06b90c499a53be163fd04985691614b63248d4` to PR head `254aa1e7cc2f247a7b682524f1041efd250e5179`, including command wiring, request paths, vendored OAuth code, tests, and documentation.
- Local `git rev-parse HEAD` reports `d2aa7b8e35a86a62161d7232c2a7cfb50e351aac`, an existing review-artifact-only commit directly atop the PR head. Frontmatter records the actual reviewed PR head so later refreshes compare the correct code revision.
- Checked all Jira call sites: the mapping component reader and both creation-metadata requests use the shared client and preserve the OAuth gateway prefix.
- Checked credential selection: the flag defaults to false, legacy Bearer credentials precede Basic credentials, service-account variables alone do not enable OAuth, and enabled OAuth fails without falling back to legacy credentials.
- Checked token reuse and replacement plus command-context cancellation against the implementation and existing tests.
- Passed `go test -mod=vendor ./pkg/jira ./pkg/jiraauth ./cmd/ci-test-mapping/flags ./cmd/ci-test-mapping`.
- Passed `go test -mod=vendor ./cmd/ci-test-mapping -run '^TestCLITerminationSignals$' -count=1 -v`; both `interrupt` and `terminated` subtests passed.
- Compared the signal-handling change directly against the revision cited in the external comment and confirmed that no production signal interception remains in repository Go code outside vendored dependencies.
- Confirmed the token grant format and gateway routing against [Atlassian's service-account OAuth guidance](https://support.atlassian.com/user-management/docs/create-oauth-2-0-credential-for-service-accounts/).
- Live credentials, gateway access, scopes, project permissions, and actual site responses were not tested. BigQuery operations and interactive Google OAuth were not exercised; removal of global signal interception addresses their shared termination regression.

## Open questions

- Could you link accessible TRT-2949 acceptance criteria? The ticket text was unavailable; the review used the PR description as its specification.
