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

## Layout

Each skill is a top-level directory containing a `SKILL.md`:

```
superdesk-e2e/
  SKILL.md
```

`skills.sh.json` controls how these skills are grouped and described on the
public [skills.sh](https://www.skills.sh/) directory.
