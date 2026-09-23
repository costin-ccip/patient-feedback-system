# Working in this repo

This repo tracks the patient feedback system for Cape Clarity, built with GitHub
Spec Kit (see `.specify/memory/constitution.md` and `README.md`). It covers six
milestones (M0-M5); requirements for all of them live in one spec:
`specs/001-feedback-collection-pipeline/spec.md`.

## Milestone implementation notes

Each milestone that has been built also gets its own implementation-notes file,
e.g. `specs/001-feedback-collection-pipeline/m0-implementation-notes.md` for M0.
Unlike spec.md (requirements, all milestones), these files are per-milestone,
as-built documentation: exact component names/IDs, verbatim code and formulas,
data contracts, known gotchas and workarounds, and access constraints that shaped
the build.

**Convention: if you (human or AI) change how a milestone actually works —
a Zoho Flow function, a CRM field, an Analytics formula, a dashboard panel, a
data contract — update that milestone's implementation-notes file in the same
session, before moving on.** Treat it the same as updating a code comment next to
code you just changed: not a separate task to remember later, part of finishing
the change. If a milestone doesn't have a notes file yet and you're building it,
create one.

## Process note

The constitution's Rollout Workflow section requires new milestone/dashboard work
to go through specify -> plan -> tasks rather than being configured ad hoc
directly in Zoho Flow. M0 was built ad hoc, without plan.md/tasks.md, before this
convention file existed — see the "Process note" section at the end of
`m0-implementation-notes.md`. Future milestones should follow the full spec-kit
workflow where practical.

## Token issuance convention (no subflows) — added 2026-09-23

Zoho Flow is on the Standard plan, which has no subflows (Built-ins → Logic offers
only Set Variable / Decision / Delay / If else). Every milestone trigger flow issues
its feedback request with two steps:

1. the shared custom function **`issueFeedbackToken`** (supersede old Issued
   instances + token/expiry + Milestone_Instances create), output variable renamed to
   `issueFeedbackToken_1`; then
2. its own **Zoho Mail "Send email"** step (Apps → Zoho Mail, connection "Connection
   to info@capeclarity.com"; not Built-ins → Notification → Send Email, which is Zoho
   Flow's own mailer), with the link `<survey_url>?token=${issueFeedbackToken_1.token}`.

M4/M5 must follow this. Don't add a subflow, and don't copy issuance logic into a new
function. Editing `issueFeedbackToken` changes every milestone at once, so treat it
like shared infrastructure and update
`specs/002-remove-subflow-dependency/contracts/issue-feedback-token.md` and that
feature's implementation notes in the same session. Details and builder gotchas
(especially: a node dropped on an If-else branch is NOT wired until you draw the
connection; check `jsplumb-connected` on endpoints) are in
`specs/002-remove-subflow-dependency/implementation-notes.md`. `[RETIRED] Subflow -
Issue Feedback Token` stays OFF and is kept for audit only.

## Pushing to GitHub (multiple builders, not all sessions push the same way)

This project is built by more than one Claude session/account over time.
**Not every session has the same git access** — that's an
environment/credential detail of the specific session, not something about
this repo.

**Environment check first** — figure out which of these you're running in:

- **A terminal with the user's own environment already set up** (Claude Code
  CLI on Costin's actual machine, a normal shell): `git`/`gh` most likely
  already work with his real credentials. Check `gh auth status` and proceed
  normally — none of the below is needed.
- **Cowork's device-bridge sandbox** (`mcp__remote-devices__device_bash`
  tools present): this is the standard path now (see below) — the repo lives
  in a connected folder on Costin's Mac at
  `Documents/Cape Clarity/Specs/patient-feedback-system`, and pushes go out
  through his machine's own network.
- **Cowork's cloud container only** (`Bash` tool, no device bridge available,
  or the device isn't connected): use the browser-upload fallback further
  down.

