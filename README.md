# Real-provider diff validation (opt-in)

This procedure uses dedicated disposable repositories on **both Forgejo and
GitHub**, the production application adapters/HTTP endpoints, and actual Git
objects as the content oracle. It never invokes an agent client. Ordinary tests
do not run it. The accompanying offline tests validate the tooling, not provider
compatibility. A successful endpoint probe alone does not complete this procedure.

No live run is recorded by this change. Provider versions, fork accessibility,
update races and assistive-technology behavior remain unverified until an operator
fills in [the report](report-template.md). Do not mark missing checks as passes.
The [local validation status](validation-status.md) records which offline checks
were run and which live evidence is still missing.

## Isolation and prerequisites

- Use an operator-authorized test account and a new empty, disposable repository
  per provider/run, without Actions workflows, hooks that launch jobs, or workers.
  Do not use the application source repository. Record who owns cleanup, including
  any fork, before creating resources. Close/delete resources even after failure.
- Use Node 24.4+, Git, installed project dependencies, and local Chromium (the
  same discovery rules as `npm run test:browser:diff`). Run browsers headlessly.
  An actual screen-reader check requires a separate operator and supported AT.
- Keep existing account credentials in the established Git credential helper or
  SSH agent and application connector configuration/secret storage. The operator
  configures the test connectors through Settings. Never put tokens in URLs,
  command arguments, fixtures, screenshots, reports, or logs. Do not inspect or
  export the connector database, credential files, environment, HARs, or headers.
- Use an isolated app data directory, loopback binding, and no registered worker.
  Configure only the disposable account/repository, and close other app tabs.
  Do not copy a production database. Start from a clean shell without production
  public URL/admin configuration; the probes require local HTTP without login.
- Run providers sequentially, with one probe/browser at a time. Inspect the
  account's available quota privately first; leave normal account capacity free.
  The probe caps itself at 60 application reads. The observer caps **all reads to
  the chosen repository** at 200 per server lifetime and pauses after HTTP 429 or
  a reported remaining quota below 100. This is a safety ceiling, not a target.
  It does not budget repository discovery, Git transport, or other consumers of
  the account's quota. Stop at throttling; do not restart to evade a pause, loop
  retries, or deliberately provoke quota exhaustion.

## Generate and install the same Git fixture

From the source checkout, record these non-secret commands/results:

```sh
git rev-parse HEAD
node --version
git --version
npm ci
npm run build
fixture_directory=$(node test/diff-live-fixture.ts)
cat "$fixture_directory/manifest.json"
```

The generator creates a temporary **bare** repository and manifest, with no remote
or checkout, and prints only its directory. The manifest contains full commit IDs.
It creates no forge resources. Keep this directory for both providers. Its graph is:

```text
                      main (target) --- target-updated
                     /
fixture-base --------
                     \
                      review (head) --- review-updated
```

`main` and `review` deliberately disagree: target-only changes at lines 10/20 must
never appear in context read from the merge base. Review changes are at lines
60/180; the head update adds a sentinel at line 120. The renamed file has two
hunks and enough space for overlapping expansions. The fixture includes changed
and pure renames, spaces/UTF-8 paths, added/deleted/empty files, missing final
newlines, and NUL-containing non-image binary data. This explicitly exercises the
historical #270/#271 scenarios without confusing them with image previews (#269).

Git may pair the two empty paths as a rename. Check its no-hunk control as well
as the nonempty pure rename: expanding must reveal the complete unchanged text
(or an empty boundary) without altering the file's old/new identities. The browser
driver automates the changed rename; record the pure/empty control checks separately.

The unchanged `bytes-*` and `lines-*` files exercise below/exactly/above 1,000,000
bytes and 100,000 lines **through the context endpoint**. They do not inflate the
PR patch or browser row count. The browser's separate 6,000-row and 12 MB budgets
remain covered by deterministic tests; do not create enormous live PRs for them.

