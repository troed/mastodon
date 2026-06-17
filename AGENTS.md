# Mastodon Agent Instructions

## Tech Stack
- **Backend:** Ruby on Rails 8.1 (Rails groups loaded manually in `config/application.rb` — no Active Storage, Action Cable, Action Mailbox, or Action Text)
- **Frontend:** React 18 + Redux Toolkit + TypeScript, compiled via Vite 8
- **Streaming:** Separate Node.js app in `streaming/` (independent `package.json`)
- **Database:** PostgreSQL 14+ (no SQLite, no ActiveRecord::Storage)
- **Cache/Queue:** Redis 7+ / Sidekiq (< 9)
- **Search:** Chewy (Elasticsearch/OpenSearch)
- **Asset pipeline:** Propshaft + Vite (dual — Vite for React, Propshaft for everything else)

## Versions
- Ruby: 3.3+ (`.ruby-version` is 4.0.5)
- Node: 22+ (`.nvmrc` is 24.16)
- Package manager: Yarn 4 (corepack)

## Setup & Running

```bash
# Full setup (gems, node, DB)
RAILS_ENV=development bin/setup

# Run all 4 dev processes (web, sidekiq, stream, vite)
bin/dev
# Falls back to foreman if overmind is not installed
```

`bin/dev` uses `Procfile.dev`:
- `web` — puma on port 3000
- `sidekiq` — background jobs
- `stream` — Node streaming API on port 4000
- `vite` — Vite dev server (HMR)

**`RAILS_ENV` must be set** — `config/boot.rb` aborts if it is not.

## Database

```bash
bin/rails db:prepare    # dev: create + migrate if needed
bin/rails db:setup      # test: create + schema:load + seed
bin/rails db:migrate
```

Test DB names: `mastodon_test`, `mastodon_test2`, etc. (suffixed by `TEST_ENV_NUMBER`).
Schema annotations via AnnotateRB (`.annotaterb.yml`): annotations placed **before** model class, with table + column comments.

## Testing

### Ruby (RSpec)
```bash
bin/rspec                          # all non-skipped specs
bin/flatware rspec                 # parallel (CI default)
bin/rspec spec/models/user_spec.rb # single file
bin/rspec spec/models/user_spec.rb:42 # single line
```

Key RSpec config (`spec/rails_helper.rb`):
- Specs run in **random order** (`config.order = 'random'`)
- Sidekiq defaults to **fake** mode; use `inline_jobs: true` metadata to run inline
- `:js`, `:streaming`, `:search` tags are **excluded by default**
- `:system` specs use Capybara + Playwright (chromium)
- `:request` specs default to https + `Rails.configuration.x.local_domain`
- `:attachment_processing` metadata skips image processing stubs
- `:inline_jobs: true` runs Sidekiq jobs synchronously
- DatabaseCleaner uses **deletion** strategy (transactional fixtures on)
- `Rails.application.reload_routes_unless_loaded` called before every example

Run tagged subsets:
```bash
bin/rspec --tag js              # JS/browser system specs
bin/rspec --tag streaming       # specs needing streaming server
bin/rspec --tag search          # specs needing ES/OpenSearch
```

### JavaScript (Vitest)
```bash
yarn test:js                              # legacy-tests project (jsdom)
yarn test:storybook                       # Storybook + browser (chromium headless)
yarn test:js run -- src/someFile.test.ts  # single file
yarn test                                 # lint + typecheck + test:js
```

Vitest projects: `legacy-tests` (jsdom, `**/__tests__/**/*.{js,ts,tsx}`) and `storybook` (Playwright/chromium).

### Full verification
```bash
yarn test    # lint:js + lint:css + typecheck + test:js run
```

## Linting & Formatting

```bash
# Ruby
bin/rubocop                          # lint
bin/rubocop -a                       # auto-correct

# JS/TS
yarn lint:js                         # ESLint
yarn fix:js                          # ESLint + --fix

# CSS/SCSS
yarn lint:css                        # stylelint
yarn fix:css                         # stylelint + --fix

# Formatter (all files)
yarn format                          # oxfmt
yarn format:check                    # oxfmt --check

# HAML
bin/haml-lint                        # haml-lint
bin/haml-lint -a                     # auto-correct

# TypeScript
yarn typecheck                       # tsc --noEmit
```

