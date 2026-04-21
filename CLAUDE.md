# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `bin/setup` — install Ruby (via mise), bundle, and run `db:prepare`. Add `--reset` to recreate the DB.
- `bin/dev` — start the Rails server on **port 3007** (not the default 3000).
- `bin/rails test` — Minitest. Parallelized by CPU. Append a file path or `file:line` to run a single test.
- `bin/rails test:system` — Capybara + Selenium. Screenshots land in `tmp/screenshots` on failure.
- `bin/rails db:setup test test:system` — full CI equivalent (matches `.github/workflows/ci.yml`).
- `bin/rubocop` — lints with `rubocop-rails-omakase`.
- `bin/brakeman` — security scan (runs in CI).
- `bin/importmap audit` — JS dependency audit (runs in CI).
- `bin/jobs` — run Solid Queue supervisor standalone (not needed in production; Puma runs it in-process via `SOLID_QUEUE_IN_PUMA=true`).
- `bin/kamal deploy` — deploy to the host configured in `config/deploy.yml`. Also `bin/kamal console`, `bin/kamal logs`, `bin/kamal shell`, `bin/kamal dbc` (aliases defined in `deploy.yml`).
- `script/admin/reset-password` and `script/admin/prepare-backup` — operational scripts intended to be run inside a deployed container.

## Architecture

Writebook is a Rails app (tracking `rails/rails` main, Ruby 4.0.2) for publishing books. SQLite is the only database — including for FTS5 full-text search — so code paths that use SQL extensions (`match`, `bm25`, `highlight`, `snippet`) are SQLite-specific and **not** portable to Postgres/MySQL.

### Multi-DB layout

Four SQLite files under `storage/db/`, all on the same `writebook_storage` volume in production:

- `*_primary.sqlite3` (just `*.sqlite3`) — domain tables, Active Storage blobs, FTS5 `leaf_search_index` virtual table.
- `*_queue.sqlite3` — Solid Queue (jobs). Schema lives in `db/queue_schema.rb`, not `schema.rb`.
- `*_cache.sqlite3` — Solid Cache (`Rails.cache`). Schema in `db/cache_schema.rb`.
- `*_cable.sqlite3` — Solid Cable (Action Cable broadcasts, production only). Schema in `db/cable_schema.rb`.

Only `primary` is in `db/schema.rb`. The Solid sub-DBs use their own `*_schema.rb` files and `migrations_paths: db/{queue,cache,cable}_migrate`. `db:prepare` handles all four. Losing the `storage/` volume loses all of them.

### Domain model: Book → Leaf → Leafable

Books are composed of ordered `Leaf` records. A `Leaf` is a `delegated_type` wrapper whose `leafable` is a `Page`, `Section`, or `Picture` (see `Leafable::TYPES` in `app/models/leafable.rb`). When adding a new content type, add it to `TYPES`, include `Leafable` in the model, and follow the existing `pages_controller` / `sections_controller` / `pictures_controller` structure plus its `app/views/<type>s` views.

- `Book#press(leafable, leaf_params)` — the canonical way to attach a new leafable to a book; do not create `Leaf` records directly.
- `Leaf` includes `Editable`, `Positionable`, `Searchable`, has `status` (`active` / `trashed`), and delegates `searchable_content` to the leafable.
- `Leaf` titles parameterize into slugs (`Leaf#slug` falls back to `"-"` when empty). URL helpers `book_slug` / `leafable_slug` / `leafable` / `edit_leafable` in `config/routes.rb` handle slugged URLs and delegated-type polymorphism — use them rather than hand-rolled paths.

### Positioning

`Positionable` uses a float `position_score` column with gap-based insertion. When gaps collapse below `REBALANCE_THRESHOLD`, positions are rebalanced in an `after_save_commit`. All moves are wrapped in `positioning_parent.with_lock`. If you touch this concern, keep the lock and the rebalance logic intact.

### Search

`Leaf::Searchable` maintains the `leaf_search_index` FTS5 virtual table via `after_*_commit` hooks. Callers should go through `Leaf.search(terms)` which calls `sanitize_query_syntax` first — passing raw user input to the FTS5 `match` clause will blow up on stray punctuation or unbalanced quotes. `sanitize_for_index` treats `html_safe` strings as already-escaped (ActionText plain text flow) and only full-sanitizes otherwise; preserve that distinction if you add a new leafable, or you'll double-encode or leak tags into the index.

