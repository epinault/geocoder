# AGENTS.md

`geocoder` is an Elixir library (published to Hex as `geocoder`, currently 2.3.0) that geocodes addresses and reverse-geocodes coordinates through pluggable providers (OpenStreetMaps, Google Maps, OpenCage Data, Fake), with a built-in in-memory cache and a Poolboy worker pool. Consumers add `Geocoder.Supervisor` to their own supervision tree; the library has no `Application` callback, no Ecto, no Phoenix, and no deploy target other than Hex.

## Project Overview

| Item | Value |
|---|---|
| Type | Mix library (Hex package), not an application |
| Elixir / OTP floor | `~> 1.17` / `~> 27` (mix.exs); CI matrix Elixir 1.17-1.20, OTP 27-29 |
| Runtime deps | `towel` (Maybe monad used by Store), `poolboy` (worker pool), `geohash` |
| Optional deps | `httpoison` (`~> 2.1 or ~> 3.0`), `jason` (`~> 1.2`), `jsx` (`~> 2.8 or ~> 3.0`) |
| Dev/test deps | `credo`, `ex_doc`, `hammox` (Mox with typespec checks), `excoveralls` |
| CI | GitHub Actions: `.github/workflows/elixir.yml` (test matrix, lint on last entry), `publish.yml` (Hex publish on any tag) |
| Dependabot | monthly, mix + github-actions |
| Top module | `Geocoder` (`lib/geocoder.ex`); app name `:geocoder` |
| Git remotes | `origin` = `git@github.com:epinault/geocoder` (the FORK); `cyrus` = `https://github.com/CyrusNuevoDia/geocoder` (UPSTREAM). Note `@source_url` in mix.exs still says `CyrusOfEden/geocoder` |

There is no `config/` directory and no `.credo.exs` or `.tool-versions`. All configuration is passed as options to `Geocoder.Supervisor`.

## Essential Commands

There are no mix aliases and no precommit alias. The quality gate is what CI runs.

### Setup
```bash
mix deps.get
```

### Test
```bash
mix test                                   # whole suite
mix test test/geocoder/providers/google_maps_test.exs          # one file
mix test test/geocoder/providers/google_maps_test.exs:12       # one test by line
mix coveralls                              # coverage (forced to :test env via def cli in mix.exs)
# Run the integration-style GeocoderTest against a real provider (default is "fake"):
PROVIDER=google API_KEY=xxx mix test test/geocoder_test.exs    # google | opencagedata | openstreetmaps | fake
```

### Quality (CI gate, mirror before pushing)
```bash
mix format --check-formatted
mix deps.get && mix deps.unlock --check-unused
mix compile --warnings-as-errors
mix test
mix credo --strict      # not run in CI, but credo is a dev/test dep; keep it clean
```

### Docs / Release
```bash
mix docs                # ex_doc; extras: README, CHANGELOG, CODE_OF_CONDUCT, LICENSE
```
Release: see "Hex publishing" under Workflow.

## Architecture

```
lib/
  geocoder.ex                 Public API: call/2, call/1, call_list/2 (dispatches to Worker)
  geocoder/
    supervisor.ex             Geocoder.Supervisor: poolboy child_spec + store_module child
    config.ex                 Geocoder.Config: defaults + merge of all options, typespecs
    worker.ex                 Geocoder.Worker: GenServer in the Poolboy pool; cache-then-provider logic
    store.ex                  Geocoder.Store: GenServer in-memory cache (Towel/Maybe), geohash for reverse lookups
    provider.ex               Geocoder.Provider behaviour (geocode, geocode_list, reverse_geocode, reverse_geocode_list)
    request.ex                Geocoder.Request: thin wrapper that calls config[:http_client].request/2
    http_client.ex            Geocoder.HttpClient behaviour (request/2)
    http_client/httpoison.ex  default client
    http_client/hackney.ex    alternative client; calls :hackney directly
    json_codec.ex             Geocoder.JSONCodec behaviour (encode/decode, bang variants)
    json_codec/jsx.ex         JSX adapter (Jason satisfies the behaviour as is)
    providers/
      open_street_maps.ex     DEFAULT provider, no key needed
      google_maps.ex          needs :key
      open_cage_data.ex       needs :key
      fake.ex                 deterministic provider driven by worker_config[:data]; for tests/dev
    structs/                  Geocoder.Coords, Geocoder.Location, Geocoder.Bounds
test/
  test_helper.exs             ExUnit.start + Hammox.defmock(Geocoder.HttpClientMock, for: Geocoder.HttpClient)
  support/helpers.ex          fixtures: payloads per provider, fake_data_cache/0, provider_test_config/2, assert_* helpers
  support/mocks.ex            compiled only in :test (elixirc_paths)
  geocoder_test.exs           end-to-end through Supervisor + Worker + Store
  geocoder/                   per-module tests (config, request, store, providers/, http_client/, json_codec/)
```