For each provider, create the empty repository in its UI, then use its SSH URL
with your already-configured Git authentication. These are operator-only mutations
of disposable fixtures; never execute them in the source worktree or change its
remotes. Replace `disposable_remote` with the test repository's credential-free
SSH URL. Review it before pushing:

```sh
git -C "$fixture_directory/repository.git" push "$disposable_remote" \
  refs/heads/main:refs/heads/main refs/heads/review:refs/heads/review
```

Create an open PR from `review` to `main` in the forge UI. Record its URL/number,
base/head repository identities, API endpoint and provider version from the
instance's version/About information. For hosted GitHub, record `github.com
(hosted; server version not exposed)`, the date, API endpoint and observed API
version header if present; do not invent a server version. Record self-hosted
Forgejo's full version and any reverse-proxy URL subpath.

## Start the actual application with safe upstream counters

Use a fresh data directory per provider. Supply the API repository prefix ending
in `/repos/OWNER/REPOSITORY/`, including a self-hosted API subpath if applicable.
These values identify only the disposable repository and contain no credentials:

```sh
app_data=$(mktemp -d /tmp/git-my-agent-diff-app.XXXXXX)
GIT_MY_AGENT_DATA_DIRECTORY="$app_data" GIT_MY_AGENT_PORT=4183 \
  node test/diff-live-server.ts --run-live "$api_repository_prefix"