Confirmed (2026-09-21): from inside the cloud container, ALL GitHub traffic —
not just `git push`, but plain reads of unrelated repos, and even GitHub's
own OAuth device-code endpoint — is intercepted by an Anthropic-side proxy
that only allows repo-scoped, read-level calls for repos this session is
explicitly bound to. This session's binding for
`costin-ccip/patient-feedback-system` allows read (`git fetch`/`clone` work
fine) but not push, and there is no tool available in a Cowork session to
expand that (the proxy's own error names an `add_repo` tool, but it belongs
to a different execution surface — Claude Code's GitHub Actions integration —
not Cowork). **Don't try to route around this with a personal token or
`gh auth login` from inside the cloud container — it will hit the same
wall.** This isn't a fixable git-config problem; move the work to the
device-bridge sandbox instead (below), or fall back to the browser-upload
method.

### Standard method: device-bridge sandbox + `gh`

The repo lives in a connected folder on Costin's Mac:
`Documents/Cape Clarity/Specs/patient-feedback-system` (device path — reached
via `device_bash` at `$HOME/mnt/Specs/patient-feedback-system` once that
folder is connected; request it with `device_request_folder_access` if it
isn't). Do the actual spec-kit editing wherever is convenient (the cloud
container is fine for drafting), but land the files here before committing,
and do all `git`/`gh` operations from `device_bash`.

By explicit decision, **credentials are never persisted to disk beyond the
session** — re-authenticate every time rather than caching a token in the
connected folder. It only takes a minute.

1. **Check what's already there first** (don't assume it's missing):
   ```bash
   export PATH="$HOME/.local/bin:$PATH"
   which git gh; gh auth status
   ```

2. **If `gh` isn't installed** — no `sudo` in this sandbox, so install into
   the user's own space:
   ```bash
   mkdir -p "$HOME/.local/bin" "$HOME/.local/opt"
   cd "$HOME/.local/opt"
   VER=$(curl -sSI https://github.com/cli/cli/releases/latest | grep -i '^location:' | sed 's#.*/tag/v##' | tr -d '\r')
   ARCH=$(uname -m); ARCH=${ARCH/aarch64/arm64}; ARCH=${ARCH/x86_64/amd64}
   curl -sSL -o gh.tar.gz "https://github.com/cli/cli/releases/download/v${VER}/gh_${VER}_linux_${ARCH}.tar.gz"
   tar -xzf gh.tar.gz
   ln -sf "$HOME/.local/opt/gh_${VER}_linux_${ARCH}/bin/gh" "$HOME/.local/bin/gh"
   ```

3. **If not authenticated** — use the two-step device-code flow, not
   `gh auth login --web` directly (that blocks in the foreground, and
   background processes don't survive between separate `device_bash` calls):
   ```bash
   # Step 1 — request a code (its own call; do this, then wait for Costin)
   curl -sS -X POST https://github.com/login/device/code \
     -H "Accept: application/json" \
     -d "client_id=178c6fc778ccc68e1d6a" \
     -d "scope=repo read:org gist workflow"
   # -> {"device_code": "...", "user_code": "XXXX-XXXX", "verification_uri": "https://github.com/login/device", ...}
   ```
   Show Costin `user_code` and `verification_uri`. **Never** ask him to paste
   a token into chat — this flow needs only the code, entered on github.com
   itself, on any device. Wait for him to confirm he's approved it, then:
   ```bash
   # Step 2 — exchange for a token (separate call, after he confirms)
   curl -sS -X POST https://github.com/login/oauth/access_token \
     -H "Accept: application/json" \
     -d "client_id=178c6fc778ccc68e1d6a" \
     -d "device_code=<DEVICE_CODE_FROM_STEP_1>" \
     -d "grant_type=urn:ietf:params:oauth:grant-type:device_code"
   # -> {"access_token": "gho_..."} or {"error": "authorization_pending"} if not done yet
   export PATH="$HOME/.local/bin:$PATH"
   echo "<ACCESS_TOKEN>" | gh auth login --hostname github.com --with-token
   gh auth setup-git
   ```
   The client ID above is GitHub CLI's own public OAuth client ID (not a
   secret — it's compiled into the open-source `gh` binary; using it just
   replicates what `gh auth login --web` does internally). Never print the
   raw token value in anything shown to Costin.

4. **Set git identity** (once per session, since global config doesn't
   survive either):
   ```bash
   git config --global user.name "Costin"
   git config --global user.email "liana.preudhomme@capeclarity.com"
   ```