Search display-time sanitization lives separately — treat FTS output as untrusted and run it through the display-time sanitizer before rendering (the recent `182a76d` XSS fix is the reference).

### Authentication & authorization

- `Authentication` concern (in `app/controllers/concerns/`): cookie-based sessions via `Session` records with `token` / `ip_address` / `user_agent`. `require_authentication` runs on every controller by default; opt out per-action with `allow_unauthenticated_access` or `require_unauthenticated_access`.
- `Authorization` concern (in `app/models/concerns/`, but used in controllers): provides `ensure_can_administer` and `ensure_current_user`. Returns `head :forbidden` — it does not redirect.
- Users have a `User::Role` enum (`member` / `administrator`). `administrator?` grants edit on every book regardless of `Access`.
- `Book::Accessable` layers per-book permissions: each `Access` has `level` `reader` or `editor`. `everyone_access` on a book means "any active user is a reader" — new users are auto-granted reader access to all such books via `User#grant_access_to_everyone_books`.
- `Current.user` / `Current.session` are the canonical accessors. Don't reach for `session[:user_id]`.

### First-run & accounts

This is a single-tenant app: exactly one `Account` exists, created by `FirstRun.create!` together with the initial administrator user and demo content. Don't add logic that assumes multiple accounts.

### Markdown rendering

`lib/rails_ext/action_text_has_markdown.rb` extends ActiveRecord with `has_markdown`, which stores content in the `action_text_markdowns` table (name-scoped, polymorphic on `record`) rather than `action_text_rich_texts`. `Page` uses `has_markdown :body`. Rendering goes through Redcarpet with `ActionText::Markdown::DEFAULT_RENDERER_OPTIONS` and Rouge for code highlighting. When reading page content programmatically, `Page#markable` returns the raw markdown source; `Page#searchable_content` returns HTML-escaped plain text for the FTS index.

### Frontend

Propshaft + importmap + Turbo + Stimulus. No bundler, no Node build step. JS lives in `app/javascript/controllers` (Stimulus) and `app/javascript/actions` (custom helpers registered in `application.js`). `allow_browser versions: :modern` gates on webp, web push, import maps, CSS `:has`, and CSS nesting — don't add polyfills for older browsers.

### Deploy (Kamal 2)

`config/deploy.yml` defines a single `web` service with no accessories — everything (web, jobs, cache, cable, FTS index) is SQLite on one volume. Kamal-proxy terminates TLS and the container runs with `DISABLE_SSL=true` so Thruster doesn't fight it for port 443. Jobs run inside Puma via `SOLID_QUEUE_IN_PUMA=true`; there is no separate `job` role, no Procfile, no `bin/boot` monitor. The container CMD is literally `./bin/thrust ./bin/rails server`. `bin/docker-entrypoint` runs `db:prepare` before the server starts, which creates any missing sub-DBs on first boot. Single-host only — the `writebook_storage` volume pins deploys to one machine.

## Testing conventions

From `AGENTS.md`, these are project rules, not stylistic suggestions:

- **Fixtures only.** Prefer `leaves(:welcome_page)` and friends in `test/fixtures/*.yml` over building records in tests. Grep existing fixture names before adding a new one.
- **Use `_path`, not `_url`,** in controller and integration tests unless you're explicitly testing across hosts.
- **Use `assert_in_body` / `assert_not_in_body`** instead of `assert_includes response.body, ...`.
- **Omit `{ render }`** in `respond_to` blocks when the default render is wanted — e.g. `format.html` alone, not `format.html { render }`.

`test/test_helper.rb` auto-loads all fixtures and includes `SessionTestHelper` (see `test/test_helpers/`) for signing in as a fixture user.

## Release process

See `CONTRIBUTING.md`. Releases are cut by pushing an annotated `v*` git tag to origin; `publish-image.yml` builds and publishes signed multi-arch images to `ghcr.io/basecamp/writebook`. The legacy ONCE release script is no longer in the repo — retrieve from commit `0ecca477^:bin/release` if needed and coordinate with the team before running.
