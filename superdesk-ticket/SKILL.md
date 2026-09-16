---
name: superdesk-ticket
description: |
  Start work on a Jira ticket for a Superdesk or Newsroom project: pull the
  ticket with `acli`, write it to the local `tickets/` directory, work out
  which app (blueprint, stt, cp, belga, ansa, newsroom-app-*) and which core
  libraries (superdesk-core, superdesk-client-core, superdesk-planning) it
  touches, resolve the pinned release branches, bring up standalone mongo,
  redis and elasticsearch containers, create the matching pyenv, link the
  core checkouts, build the client if needed, and run the relevant tests
  against that stack.

  Invoke when the user says "start on TICKET-123", "set up for STT-1725",
  "pull SDESK-8014 and get an env running", "spin up the stack for this
  ticket", or pastes a Jira key and wants a working local reproduction
  environment before investigating.

  Do not invoke for: writing the fix itself, opening PRs, e2e test authoring
  (use `superdesk-e2e`), or tickets outside the Superdesk / Newsroom family.
---

# Starting work on a Superdesk ticket

You are turning a Jira key into a running, ticket-specific local stack plus a
ticket file the developer keeps notes in. The developer is technical and
usually knows the answer to any question you have, so ask short concrete
questions at the defined checkpoints and otherwise proceed. Be disciplined
about the order: every step feeds the next, and skipping the version
resolution is how you end up testing a fix against the wrong branch.

Paths below are relative to the **workspace root**: the directory that holds
the sibling checkouts (`superdesk-core/`, `superdesk-client-core/`,
`superdesk-planning/`, `superdesk-stt/`, `tickets/`, ...). Detect it by
walking up from the current directory until a directory containing
`superdesk-core/` and `tickets/` is found. If there is no such directory,
stop and ask where the workspace is.

## Three mandatory checkpoints

Stop and confirm with the user at these three points, even when the answer
looks obvious:

1. **After mapping the ticket** (end of step 3): which app repo, which core
   repos, which branches, which Python and Node versions, which containers
   and ports. The user corrects this once and everything downstream is right.
2. **Before installing anything** (start of step 6): creating a pyenv, running
   `pip install`, `npm install` or `npm run build` takes minutes and can
   overwrite a linked environment the user is relying on for another ticket.
3. **Before running tests** (start of step 7): name the exact suite and
   command. Test suites share one mongo and elasticsearch test database, so
   running the wrong one or running two concurrently corrupts someone else's
   run.

## Workflow (follow in order)

### 1. Pull the ticket

```
acli jira workitem view KEY-123 --fields '*all' --json
```

Read summary, description, status, assignee, reporter, priority, fix version,
labels, created date, parent, linked issues, comments and attachments. If
`acli` reports it is not authenticated, tell the user to run
`acli jira auth login` themselves and stop; do not try to authenticate on
their behalf.

If the description is empty, say so explicitly in the ticket file and in your
reply. Do not invent requirements from the title.

### 2. Write the ticket file

Create `tickets/KEY-123/KEY-123.md`. If the directory already exists, read
the existing file first and only refresh the Jira-sourced header and
description; never overwrite investigation notes or attachments that are
already there.

Use this layout, which matches the existing ticket files:

```markdown
# KEY-123: <summary>

- **Type:** Bug
- **Status:** <Jira status>
- **Priority:** <priority>
- **Assignee:** <name> (<email>)
- **Reporter:** <name>
- **Jira:** https://sofab.atlassian.net/browse/KEY-123
- **Fix version:** <fix version or none>
- **Labels:** <labels or none>
- **Created:** YYYY-MM-DD
- **Parent:** <epic key or none>

## Summary
<description, lightly reformatted; keep the reporter's wording>

## Reproduce steps
<numbered list from the ticket, or "not given">

## Expected / Actual
<if the ticket has them>

## Notes
<comments summarised with author and date; attachment filenames; links>

## Environment
<filled in at step 5; see the template there>

## Investigation
<left for the developer>
```

Download attachments the ticket references (screenshots, recordings, sample
files) into the same directory when `acli` exposes a URL for them; note the
filename under Notes. Do not download anything larger than 50 MB without
asking.

### 3. Map the ticket to an app and to core repos

Start from the Jira project key, then confirm against the ticket text.

