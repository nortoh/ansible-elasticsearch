<!-- PR template -->
**Ticket:** <!-- Hyperlink the issue, e.g. [PROJ-123](https://liquidweb.atlassian.net/browse/PROJ-123)
(Jira) or [TEAM-123](https://linear.app/nexcess/issue/TEAM-123/slug) (Linear), or a GitHub issue —
required, or state why there isn't one -->

## Why

<!-- What problem does this solve, or what requirement does it fulfill? One sentence is usually
enough if there's a ticket — it should contain the details. -->

## What

<!-- What changed? Bullet points of the approach taken. -->

### Blast radius

<!-- e.g. single platform (Debian/RedHat) / all platforms / X-Pack only / defaults change -->

### Deviations from spec

<!-- Bullet points: what, if anything, changed between the ticket's description and the final
implementation, and why. Delete this section if there's no spec to deviate from. -->

## Reading guide

<!-- Optional for a small change. For anything touching more than one file, map it for reviewers
before they read the diff. -->

| File | What to look at | Why |
|---|---|---|
| `` | … | … |

## Test plan

**Automated tests added/updated:**
- [ ] …

**Manual verification steps:**
1. …

## Deploy notes

<!-- Delete this section if not needed. -->

**Key/secret changes:** <!-- new secrets to provision, keys to issue, or config to update? -->
**Rollout order:** <!-- any sequencing required (e.g. migrate before deploying)? -->
**Rollback:** <!-- how to revert if this causes a problem in production? -->

## Checklist

- [ ] `bundle exec kitchen test <PATTERN>` passes for any Kitchen suite this change affects
- [ ] Docs updated (`README.md`/`AGENTS.md`) if this changes how the role is built, run, or used
- [ ] No secrets, tokens, or real customer/financial data included