```

The wrapper starts `src/main.ts`, observing the real transport used by the
adapters. It keeps normal caching, timeouts, errors, request pacing and quota
handling. Its `live-forge-request` JSON records contain only a sequence, operation,
status and numeric remaining quota when reported. They contain no request URL,
path, body, token, authorization header, or provider error body. Copy only these
records into evidence; do not upload a full process log or private app data.
The wrapper is test tooling and adds no production instrumentation.

In Settings, the operator connects the test account and selects the disposable
repository. Do not enroll a worker or press any agent action. Get the connector
and repository IDs from the app's PR route. Set `app_diff_url` to:

```text
http://127.0.0.1:4183/api/connectors/CONNECTOR_ID/repositories/REPOSITORY_ID/change-requests/PR_NUMBER/diff
```

## Endpoint, content and reuse checks

Close the PR view before the first probe so unrelated UI reads cannot warm caches.
Record the upstream counter immediately before/after each command:

```sh
node test/diff-live-check.ts --run-live "$fixture_directory" "$app_diff_url" initial
node test/diff-live-browser.ts --run-live "$app_diff_url"
```

The endpoint check requires the real patch to parse completely and its old/head/
target IDs to match the manifest. It compares changed lines, paths, absent sides,
binary markers, line numbers and newline markers against `git diff` at the merge
base. It checks whole file sides against `git show`, including byte/line counts,
empty and unterminated text, byte/line boundaries, binary and missing-file results.
Wrong paths at the opposite revision must return `not-found`. It does not accept
silently missing `comparison`, capped text labelled complete, or target-as-old.

The production `DiffController` performs overlapping expansions against the real
HTTP endpoint: exactly two cold application context requests, then zero extra
requests for warm expansion. Its returned content must actually merge and reveal
lines. The initial snapshot is the only read before expansion. The separate
headless browser check opens the actual app route, activates expansion with Enter,
checks retained content/place through Comments/Diff and resizes at 390/820/1200px,
then expands by touch at 390px. It requires one application diff request, two
cold file-side reads, zero warm reads, focus retention, touch target sizes, and no
page-wide horizontal overflow. After the final desktop-to-mobile resize, it waits
for settled geometry and a center hit on the expansion control, then verifies
the targets of actual trusted touch events and requires more code rows. Geometry
settling is bounded to three seconds; it does not retry touch activation or
disable position preservation. It does not mock API responses or certify speech.

Keep **application** and **upstream** counts separate in the report:

| Operation, with cold process cache                                 | Expected upstream calls                                              |
| ------------------------------------------------------------------ | -------------------------------------------------------------------- |
| GitHub snapshot                                                    | 1 pull metadata + 1 merge-base comparison + 1 fixed comparison patch |
| Same GitHub snapshot again                                         | 1 metadata + 1 fixed comparison patch; no new merge-base read        |
| Stable Forgejo snapshot                                            | 2 metadata + 1 PR diff                                               |
| First expansion of renamed file                                    | 2 file-side reads, at old/new full SHAs                              |
| Overlapping/warm expansion, tabs                                   | 0 additional file-side reads                                         |
| Re-reading successful sides within ten minutes in the same process | 0 additional file-side reads                                         |
| Missing/unsupported side retry                                     | New read; these outcomes are not cached                              |

These are expectations to measure, not live observations. UI startup may fetch
other PR metadata/checks, counted as metadata/other. Attribute deltas within the
probe's interval, not the entire session. Repeat the endpoint probe **once** in the
same process to establish successful upstream reuse (and repeated missing reads).
If isolating snapshot counts, request just `app_diff_url` twice while no page is
open. Retain only sanitized counters and probe JSON, not content payloads. A probe
failure exits nonzero and intentionally omits assertion values/error bodies; inspect
the local response privately to locate the failed scenario and record a sanitized
gap. Browser touch failures additionally report bounded numeric geometry/scroll
samples, hit categories and event booleans; these exclude DOM text/attributes,
URLs and response bodies. Do not change expected values or substitute a mock to
obtain a pass.

## Head/target updates during review

Keep one real-provider diff mounted after expansion. Record displayed snapshot
SHAs, revealed line numbers and scroll anchor. Use a headless browser-driver REPL
(`node --experimental-repl-await`, then `await import('./test/browser-driver.ts')`)
if controlling the browser interactively; `openBrowser(url)` always runs headless.
Use `evaluate` to inspect text/geometry and `send` for CDP input, as in
`test/diff-live-browser.ts`. Record commands and observations in the report.

1. Advance only the disposable PR's head with this fast-forward:

   ```sh
   git -C "$fixture_directory/repository.git" push "$disposable_remote" \
     refs/heads/review-updated:refs/heads/review
   ```

   Expand an unrevealed gap before reload: requests must still use the displayed
   old/new SHAs and must **not** display `UPDATED HEAD line 120`. Switch tabs and
   verify retained rows/place. The app does not poll just for diff changes; use
   its ordinary refresh/check activity to observe a retained-snapshot notice, then
   **Reload diff**. Reload must produce `updatedHead` and allow the sentinel to
   appear only in that new snapshot. Run the endpoint probe with stage `head`.

2. Repeat while advancing the disposable target:

   ```sh
   git -C "$fixture_directory/repository.git" push "$disposable_remote" \
     refs/heads/target-updated:refs/heads/main
   node test/diff-live-check.ts --run-live "$fixture_directory" "$app_diff_url" target
   ```

   Retained expansions must still use `base`/`updatedHead`. Reload must report
   `updatedTarget`, retain `base` as merge base, and exclude both target-only
   sentinels from context. Record any delay before the forge updates PR metadata;
   retry a reload once after it settles instead of polling in a loop.

3. To test a late context response, on a fresh mount install a one-shot wrapper
   around **browser** `fetch` that consumes the real `/diff-context` JSON and
   holds its delivery behind a locally released promise. Record that the provider
   response was real and only its delivery was delayed. Start expansion, click
   Reload diff while held, wait for the replacement snapshot, then release the
   old response within the 30-second context deadline. Compare row identities,
   ranges and snapshot labels before/after release. Old text must never populate
   the replacement. Capture the same case with the head update above between
   expansion and reload. This is controlled delivery of live data, not a claim
   about naturally occurring provider latency. Without controllable delivery,
   mark the live race **not run** and cite the deterministic disposal test.

4. Where the operator can pause the server's real transport (for example a local
   Node inspector breakpoint in `countedLiveFetch` immediately after the first
   `pull-metadata` response arrives), advance head or target before releasing
   that real metadata to the adapter. Do this only once per update type, within
   the adapter's timeout, using fresh disposable PRs if the earlier refs moved.
   GitHub should return a coherent comparison at the commits it first read;
   subsequent reload should show the update. Forgejo should detect differing
   before/after metadata and retry to a coherent snapshot (at most six upstream
   calls), or provide a useful 409/timeout and allow reload. Verify that no patch
   is labelled with mismatched SHAs. Record the exact pause point, update and
   result. If timing cannot be controlled, record **not run**, never an inferred
   pass. Repeated mutation/429/retry exhaustion remains simulated coverage.

For step 3, this headless REPL recipe holds actual responses without inventing
content. Start a fresh PR view for each run. In `node --experimental-repl-await`:

```js
const { openBrowser, waitUntil } = await import('./test/browser-driver.ts');
const b = await openBrowser(
  'http://127.0.0.1:4183/repositories/CONNECTOR_ID/REPOSITORY_ID/change-requests/PR_NUMBER#diff',
);
await waitUntil(() => b.evaluate("!!document.querySelector('.diff-expand')"), Boolean);
await b.evaluate(`(() => {
  const realFetch = window.fetch;
  window.liveHeld = [];
  window.liveDelivered = 0;
  window.liveSides = [];
  window.fetch = async function(input, init) {
    const url = new URL(input instanceof Request ? input.url : String(input), location.href);
    const hold = url.pathname.endsWith('/diff-context');
    const response = await realFetch.call(this, input, init);
    if (hold) {
      const readJson = response.json.bind(response);
      response.json = async () => {
        const data = await readJson();
        window.liveSides.push({ path: url.searchParams.get('path'), revision: url.searchParams.get('revision') });
        await new Promise(resolve => window.liveHeld.push(resolve));
        window.liveDelivered++;
        return data;
      };
    }
    return response;
  };
  const file = Array.from(document.querySelectorAll('.diff-file'))
    .find(node => node.querySelector('.diff-file-heading').textContent.includes('new name 日本.txt'));
  file.querySelector('.diff-expand').click();
})()`);
await waitUntil(
  () => b.evaluate('window.liveHeld.length'),
  (count) => count === 2,
);
```

Now perform the head fast-forward from the other terminal, then immediately:

```js
await b.evaluate("document.querySelector('.diff-snapshot button').click()");
await waitUntil(
  () => b.evaluate("document.querySelector('.diff-snapshot button').disabled"),
  (disabled) => !disabled,
);
await b.evaluate(`(() => {
  window.liveReplacement = document.querySelector('.diff-snapshot').textContent;
  window.liveReplacementRows = Array.from(document.querySelectorAll('[data-diff-row]')).map(node => node.dataset.diffRow);
  window.liveHeld.splice(0).forEach(release => release());
})()`);
await waitUntil(
  () => b.evaluate('window.liveDelivered'),
  (count) => count === 2,
);
await b.evaluate(
  'new Promise(resolve => requestAnimationFrame(() => requestAnimationFrame(resolve)))',
);
await b.evaluate(`({
  labelRetained: document.querySelector('.diff-snapshot').textContent === window.liveReplacement,
  rowsUnchanged: JSON.stringify(Array.from(document.querySelectorAll('[data-diff-row]')).map(node => node.dataset.diffRow)) === JSON.stringify(window.liveReplacementRows),
  heldSides: window.liveSides
})`);
await b.close();
```

Require both booleans true, the replacement label to identify the new head, and
held sides to identify the original displayed commits. If any response times out
before release, record that timeout/reload result separately; it is not evidence
of successful late delivery. In all cases close the browser in cleanup. This
controlled delivery experiment supplements the retained-context check in step 1;
it does not replace it.

## Forks and provider-specific capabilities

Create a disposable fork of each still-initial test repository where supported
by the test account/instance. Put the fixture's `review`/`review-updated` refs
**only in the fork**, and target upstream `main`. For a meaningful fork-head
test, use a fresh upstream containing only `main`; do not leave the same head
object reachable through an upstream branch or earlier same-repository PR.
Configure the **base repository** in the app. Repeat the initial endpoint and
browser commands using that PR, then the head update. Old and new raw sides must
be readable through the base-repository adapter with identical manifest SHAs.
Record fork/base owner identities, permissions and counts. If the service disables
forks or the account cannot create/access one, record that capability as
**unsupported in this setup**, including the reason; same-repository success is
not fork evidence. A 404 for an otherwise supported fork head is a compatibility
gap requiring independent tracking.

The main fixture does not assert symlink/submodule parity. In a separate small
disposable PR, commit a symlink to a text file and request its old/new context at
full commits. Record whether GitHub/Forgejo returns link text, target contents or
unsupported metadata. If it returns target text, verify that the viewer refuses
to merge it against a symlink patch; never claim the target is valid link context.
Directory/submodule metadata should be explicitly unavailable/unsupported where
applicable. Record the exact observed capability; do not reinterpret an error as
an empty file. If not exercised, leave these assumptions as gaps in the report.

## Announcements and failure/reload recovery

Record an actual operator screen-reader session on real-provider data separately
from headless DOM checks: OS, browser, AT/version, input method, exact announced
loading/failure/completion text, focus destination, and pass/fail per transition.
Use a cold expansion for loading. For safe failure, temporarily block just the
local app context request or disconnect the test browser; label that failure as
**locally induced**, not a provider failure. Restore connectivity and retry: the
successful side should remain reusable and focus/anchor retained. Check a failed
Reload diff retains the previous content and a later successful reload replaces
it. Verify the completion announcement after rows appear, not merely the presence
of `role=status`. Missing announcements are gaps, not passes. This requires an
AT-capable operator environment; headless accessibility trees do not replace it.

Do not provoke live malformed payloads, partial streams, missing commit metadata,
wrong identities, secondary rate limits or quota exhaustion. Keep running the
deterministic parser/controller/adapter/application/cache/HTTP/browser coverage:

```sh
npm run build
node --test test/diff-model.test.ts test/diff-controller.test.ts \
  test/change-request-diff.test.ts test/change-request-context.test.ts \
  test/immutable-read-cache.test.ts test/http.test.ts test/diff-live.test.ts