| Jira key | App repo | Product | Core libraries |
|----------|----------|---------|----------------|
| SDESK | `superdesk/` (blueprint) unless the ticket names a client | Superdesk | superdesk-core, superdesk-client-core, superdesk-planning |
| STT | `superdesk-stt/`, or `newsroom-app-stt/` when the ticket is about NewsHub (`newshub.pro` URLs, My Topics, wire, agenda) | Superdesk / NewsHub | as above, or newsroom-core |
| SDCP | `superdesk-cp/` | Superdesk | superdesk-core, superdesk-client-core, superdesk-planning, superdesk-analytics |
| CPCN | `newsroom-app-cp/` | NewsPro | newsroom-core, superdesk-core |
| NHUB | `newsroom-app-stt/` | NewsHub | newsroom-core, superdesk-core |
| SDBELGA | `superdesk-belga/` | Superdesk | superdesk-core, superdesk-client-core, superdesk-planning, superdesk-analytics |
| SDANSA | `superdesk-ansa/` | Superdesk | superdesk-core, superdesk-client-core |
| SDEWTN | `superdesk-ewtn/` | Superdesk | superdesk-core, superdesk-client-core |

Then read the ticket for the **layer** where the defect lives, because that
decides which core repos get linked and built:

- Server behaviour (API responses, publishing, ingest, celery tasks, emails,
  data migrations): superdesk-core or superdesk-planning server side, or the
  app's own `server/` package (`stt/`, `cp/`, `belga/`).
- UI behaviour (editor, lists, previews, filters, modals): superdesk-client-core
  or superdesk-planning client side, or the app's `client/extensions/` and
  `client/superdesk.config.js`.
- Planning, events, assignments, coverages, agendas: superdesk-planning, both
  sides.
- Configuration and vocabularies: the app's `server/data/` only; no core repo
  needs linking.

When the ticket mentions an instance URL, use it: `stt2-next` or `-next`
hostnames mean the app's `next` or `develop` branch, `-uat` means `uat`, a
plain production hostname means the app's default branch and the release
branch it pins.

**Checkpoint 1.** Present the mapping as a short table (app repo and branch,
each core repo and branch, layer, Python and Node versions once known from
step 4) and wait for confirmation. If you could not decide between two apps
or two layers, say which and why.

### 4. Resolve versions and branches

Everything comes from the app repo. Do not guess from memory; branches move.

1. **App branch.** `git -C <app> branch --show-current`. If the ticket points
   at a different environment branch (`next`, `uat`, `develop`), check the
   working tree is clean and check that branch out; if it is dirty, stop and
   ask.
2. **Server pins.** In `<app>/server/requirements.txt` find the lines for
   `superdesk-core`, `superdesk-planning`, `superdesk-analytics` and
   `newsroom-core`. They look like
   `superdesk-core @ git+https://github.com/superdesk/superdesk-core.git@release/3.6`.
   The part after `@` is the branch, tag or commit to check out. Ignore the
   `# via superdesk-core` comment lines.
3. **Client pins.** In `<app>/client/package.json` find `superdesk-core` and
   `superdesk-planning` under `dependencies`. Note that the npm package named
   `superdesk-core` **is the superdesk-client-core repo**. The ref after `#`
   is the branch. `<app>/client/package-lock.json` pins the exact commit
   under `node_modules/superdesk-core` and `node_modules/superdesk-planning`;
   record both the branch and the locked commit.
4. **Python version.** Read `python-version` in
   `<app>/.github/workflows/tests.yml` (or `ci-*.yml`). Fall back to: release
   3.6 and `develop` use 3.12, release 3.5 and earlier use 3.10. Check which
   interpreters pyenv has with `pyenv versions --bare`.
5. **Node version.** `volta.node` in `<app>/client/package.json`. Volta picks
   it up automatically when commands run from inside that directory.