5. **Deleting files in the connected folder** (git needs this for its own
   lock files, even just to `clone` or `commit`) is off by default. If a git
   command fails with `Operation not permitted` on a `.git/*.lock` file, call
   `device_request_delete_permission` for that folder before retrying — don't
   work around it another way.

6. Push normally from there: `git add` / `git commit` / `git push origin
   main`. Never force-push, never skip hooks.

Verified working 2026-09-21: bootstrapped `gh` fresh in the device-bridge
sandbox, authenticated as `costin-ccip` via device code, cloned the repo into
the connected folder, and pushed this very documentation update through it.

### Fallback: browser-upload (cloud-container-only sessions)

If there's no device bridge available (or Costin hasn't connected a folder
yet) and you're stuck in the cloud container, use the browser to push via
GitHub's web upload flow instead:

- Commit locally as normal first (`git commit`), so there's a clean local
  commit to match against afterward.
- Navigate the browser to
  `https://github.com/costin-ccip/patient-feedback-system/upload/main/<dir>`
  (the directory containing the changed files).
- For each changed file, build it as a JS `File` object with its exact
  content, wrap it in a `DataTransfer`, assign that to the page's
  `input[type="file"]`, and dispatch a `change` event — do this once per
  file (the native input's `.files` gets reset after each event, so stage
  files one at a time from fresh `DataTransfer` objects, not by trying to
  recombine a previously staged one). Verify each staged file's byte size
  against the real on-disk file before moving on.
- Fill in the commit summary/description via the native property setter
  (`Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,
  'value').set.call(el, value)`, then dispatch `input`/`change`) — plain
  simulated typing doesn't reliably land in these fields on this page.
- Confirm "Commit directly to the `main` branch" is selected, then commit.
- Verify the push by fetching each file's raw content from
  `raw.githubusercontent.com/<org>/<repo>/<new-commit-sha>/<path>` and
  comparing byte counts to the local files.
- Reconcile local git: `git fetch origin`, confirm
  `git diff HEAD origin/main --stat` is empty (the browser-made commit
  will have a different SHA/author than your local one but identical
  content), then `git reset --hard origin/main` to bring local `main` in
  line with the new remote commit.

Whichever way the push happens, never force-push and never skip hooks to get
around a block.

## Access constraints

Do not open the CRM's Leads or Patients modules in the browser without explicit
permission from Costin — use MCP tool calls (getRecords/createRecords/etc.) for
any CRM module that doesn't carry PII instead.


## Branch reconciliation (2026-09-06)

A separate session had already built this repo's real history on GitHub
(what is now `main`: constitution v1.2.1, a materially different spec.md —
richer Milestones table, Clinical Safety Flag Rules / User Story 4, stricter
zero-contractor-dashboard-access framing, Principle VI on testimonial/
marketing use). A later session (this one, in an earlier part of the same
conversation) instead initialized a fresh, disconnected local repo from
scratch — constitution v1.0.1, a differently-worded and less complete spec —
and built the M0 backfill (`m0-plan.md`/`m0-tasks.md`) and all of the M1
planning (`m1-plan.md`/`m1-research.md`/`m1-data-model.md`) against that
divergent, non-authoritative version, never knowing the real one existed on
GitHub until Costin asked for a staleness check before the first push.

Costin confirmed the GitHub branch is authoritative. The two histories share
no common commit, so reconciliation was: keep the GitHub history as `main`,
preserve the disconnected local work under the branch
`main-orphaned-v1-nonauthoritative` (not deleted, for audit purposes only —
not for reuse), and re-apply the M0/M1 planning documents as new commits on
top of the authoritative `main`, corrected wherever they conflicted with or
mis-cited the real constitution/spec. See the "Constitution Check" sections
of `m0-plan.md` and `m1-plan.md` for exactly what was corrected, and
`m1-data-model.md` / `m0-implementation-notes.md` for the `5A` dead-CRM-config
correction.

**Lesson for future sessions**: before doing any spec-kit work in a repo that
has a remote configured, run `git fetch` and diff local history against the
remote's actual branches before trusting what's in the local working
directory or assuming a local git init reflects what's already on GitHub.
