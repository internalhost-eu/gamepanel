# InternalHost — gamepanel fork

`gamepanel` is [InternalHost](https://internalhost.eu)'s deployment of
[Pterodactyl Panel](https://github.com/pterodactyl/panel). It tracks upstream's
`1.0-develop` branch (Laravel 11 + React/TypeScript) and adds a thin,
rebase-friendly overlay. All fork-owned files live under `.internalhost/` so they
never collide with upstream during a rebase.

## Relationship to upstream

| | |
|---|---|
| Upstream | `pterodactyl/panel` @ `1.0-develop` |
| Base commit | `676d25c0` ("wait for lock properly", #5638) |
| App version stamp | `1.0-develop+ih.676d25c` (replaces upstream's `'canary'`) |

To pull upstream changes:

```bash
git remote add upstream https://github.com/pterodactyl/panel.git   # once
git fetch upstream 1.0-develop
git rebase upstream/1.0-develop          # or merge, per maintainer preference
```

> **Note:** there is **no stable tag** for this codebase. All of Pterodactyl's
> tagged releases (latest `v1.12.4`) are on the older **Laravel 10** line. The
> `1.0-develop` branch is the unreleased next-gen panel — so we pin a known-good
> commit instead of a release tag.

## Divergence from upstream

Changes intentionally carried on top of upstream (see branch
`harden/pin-deps-config`, commit `e7babbdc5`):

### Security / dependencies
- **Version stamp** — `config/app.php` `version` set to `1.0-develop+ih.676d25c`
  (traceable) instead of `'canary'`.
- **PHP deps refreshed** via `composer update` — clears 17 CVE advisories that
  affected the committed lockfile (Laravel 11.54, `symfony/mime` 7.4.13,
  `phpseclib` 3.0.52, `aws/aws-sdk-php` 3.383.1, `league/commonmark` 2.8.2).
  `composer audit` is clean.
- **JS deps refreshed** via in-range `yarn upgrade` — `yarn audit` advisories
  173 → 3 (the residual `dset` / `uuid` are transitive and pinned by upstream's
  manifest ranges; left alone to avoid `resolutions` churn). Production build
  verified to still compile.
- **CI** — `composer audit --locked` runs on every PR (non-blocking, so newly
  published advisories surface without turning unrelated PRs red).

### ⚠️ Compatibility hold — `laravel/sanctum` pinned to `~4.2.0`
Sanctum **4.3** changed `Laravel\Sanctum\NewAccessToken` to a typed promoted
property `public PersonalAccessToken $accessToken`. Pterodactyl's override
`app/Extensions/Laravel/Sanctum/NewAccessToken.php` assigns a
`Pterodactyl\Models\ApiKey` to that property, which throws a `TypeError` under
4.3 and **breaks API-key creation** (`POST /api/client/account/api-keys`).
Upstream's committed lock also ships 4.2.x, so this only bites on a dep bump.

**Do not lift this pin** until upstream Pterodactyl adapts the override. When
they do, this is the line to revert in `composer.json`:
```json
"laravel/sanctum": "~4.2.0",   // revert to "^4.2.0" once upstream supports 4.3
```

### Performance / deployment
- **OPcache** enabled and tuned for production in the Docker image
  (`.github/docker/opcache.ini`, installed via `docker-php-ext-install opcache`).
- **`.env.example`** defaults `CACHE_DRIVER` and `SESSION_DRIVER` to `redis`
  (queue was already `redis`, so Redis is already a hard dependency of the
  recommended setup; this just makes it consistent and multi-node ready).
- **`Dockerfile`** tightens `bootstrap`/`storage` perms from `777` to `775`.

### Deliberately *not* changed
- **In-container certbot** is left intact. It is opt-in (only runs when
  `LE_EMAIL` is set) and deeply wired into `.github/docker/entrypoint.sh`.
  InternalHost terminates TLS at the edge (Caddy / Cloudflare), so it stays
  dormant — removing it would break the documented HTTPS path and fight rebases.

## Known cosmetic findings (not addressed)
- **phpstan** reports ~31 findings after the dep bump (mostly
  `staticClassAccess.privateProperty` in the `HasRealtimeIdentifier` trait, plus
  stale `@phpstan-ignore` annotations). These come from `larastan` 3.8 → 3.10
  getting stricter — they are **not** runtime errors and phpstan is not a CI
  gate. Clean up separately if desired.

## Roadmap
- **Paymenter integration** — see [`PAYMENTER_INTEGRATION.md`](./PAYMENTER_INTEGRATION.md).