6. **Core checkouts.** For each core repo the ticket needs, run
   `git -C <repo> fetch upstream` (the upstream remote is `superdesk/<repo>`;
   `origin` is the developer's fork), then `git -C <repo> status --short`
   and `git -C <repo> branch --show-current`. If the checkout is on another
   branch or dirty, do not stash or switch on your own; list what is there
   and ask at checkpoint 1. When clear, check out the pinned branch tracking
   upstream, for example `git checkout -B release/3.6 upstream/release/3.6`.
   Stop at the pinned release branch; the developer creates the ticket
   branch themselves.

Record everything in the ticket file's Environment section:

```markdown
## Environment
- **App:** superdesk-stt @ `develop`
- **superdesk-core:** `release/3.6` (locked `5d1d89e0` in requirements) — linked editable
- **superdesk-client-core:** `release/3.6` (locked `83f30e2` in package-lock) — npx linked
- **superdesk-planning:** `release/3.6` — not linked (ticket is not planning)
- **Python:** 3.12.7, pyenv `superdesk-stt-release-3-6`
- **Node:** 22.22.0 (volta)
- **Containers:** `sd-KEY-123-mongo` 27017, `sd-KEY-123-redis` 6379, `sd-KEY-123-elastic` 9200
- **Data:** reused `superdesk-stt/data/` | fresh `tickets/KEY-123/data/` + `app:initialize_data`
- **Tests:** <command from step 7>
```

### 5. Bring up standalone containers

One container each for mongo, redis and elasticsearch, named after the
ticket so they can be stopped without touching anyone else's stack:
`sd-<KEY>-mongo`, `sd-<KEY>-redis`, `sd-<KEY>-elastic`.

Take the **image tags** from the app's `docker-compose.services.yml` (or
`docker-compose.local.yml` for newsroom apps) rather than hardcoding; at the
time of writing they are `mongo:4`, `redis:3` (newsroom: `redis:alpine`) and
`docker.elastic.co/elasticsearch/elasticsearch:7.10.1`.

**Data directory.** If `<app>/data/mongodb`, `<app>/data/redis` and
`<app>/data/elastic` exist (the compose file creates them on first run),
mount those: the developer wants the existing users, desks and content, and
`app:initialize_data` must not run. Otherwise the ticket gets fresh data:
create `tickets/<KEY>/data/{mongodb,redis,elastic}`, mount those, and plan
to run `app:initialize_data` in step 6. Say which case applied.

**Ports.** Prefer the default ports 27017, 6379 and 9200, because test
suites and settings default to them and the newsroom test conftest
hardcodes them. First run `docker ps --format '{{.Names}} {{.Ports}}'`. If
another container already holds a default port, do not stop it silently.
Ask the user: stop that stack, or run this one on an offset. If they choose
an offset, write `tickets/<KEY>/env.sh` exporting every URL the server reads
and source it before any server command:

```bash
export MONGO_URI=mongodb://localhost:27018/superdesk
export CONTENTAPI_MONGO_URI=mongodb://localhost:27018/superdesk_capi
export PUBLICAPI_MONGO_URI=mongodb://localhost:27018/superdesk_papi
export LEGAL_ARCHIVE_URI=mongodb://localhost:27018/superdesk_legal
export ARCHIVED_URI=mongodb://localhost:27018/superdesk_archive
export ELASTICSEARCH_URL=http://localhost:9201
export REDIS_URL=redis://localhost:6380/1
export CELERY_BROKER_URL=redis://localhost:6380/1
```

Start the containers:

```bash
docker run -d --name sd-KEY-mongo   -p 27017:27017 -v "$DATA/mongodb:/data/db" mongo:4
docker run -d --name sd-KEY-redis   -p 6379:6379   -v "$DATA/redis:/data" redis:3
docker run -d --name sd-KEY-elastic -p 9200:9200   -e discovery.type=single-node \
  -v "$DATA/elastic:/usr/share/elasticsearch/data" \
  docker.elastic.co/elasticsearch/elasticsearch:7.10.1
```

If a container with the ticket's name already exists (a previous session),
`docker start` it instead of recreating it. Wait until
`curl -sf http://localhost:9200/` answers and `mongosh --eval 'db.runCommand({ping:1})'`
(or `docker exec sd-KEY-mongo mongo --eval ...` on mongo 4) succeeds before
moving on.

Tear-down for the hand-off note: `docker rm -f sd-KEY-mongo sd-KEY-redis sd-KEY-elastic`.
Do not run it yourself.

### 6. Python environment, links, client build

**Checkpoint 2** first: list the pyenv name, whether it exists already, the
pip installs, the editable links and whether a client install and build are
planned. Wait for confirmation.