### Request flow
`Geocoder.call/2` -> `Geocoder.Worker.geocode/1` -> `:poolboy.transaction` checks out a Worker -> `GenServer.call` (or `cast` + `Task` when `:stream_to` is given) -> `run/3`:
1. `*_list` functions always go straight to the provider (never cached).
2. `store: true` (default): ask `store_module` first; on `:nothing` fall through to the provider.
3. Provider result is written with `store_module.update/2` and linked to the original query with `link/3` (cast).

Supervision tree (started by the consumer): `Geocoder.Supervisor` (one_for_one) -> [poolboy pool named `:geocoder_workers` (size 4, max_overflow 2, LIFO) of `Geocoder.Worker`, `Geocoder.Store` named `:geocoder_store`].

### Configuration keys (all passed to `Geocoder.Supervisor`, defaults in `Geocoder.Config`)
- `:pool_name` (default `:geocoder_workers`), `:pool_config` (poolboy: `size`, `max_overflow`, `strategy`, `worker_module`; `:name` is forced from `:pool_name`)
- `:store_module` (default `Geocoder.Store`; custom stores must implement the same GenServer-ish API: `geocode/2`, `reverse_geocode/2`, `update/2`, `link/3`, plus child_spec taking `store_config`), `:store_config` (`name: :geocoder_store`, `precision: 6`)
- `:worker_config`: `provider` (default OpenStreetMaps), `key`, `http_client` (default `Geocoder.HttpClient.Httpoison`), `http_client_opts` (default `[recv_timeout: 30_000]`), `json_codec` (default `Jason`), `data` (Fake provider only)
- Per-call options on `Geocoder.call/2`: `:timeout` (5000 ms), `:stream_to`, `:store` (true), `:pool_name`, plus anything overriding `worker_config` for that call.

### HTTP client and how tests stub it
Providers never call an HTTP library directly: they build a request map (`%{method, url, query_params, headers?}`) and call `Geocoder.Request.request/2`, which dispatches to `config[:http_client]`. Tests pass `http_client: Geocoder.HttpClientMock` (a Hammox mock of the behaviour) in the provider config and assert on the exact request map with `expect(:request, fn req, _config -> ... end)`. Hammox checks returned values against the `Geocoder.HttpClient` typespec, so mocks must return `{:ok, %{status_code:, headers:, body:}}` with an already-decoded body (clients decode via `config[:json_codec]`). No network in the default test run.

## Conventions

- `.formatter.exs` is the default (`mix format`, 98-column default line length). No `.credo.exs`, so Credo defaults apply.
- Public modules have `@moduledoc`; behaviours define `@callback` typespecs; `Geocoder.Config` carries `@type`/`@spec`. New public functions should get `@spec`.
- Returns are tagged tuples: `{:ok, %Geocoder.Coords{}}` / `{:error, term}`. The Store returns Towel Maybe values (`{:just, x}` / `:nothing`).
- Providers implement `@behaviour Geocoder.Provider`; they receive `(payload_opts, provider_config)` where `provider_config` is `Keyword.take(worker_config, [:http_client, :http_client_opts, :json_codec, :key, :data])`.
- Optional deps: modules that wrap them guard with `if Code.ensure_loaded?(...)` (see `http_client/hackney.ex`). Keep the library compiling when httpoison/jason/jsx are absent.
- Docs grouping in mix.exs: `Providers` = `~r/^Geocoder.Providers/`; structs listed explicitly. Update `groups_for_modules` if you add new structs.
- Commits: history uses conventional-style subjects for maintenance (`build(deps): ...`, `chore: release v2.3.0`) and merge commits for PRs.