npm run check
npm run format:check
npm run test:browser:diff
node test/diff-live-browser-selftest.ts
node test/diff-live-touch-browser-selftest.ts
```

These are simulated assertions (including the new probe's offline self-test),
not live observations. Keep their results in a separate report section.

## Report, defects and cleanup

Complete [report-template.md](report-template.md) for both providers. Include UTC
times, exact application source revision and clean/dirty state, fixture manifest,
versions/endpoints, setup/commands, per-scenario outcomes, safe request counts,
unsupported capabilities, unrun checks and cleanup confirmation. Do not commit
unsanitized generated output. A result applies only to its recorded versions,
permissions and source revision. The check does not certify other deployments.

Track each newly observed compatibility defect independently, with a minimal
sanitized reproduction, expected/actual behavior and source/provider revisions.
Link an existing matching defect (#270/#271 only if still applicable) or create a
new issue through the project's issue workflow. Include a task-specific worker
memory/CPU/GPU and agent/model/reasoning recommendation with rationale. Do not
hide defects with mocks, expand this verification task into source fixes, or mark
the overall validation complete while required evidence is missing.

In a finally/cleanup step owned by the operator: stop browsers and app (Ctrl-C),
close PRs, delete all disposable forks **before** upstream repositories through
the forge UI, and confirm their absence. Remove any disposable credential grant
if one was created. Delete only the generated temporary fixture/app directories
after checking their printed paths; the app data contains encrypted connector
secrets and is never an artifact. Record resource IDs, cleanup times and failures
in the report; retry ambiguous deletion only after checking the forge. The probes
are read-only and cannot clean resources that an operator created out of band.