**Pre-commit hooks** (husky + lint-staged):
- All files: `oxfmt --no-error-on-unmatched-pattern`
- `Gemfile|*.rb`: `bin/rubocop -a`
- `*.{js,jsx,ts,tsx}`: `eslint --fix`
- `*.{css,scss}`: `stylelint --fix`
- `*.haml`: `bin/haml-lint -a`
- `**/*.ts?(x)`: `tsc -p tsconfig.json --noEmit`
- `app/javascript/**/*.{js,jsx,ts,tsx}`: `yarn i18n:extract` + diff check

## Architecture Notes

- **App structure:** `app/models`, `app/controllers`, `app/services`, `app/workers`, `app/serializers`, `app/policies`, `app/presenters`, `app/helpers`, `app/javascript`, `app/views` (HAML)
- **API routes:** drawn via `draw(:api)` from `config/routes/api.rb`
- **Admin routes:** drawn via `draw(:admin)` from `config/routes/admin.rb`
- **Web app routes:** drawn via `draw(:web_app)` from `config/routes/web_app.rb`
- **Serializers:** ActiveModelSerializers (not JSON API)
- **Authorization:** Pundit policies in `app/policies/`
- **Frontend paths:** TypeScript aliases: `@/*` → `app/javascript/*`, `mastodon/*` → `app/javascript/mastodon/*`, `images/*` → `app/javascript/images/*`, `styles/*` → `app/javascript/styles/*`
- **Features:** `ENV`-gated feature flags via `Mastodon::Feature`
- **Mailers:** previews in `spec/mailers/previews/`
- **CLI:** specs in `spec/lib/mastodon/cli/` auto-tagged `type: :cli`
- **Search specs:** in `spec/search/` auto-tagged `search: true`

## Environment & Config

- Config files in `config/*.yml`: `mastodon.yml`, `settings.yml`, `sidekiq.yml`, `email.yml`, `omniauth.yml`, `vapid.yml`, `translation.yml`, `captcha.yml`, `themes.yml`
- `.env.test` contains ActiveRecord encryption keys — do not commit `.env`
- Test environment compiles JS as production (`NODE_ENV=production` in `.env.test`)
- `bin/tootctl` — Mastodon CLI tool for admin operations
- `bin/rails dev:populate_sample_data` — populate dev DB with sample data
- Default admin login: `admin@mastodon.local` / `mastodonadmin`

## CI Quick Reference

- `test-ruby.yml` — RSpec + system specs + search specs (multi-Ruby matrix)
- `test-js.yml` — vitest (legacy-tests project)
- `lint-ruby.yml` / `lint-js.yml` / `lint-css.yml` / `lint-haml.yml` — linters
- `format-check.yml` — oxfmt check
- `chromatic.yml` — Storybook visual regression
- `test-migrations.yml` — DB migration compatibility

## Server & Deployment

- **Live server:** `ssh -i ssh_mastodon mastodon@headless.local` — runs at `mastodon@headless.local:~/mastodon`
- **Git remotes:** `origin` = `ssh://git@github.com/troed/mastodon` (your fork), `upstream` = `https://github.com/mementomori-social/mastodon` (mementomods fork)
- **Branches:** `main` and `mementomods-2026-06-07` both push to origin
- **Images:** Built with `podman build --format docker` (workstation runs podman 5.8, not Docker daemon)
  - Web + Sidekiq share the same image (same Dockerfile, different `command:` in compose)
  - Streaming has its own Dockerfile at `streaming/Dockerfile`
  - Images pushed to `192.168.0.2:2997/troed/mastodon/packages/`
  - **Build command:**
    ```bash
    # Web + Sidekiq (same image, different target)
    podman build --format docker -t 192.168.0.2:2997/troed/mastodon/packages/mastodon-web:unified-latest -f Dockerfile --target mastodon --load .
    podman tag 192.168.0.2:2997/troed/mastodon/packages/mastodon-web:unified-latest 192.168.0.2:2997/troed/mastodon/packages/mastodon-sidekiq:unified-latest
    podman push 192.168.0.2:2997/troed/mastodon/packages/mastodon-web:unified-latest
    podman push 192.168.0.2:2997/troed/mastodon/packages/mastodon-sidekiq:unified-latest

    # Streaming
    podman build --format docker -t 192.168.0.2:2997/troed/mastodon/packages/mastodon-streaming:unified-latest -f streaming/Dockerfile --load .
    podman push 192.168.0.2:2997/troed/mastodon/packages/mastodon-streaming:unified-latest
    ```
  - **Dockerfile note:** `ARG TARGETPLATFORM=${TARGETPLATFORM}` must be `ARG TARGETPLATFORM` (podman fails on self-referencing ARG before FROM re-declaration)