## Testing

- ExUnit + Hammox. Most files are `async: true` with `setup :verify_on_exit!`; `GeocoderTest` starts its own Supervisor via `start_supervised` in `setup`, and the named processes (`:geocoder_workers`, `:geocoder_store`) mean tests that start a Supervisor must not run concurrently with another doing the same (check before adding more `async: true` Supervisor-based tests).
- Provider tests: stub `Geocoder.HttpClientMock`, call `Provider.geocode/2` directly with `http_client: Geocoder.HttpClientMock`, assert the request map and the resulting `Coords` using `assert_belgium/1` etc. from `Geocoder.Support.Helpers`. Add payload fixtures to `test/support/helpers.ex`.
- `GeocoderTest` runs the full stack against the `Fake` provider by default; `PROVIDER`/`API_KEY` env vars switch to real providers (network + key needed, not for CI).
- Coverage: `mix coveralls`; CI has a `coverage` matrix flag.
- Never `Process.sleep` for the Store's async `link/3` cast; synchronize with a `GenServer.call` such as `Geocoder.Store.state/1`.

Short example (pattern to copy):
```elixir
Geocoder.HttpClientMock
|> expect(:request, fn req, _config ->
  assert req.url == "https://maps.googleapis.com/maps/api/geocode/json"
  {:ok, %{status_code: 200, headers: [], body: belgium_googlemap_payload()}}
end)

{:ok, coords} = GoogleMaps.geocode([address: "..."], http_client: Geocoder.HttpClientMock)
assert_belgium(coords)
```

## Elixir Guidelines

- This is a library: keep the public surface small and backward compatible (README promises compatibility with 1.x process + API). Changing a default, a config key, or a callback signature is a breaking change and needs a CHANGELOG entry and a version decision.
- Prefer pattern matching and `with`; avoid nested `case`. Use `@impl true` on callbacks you add (existing GenServer modules omit it; do not mass-edit unrelated code).
- Never `String.to_atom/1` on user input (addresses, API payloads).
- Do not add a new hard dependency lightly: HTTP and JSON libraries must stay optional and swappable through the behaviours. The README/moduledocs mention Req nowhere; do not introduce it as a default client without a design discussion.
- GenServer boundaries: Worker holds only config; Store holds the cache state. Keep provider HTTP calls inside the checked-out Worker (or its `Task` in the `:stream_to` path), not in the Store.
- Let it crash for genuine bugs; return `{:error, reason}` for provider/HTTP failures (see CHANGELOG 2.2.2 for non-200 handling in the HTTPoison client).
- Avoid `Process.sleep` in tests.

## Workflow for Agents

1. Read the relevant module and its test, plus `Geocoder.Config` if config is involved.
2. Write or adjust the test first (Hammox stub for any new HTTP behaviour).
3. Implement.
4. Run the CI gate commands above (format check, `--warnings-as-errors`, `mix test`; add `mix credo --strict`).
5. Add a CHANGELOG.md entry for any user-visible change.

### Git and PR flow (fork of upstream)
- Never push directly to `master`. All changes land on `epinault/geocoder` `master` via PR, including upstream syncs (branch from `cyrus/master`, PR into the fork).
- `gh` defaults to the upstream repo on forks: always pass `--repo epinault/geocoder` for day-to-day PRs.
- After the fork PR is merged, open the upstream PR immediately and without being asked: `gh pr create --repo CyrusNuevoDia/geocoder --base master --head epinault:master`. Full chain: work branch -> PR into fork master -> (user merges) -> PR `epinault:master` -> `CyrusNuevoDia:master`.