**pyenv.** Name it `<app-repo>-<branch>` with `/` and `.` replaced by `-`:
`superdesk-stt-release-3-6`, `superdesk-cp-develop`, `newsroom-app-cp-develop`.
Reuse it if `pyenv versions --bare` lists it; otherwise
`pyenv virtualenv <python> <name>`. Always invoke the interpreter by its full
path, `~/.pyenv/versions/<python>/envs/<name>/bin/python`, so the wrong env
is never picked up from a stale `.python-version`.

**Server install**, from `<app>/server/`:

```bash
$PY -m pip install -U pip wheel setuptools
$PY -m pip install -Ur requirements.txt
$PY -m pip install -r dev-requirements.txt
```

`dev-requirements.txt` pins `aiomoto[sqs]==0.5.3`, which needs Python 3.11+.
On a 3.10 env that line makes pip abort the **whole** dev install silently.
Verify with `$PY -m pytest --version` and install pytest, behave and their
plugins individually if it is missing.

**Editable links**, only for the core repos identified in step 3, after they
are on the pinned branch:

```bash
$PY -m pip install -Ue ../../superdesk-core
$PY -m pip install -Ue ../../superdesk-planning
```

Confirm the link took: `$PY -c 'import superdesk, planning; print(superdesk.__file__)'`
must print a path inside the workspace, not `site-packages`.

**Fresh data only.** If step 5 created a fresh data directory:

```bash
$PY manage.py app:initialize_data
$PY manage.py users:create -u admin -p admin -e admin@localhost --admin
```

**Client**, only when the layer is client-side. From `<app>/client/`:

```bash
npm install
npx --yes link ../../superdesk-client-core      # only if client-core is the repo under investigation
npx --yes link ../../superdesk-planning         # only if planning client code is involved
npm run build
```

Known build failures and their fixes, apply without being told:

- `Could not load TypeScript`: the thin app `client/package.json` does not
  pin typescript, so it nests under `node_modules/superdesk-core/`. Add
  `"typescript": "~4.9.5"` to the client `devDependencies` (match the version
  in superdesk-client-core's package.json), run `npm install` again. Tell the
  user this edit is local and must not be committed.
- After `npx link`ing client-core, webpack rebuilds the planning extension
  into a nested `dist` and fails with `Can't resolve '../../../interfaces'`.
  If the ticket does not need planning, comment out the `planning-extension`
  entry in `<app>/client/index.js` for the session and say so; if it does,
  link planning as well.
- `npx link` does not install the linked package's own dependencies. Run
  `npm install` inside `superdesk-client-core/` or `superdesk-planning/`
  before linking if their `node_modules` is missing.
- Links are destroyed by any later `npm install`; re-link after one.

**Running the app** (only when the ticket needs manual reproduction, and
after telling the user which ports it will bind): server `honcho start` from
`<app>/server/` with the env sourced, client `npm run start` from
`<app>/client/`, which serves on 9000 and proxies to the API on 5000.

### 7. Run the relevant tests

**Checkpoint 3** first: name the suite and the exact command, and confirm
no other pytest or behave run is in progress on this machine, because the
suites share one test database on the default ports.

Pick the **narrowest** suite that covers the layer from step 3:

| Layer | From | Command |
|-------|------|---------|
| superdesk-core server | `superdesk-core/` | `$PY -m pytest --log-level=ERROR --disable-warnings tests/<area>` then, for feature-level behaviour, `$PY -m behave --format progress2 --logging-level=ERROR features/<file>.feature` |
| superdesk-planning server | `superdesk-planning/server/` | `$PY -m pytest --log-level=ERROR --disable-warnings planning/<area>`; behave features live in `superdesk-planning/server/features/` |
| app server (`stt/`, `cp/`, `belga/`) | `<app>/server/` | `$PY -m pytest tests/` and `$PY -m behave --format progress2` |
| superdesk-client-core | `superdesk-client-core/` | `npm run unit` (karma); `npm run lint` and `npm run typecheck` before hand-off |
| superdesk-planning client | `superdesk-planning/` | `npm run test`; `npx tsc --noEmit -p src/tsconfig.json` |
| newsroom-core | `newsroom-core/` | `$PY -m pytest tests/<area>`; JS `CHROME_BIN=/usr/bin/google-chrome npm test` |
| UI reproduction | either core repo | invoke the `superdesk-e2e` skill rather than driving Playwright here |

Run a single file first to prove the environment works, then the directory.
Report failures verbatim with the failing test id and the assertion. A
failure that reproduces the ticket is a success for this skill: say that
plainly and stop. Do not start fixing product code; the skill ends at a
running stack with a known test state.

Record the command that was run and its result in the ticket file's
Environment section.

### 8. Hand off

Reply with: the ticket file path, the mapping table, the container names and
ports, the pyenv name and interpreter path, which repos are linked, what
data was used, the test command and its result, and the tear-down command.
Everything in that reply must also be in the ticket file so the next session
can pick it up from there.

## Failure modes

### `acli` returns nothing or an auth error
Tell the user the exact command that failed and ask them to run
`acli jira auth login` in their own terminal. Do not paste tokens anywhere.

### Two apps are plausible
STT tickets in particular can be Superdesk (`superdesk-stt`) or NewsHub
(`newsroom-app-stt`). Look for `newshub.pro` URLs, "My Topics", "wire",
"agenda", "scheduled notifications" (NewsHub) versus "desk", "publish",
"ingest", "planning" (Superdesk). If still unsure, ask at checkpoint 1 with
both options.

### The pinned branch is a commit hash, not a branch
`requirements.txt` sometimes pins `@<sha>`. Check out that sha in the core
repo (`git checkout <sha>`) and say so; the developer decides whether to
move to branch HEAD.

### A core checkout is dirty or on a ticket branch
Do not stash, reset or switch. List the branch and `git status --short`
output at checkpoint 1 and let the developer decide. Another ticket's work
may be sitting there uncommitted.

### The pyenv exists but was rewired for a different ticket
`pip show superdesk-core` shows `Location:` pointing at an unexpected
checkout, or `Editable project location` on a branch other than the pinned
one. Report it at checkpoint 2 and offer a new pyenv named with the ticket
key appended (`superdesk-stt-release-3-6-KEY-123`) rather than repointing
the shared one.

### Elasticsearch exits immediately
Usually `vm.max_map_count` too low or the mounted data directory owned by
another uid. Show `docker logs sd-KEY-elastic | tail -20` and the one-line
fix (`sudo sysctl -w vm.max_map_count=262144`); the sysctl needs the user to
run it.

### Tests fail on fixture setup before any test runs
Almost always a version pin, not a broken test: pytest 9 or pytest-asyncio 1.x
in an env whose `dev-requirements.txt` wants 8.x / 0.26. Compare
`$PY -m pip list | grep -i pytest` against the pins and reinstall the pinned
versions.

## Common pitfalls, avoid these without being told

- **Never `docker compose up` from an app directory** for this workflow. It
  binds the default ports under compose-managed names and collides with the
  ticket containers; the whole point of standalone containers is that they
  can be stopped per ticket.
- **Never run `app:initialize_data` against a reused data directory.** It
  reloads vocabularies and content types over the developer's data.
- **Never edit `package-lock.json` or `requirements.txt` in the app repo** as
  part of setup. Local typescript hoisting is a `package.json` devDependency
  edit that stays uncommitted; say so in the hand-off.
- **Never run two server test suites at once** on this machine.
- **Never create the ticket branch.** Stop at the pinned release branch.
- **Never `black .` or `npm run lint -- --fix` across a repo.** Only the files
  the ticket work will touch, and not during setup at all.
- **Do not trust the pyenv `.python-version` files** in core repos; several
  name envs that no longer exist. Use the full interpreter path.
- **Do not paste raw stack traces at the user** for infrastructure failures.
  One line on what failed and the command that fixes it.

## When to ask vs when to proceed

- Which app or which layer, when the ticket text supports both → ask at
  checkpoint 1.
- A dirty or diverged core checkout → ask, never touch.
- A default port held by another container → ask: stop it or offset.
- A missing Python interpreter for the resolved version → ask before
  `pyenv install`; it takes several minutes.
- Which test file to run first → do not ask, pick the closest by name to the
  ticket's feature and say which you picked.
- A build error listed under step 6 → fix it, say what you did.
- A build error not listed there → stop, show the last 20 lines, ask.
