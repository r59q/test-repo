# Revision-pinned diff live validation report

Status: **NOT RUN**. Copy this template; do not replace blanks with inferred passes.

- UTC start/end:
- Operator and cleanup owner:
- Source `git rev-parse HEAD`; `git status --short` (clean/dirty; describe changes):
- Node/Git/Chromium versions; OS:
- Fixture manifest (five full SHAs, no private configuration):
- Commands executed, exit statuses, sanitized artifact references:
- Agent subscription invocations: none / explain unexpected activity:

| Setup                                                         | Forgejo      | GitHub       |
| ------------------------------------------------------------- | ------------ | ------------ |
| Instance/web/API endpoint                                     | Not recorded | Not recorded |
| Server version (hosted GitHub may not expose it), API version | Not recorded | Not recorded |
| Disposable base repo/PR and fork identities                   | Not recorded | Not recorded |
| Granted capability descriptions (no tokens)                   | Not recorded | Not recorded |
| Initial quota and reserved margin; rate-limit observations    | Not recorded | Not recorded |

For every row record **pass / fail / unsupported / not run**, observed IDs/results,
command/observation time, and evidence. Separate application reads from upstream
calls; record counts before and after cold/warm operations and elapsed times.

| Live scenario                                                         | Forgejo | GitHub  |
| --------------------------------------------------------------------- | ------- | ------- |
| Diverged target: merge base differs from target; patch matches Git    | Not run | Not run |
| Changed rename, spaces/UTF-8 paths; immutable old/new contents        | Not run | Not run |
| Pure rename/no-hunk controls (#270); absent sides                     | Not run | Not run |
| Added/deleted; empty and no-final-newline text                        | Not run | Not run |
| Binary and unavailable file statuses; no fabricated empty text        | Not run | Not run |
| Byte bounds below/exactly/above 1,000,000                             | Not run | Not run |
| Line bounds below/exactly/above 100,000                               | Not run | Not run |
| Cold overlapping expansion; application/upstream counts               | Not run | Not run |
| Warm expansion/tabs and repeated successful-side reads; counts        | Not run | Not run |
| Head update while reviewing; retained context pinned, reload useful   | Not run | Not run |
| Target update while reviewing; old content remains merge-base data    | Not run | Not run |
| Held real context delivery after snapshot replacement is ignored      | Not run | Not run |
| Head/target update during snapshot loading; pause point, retry/reload | Not run | Not run |
| Fork-only head readable via base repository; update                   | Not run | Not run |
| Symlink raw behavior and mismatched-target rejection                  | Not run | Not run |
| Directory/submodule capability, if exercised                          | Not run | Not run |
| Keyboard/touch expansion; 390/820/1200px content/place retention      | Not run | Not run |
| Locally induced failure/retry and failed reload retain content        | Not run | Not run |

Screen-reader observations (actual AT session, separate from headless assertions):

| Provider / OS / browser / AT version / input | Loading words | Locally induced failure words | Completion words / focus | Result  |
| -------------------------------------------- | ------------- | ----------------------------- | ------------------------ | ------- |
| Not recorded                                 | Not run       | Not run                       | Not run                  | Not run |

Deterministic simulated checks (command, source revision, outcome/count; **not**
live evidence):

- Parser/controller/adapter/application/cache/HTTP:
- Offline probe self-test:
- `npm run check`, `npm run format:check`:
- `npm run test:browser:diff`:

Gaps/unsupported capabilities/defects (independent issue links, impact and required
follow-up; never silently downgrade a failed scenario):

- Not recorded.

Cleanup (every resource, verification time, result; include failed/partial runs):

- Browser/app stopped:
- Fork PRs/forks removed:
- Base PRs/repositories removed and absence confirmed:
- Disposable grants revoked if created:
- Generated local fixture/app data removed; no secrets retained in artifacts:

Overall outcome and limitations: **No provider compatibility claim until this
report has actual observations for both providers and explicitly accounts for
all remaining gaps.**