### Hex publishing and CHANGELOG
- Version lives in `@version` in mix.exs. CHANGELOG.md format: `[x.y.z] - YYYY-MM-DD` heading followed by `### Changes` bullets, newest first.
- Release commit: `chore: release vX.Y.Z` (version bump + CHANGELOG), merged via PR.
- Pushing any tag triggers `publish.yml`, which publishes to Hex using the `HEX_API_KEY` secret (OTP 27 / Elixir 1.17). Do not publish or tag unless asked.
- mix.exs `links` point "Changelog" at GitHub releases, not the file.

## Constraints - Never Do

- Never push to `master`; never skip the fork-then-upstream PR chain.
- Never edit `mix.lock` by hand; use `mix deps.update`.
- Never make httpoison, jason or jsx non-optional, and never change their version constraints casually: they define the contract for downstream users.
- Never leave `IO.inspect`, `dbg`, or `IEx.pry` in committed code; never use `--no-verify`.
- Never commit API keys (Google/OpenCage) or `.env` files; tests must pass without a key.
- Never make real network calls in the default `mix test` run.
- Do not commit `erl_crash.dump` (one exists in the working directory; leave it alone).
- Do not tag/publish a release without being asked.

## Gotchas

- Held dependency update: httpoison 2.x -> 3.0 is the only path to hackney >= 4.0.1, which fixes four CVEs in hackney 1.25.0. The constraint is already `~> 2.1 or ~> 3.0`, but `lib/geocoder/http_client/hackney.ex` calls `:hackney.request/5` and `:hackney_url.qs/1` directly and must be checked against the hackney 4 API before relying on it.
- `Geocoder.Request` aliases `Geocoder.Request.HttpClient`, which does not exist (only used in typespecs; real type is `Geocoder.HttpClient`). Harmless today, confusing when editing.
- `Geocoder.Supervisor` moduledoc mentions `Geocoder.Providers.Google`; the real module is `Geocoder.Providers.GoogleMaps`. `Geocoder.JSONCodec` doc mentions `Geocoder.JSON.JSX`; real module is `Geocoder.JSONCodec.JSX`.
- `Config.store_config/1` has a copy-pasted `@spec worker_config(...)`; behaviour is correct.
- `*_list` functions bypass the cache entirely and are typed (incorrectly) as returning a single `Coords.t()` in the `Provider` behaviour.
- The default `:timeout` is 5000 ms per call while `http_client_opts` default `recv_timeout` is 30 s, so the pool call can time out before the HTTP request does.
- The Store is per-VM, unbounded, and has no TTL.
- OpenStreetMaps (default) needs no key but has usage-policy rate limits; do not hammer it in tests.
- The Fake provider matches addresses by exact key or by regex keys in `worker_config[:data]` (see `fake_data_cache/0`).

## Key Files

| Path | Purpose |
|---|---|
| `lib/geocoder.ex` | Public API entry points |
| `lib/geocoder/config.ex` | All config keys, defaults, typespecs |
| `lib/geocoder/supervisor.ex` | What consumers start; pool + store wiring |
| `lib/geocoder/worker.ex` | Cache-then-provider flow, per-call options |
| `lib/geocoder/store.ex` | In-memory cache and geohash logic |
| `lib/geocoder/provider.ex` | Behaviour to implement a new provider |
| `lib/geocoder/providers/open_street_maps.ex` | Reference provider implementation |
| `lib/geocoder/http_client.ex` | HTTP client behaviour and contract |
| `test/test_helper.exs` | Hammox mock definition |
| `test/support/helpers.ex` | Fixtures and `provider_test_config/2` |
| `mix.exs` | Version, deps, package/docs config |
| `CHANGELOG.md` | Release notes, update on each release |
| `.github/workflows/elixir.yml` | The CI gate |
