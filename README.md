# Superdesk AI Skills

A collection of [AI Agent skills](https://www.skills.sh/) for working with Superdesk projects. Install them with the `skills` CLI to give your AI agent procedural knowledge about Superdesk workflows.

## Install

All skills in this repo:

```
npx skills add superdesk/skills
```

A single skill (use the `--skill` flag):

```
npx skills add superdesk/skills --skill superdesk-e2e
```

## Available skills

### superdesk-e2e

Authors a Playwright end-to-end test from a plain-language UI scenario. Brings
up the local e2e stack, writes the spec following the repo's conventions, and
iterates to a deterministic pass with a trace artifact for review.

Scope: runs only inside `superdesk-client-core` or `superdesk-planning`. The
skill detects which repo it is in and stops if it is anywhere else.

### superdesk-ticket

Starts work on a Jira ticket: pulls it with `acli`, writes it to the local
`tickets/<KEY>/` directory, maps it to the right app (blueprint, stt, cp,
belga, ansa, newsroom-app-*) and core libraries, resolves the pinned release
branches from the app's `requirements.txt` and `client/package.json`, brings
up standalone mongo/redis/elasticsearch containers named after the ticket,
creates the matching pyenv, links the core checkouts, builds the client when
the ticket is UI-facing, and runs the narrowest relevant test suite.

Scope: stops at a running stack on the pinned release branch with a known
test state. It does not create the ticket branch, write the fix, or open a PR.
Three human checkpoints: after mapping, before installing, before testing.

## Layout

Each skill is a top-level directory containing a `SKILL.md`:

```
superdesk-e2e/
  SKILL.md
superdesk-ticket/
  SKILL.md
```

`skills.sh.json` controls how these skills are grouped and described on the
public [skills.sh](https://www.skills.sh/) directory.