### Deploying to server

The server uses rootless Docker under the `mastodon` user. Docker socket is at `/run/user/1008/docker.sock` (not the default `~/.docker/run/docker.sock`).

```bash
# 1. Build & push images from workstation (see above)

# 2. On server, pull and tag images to short names used in docker-compose.yml
export DOCKER_HOST=unix:///run/user/1008/docker.sock
docker pull 192.168.0.2:2997/troed/mastodon/packages/mastodon-web:unified-latest
docker tag 192.168.0.2:2997/troed/mastodon/packages/mastodon-web:unified-latest mastodon-web
docker tag 192.168.0.2:2997/troed/mastodon/packages/mastodon-web:unified-latest mastodon-sidekiq
docker pull 192.168.0.2:2997/troed/mastodon/packages/mastodon-streaming:unified-latest
docker tag 192.168.0.2:2997/troed/mastodon/packages/mastodon-streaming:unified-latest mastodon-streaming

# 3. Ensure docker-compose.yml uses the short image names (mastodon-web, mastodon-sidekiq, mastodon-streaming)

# 4. Stop old containers, run migrations, start new ones
cd ~/mastodon
docker compose stop web sidekiq streaming
docker run --rm --network mastodon_internal_network -e RAILS_ENV=production \
  -e DB_HOST=db -e DB_NAME=postgres -e DB_USER=postgres \
  -e REDIS_HOST=redis -e LOCAL_DOMAIN=sangberg.se -e SINGLE_USER_MODE=false \
  -e SECRET_KEY_BASE="$(grep SECRET_KEY_BASE .env.production | cut -d= -f2-)" \
  -e OTP_SECRET="$(grep OTP_SECRET .env.production | cut -d= -f2-)" \
  -e ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY="$(grep ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY .env.production | cut -d= -f2-)" \
  -e ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT="$(grep ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT .env.production | cut -d= -f2-)" \
  -e ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY="$(grep ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY .env.production | cut -d= -f2-)" \
  mastodon-web:latest bundle exec rails db:migrate
docker compose up -d web streaming sidekiq

# 5. Verify health
curl -s http://localhost:3000/health
curl -s http://localhost:4000/api/v1/streaming/health

# 6. Clean up
docker image prune -f
docker builder prune -f
```

## Gotchas

- `boot.rb` requires `RAILS_ENV` — scripts/tasks must set it
- RSpec random order means specs must be isolated (DatabaseCleaner deletion strategy)
- Sidekiq is **fake** by default in tests — real job execution needs `inline_jobs: true`
- `bin/rspec` does not run `:js`, `:streaming`, or `:search` specs by default
- Streaming API runs on a separate port (4000 in dev, configurable via `STREAMING_HOST`/`STREAMING_PORT`)
- Vite dev server provides HMR; production builds via `yarn build:production`
- i18n extraction (`yarn i18n:extract`) must pass pre-commit — locale files are checked for diffs
- Rubocop auto-correct: `bin/rubocop -a` (lint-staged uses this)
- `strong_migrations` gem enforces safe migration patterns
- `bootsnap` speeds up boot — clear `tmp/cache/bootsnap` if stale
