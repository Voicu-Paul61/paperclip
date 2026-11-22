# Hermes + Paperclip: bugs found and fixed

This fork carries fixes I made so that Paperclip agents can run on
[Hermes Agent](https://github.com/NousResearch/hermes-agent) with model routing
across OpenAI Codex and Anthropic Claude, without waiting for upstream releases.

Tested with Paperclip 2026.1117.0 (and `master` at `be08705f`), Hermes Agent
0.23.3, macOS, local trusted deployment.

## The goal

One Paperclip agent backed by local Hermes. Hermes picks the model for each step
from a routing table (`delegation.model_map` in `~/.hermes/config.yaml`):
cheap Codex models for simple steps, Claude Sonnet for everyday work, Opus or
Fable for hard work. Paperclip only sees one agent.

## What was broken

| # | Symptom | Root cause | Status |
|---|---------|------------|--------|
| 1 | Every agent reply is the agent's own prompt (`Query: You are "…"`), cut at 2000 characters. The real answer is lost. | The adapter passes `-Q` only when `quiet` is explicitly `true`, but the settings screen shows it as on by default. Non-quiet Hermes 0.23 output starts with a `Query:` echo of the whole prompt, and that echo becomes the run result. | Fixed: [`fix/hermes-quiet-default`](../../tree/fix/hermes-quiet-default). Upstream issue [#11976](https://github.com/paperclipai/paperclip/issues/11976). Two earlier fix PRs ([#12016](https://github.com/paperclipai/paperclip/pull/12016), [#16597](https://github.com/paperclipai/paperclip/pull/16597)) were closed without merging. |
| 2 | Every new Hermes agent is stuck in "verifying" with `harness_test_failed`. | Lifecycle verification needs a passed hello probe or a clean "pass". The Hermes environment test ran its hello probe only for managed AI routing, and it always warns `hermes_no_api_keys` because Hermes keeps credentials in its own profile. Result: "warn", no probe, never verified. | Fixed: [`fix/hermes-local-hello-probe`](../../tree/fix/hermes-local-hello-probe). |
| 3 | Tasks end in "blocked" after two retries even though Hermes did the work. The agent says it "couldn't verify the Paperclip title or status update". | Hermes runs tools inside its gateway daemon. The daemon drops any environment variable whose name contains `KEY`, `TOKEN`, `AUTH` and similar, by design. `PAPERCLIP_API_KEY` never reaches the agent's `curl`/Python, so it can't comment on or close the issue. | Worked around: [`integrations/hermes-local-fixes/paperclip-hermes-routed`](integrations/hermes-local-fixes/paperclip-hermes-routed) hands the per-run key over as a private file (mode 0600, deleted after the run). A proper fix needs the adapter and Hermes to agree on a credential channel, so this is not an upstream PR yet. |
| 4 | Clicking "open in inbox" on a task page does nothing visible. | In this version, every `/inbox/*` route redirects to the Tasks list. The new inbox lives at `/decisions`. | Documented only. |

## How each was found

1. Read the raw run log (`heartbeat-runs/<id>/log`): Hermes did produce an answer,
   but the stored result started with `Query:`. Running `hermes chat -q … --quiet`
   by hand printed only the answer and `session_id:`.
2. Read the server log for the lifecycle failure, then
   `server/src/services/agent-harness-verification.ts` for the pass rule, then
   `packages/adapters/hermes/src/server/test.ts` for when the probe runs.
3. Found `KeyError: 'PAPERCLIP_API_URL'` in Hermes' session store. Confirmed the
   cause by forwarding a non-secret name through `terminal.client_env_passthrough`
   (it arrived) while `PAPERCLIP_API_KEY` stayed empty. Hermes' own code
   (`tools/code_execution_env.py`, `agent/session_client_env.py`) documents the rule.

## Verification

- Adapter test suite (`pnpm --filter @paperclipai/hermes-paperclip-adapter test`)
  passes on every branch: 177 tests on `fix/hermes-quiet-default`, 178 on
  `fix/hermes-local-hello-probe`, 181 on `hermes-fixes`. `tsc --noEmit` passes.
  Lint was not run locally; upstream CI runs it.
- Each fix has a test that failed before the change (red) and passes after (green).
- Live: the changed environment test against a real Hermes 0.23.3 returns
  `hermes_hello_probe_passed` in about 4 seconds.
- Live: a Hermes-backed agent took a real writing task, posted the answer as a
  clean comment, and marked the issue done with an API call that read the key
  from `PAPERCLIP_RUN_ACCESS_FILE`. No key files were left after the run.
- Routing: Hermes' `model-routing` skill delegates only multi-part work, so a
  small task stays on the default model. Earlier multi-step runs did route steps
  to `codex-luna` and `claude-sonnet`. The skill climbs the routes in list
  order, so keep `delegation.backends.model_map.options` ordered from cheapest
  to strongest.

## Using the workaround (#3)

1. Copy `integrations/hermes-local-fixes/paperclip-hermes-routed` somewhere on
   your machine and make it executable.
2. Add the `terminal.client_env_passthrough` list from the script header to
   `~/.hermes/config.yaml`.
3. In Paperclip, set the agent's Hermes command to the script path and turn
   "Quiet output" on.
