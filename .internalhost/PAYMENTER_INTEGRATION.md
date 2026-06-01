# Paymenter ⇄ gamepanel integration — prep notes

Context preparation for wiring this panel into
[`internalhost-paymenter`](https://github.com/internalhost-eu) (Paymenter
replaces WHMCS for InternalHost). **Nothing in this panel repo needs to change
for the integration** — the provisioning logic lives on the Paymenter side as a
server module / `ProvisionDriver` (same pattern as the SAIG portal demo). This
doc records the verified panel-side API surface so the Paymenter module can be
built without re-discovering it.

## Auth model (verified in code)
- Provisioning uses the **Application API** under `/api/application/*`.
- Create an **Application API key** in the panel admin → keys are prefixed
  `ptla_` (`app/Models/ApiKey.php:219`; client keys are `ptlc_`).
- Send it as `Authorization: Bearer ptla_…`, `Accept: application/json`.
- The Application API is rate-limited (`throttle:api.application`) and CORS is
  scoped to `/api/application*` only — both already verified. The Paymenter
  server is server-to-server, so CORS is irrelevant; mind the throttle on bulk ops.

## Provisioning lifecycle → endpoints (verified in `routes/api-application.php`)

| Paymenter event | Panel call |
|---|---|
| Order placed (ensure customer) | `GET /api/application/users/external/{external_id}` → if 404, `POST /api/application/users` |
| Service activated / paid | `POST /api/application/servers` |
| Payment overdue / suspend | `POST /api/application/servers/{id}/suspend` |
| Reactivated | `POST /api/application/servers/{id}/unsuspend` |
| Plan change (resources) | `PATCH /api/application/servers/{id}/build` (limits/allocations) · `PATCH …/details` · `PATCH …/startup` |
| Cancelled / terminated | `DELETE /api/application/servers/{id}` (or `/{id}/force`) |
| Pick a node with capacity | `GET /api/application/nodes/deployable` |
| Product → game mapping | `GET /api/application/nests` + eggs, `…/locations`, `…/nodes/{id}/allocations` |

## Key design decisions to make in the Paymenter module
1. **Idempotent linkage via `external_id`.** Both users and servers support an
   `external_id` and an `external/{external_id}` lookup route. Store Paymenter's
   customer-id and service-id there so retries/webhooks never double-provision.
2. **Product → egg/nest + resource limits mapping.** Each Paymenter product
   needs config: nest, egg, default startup vars, CPU/RAM/disk/swap/io,
   databases/backups/allocations, and location *or* node selection
   (prefer `nodes/deployable` for auto-placement).
3. **Suspend vs terminate policy.** Map Paymenter dunning states to
   suspend/unsuspend; only `DELETE` on full cancellation (consider a grace
   window before destructive delete).
4. **Credential handoff.** On first provision, surface the panel URL + the
   created user (panel emails a password-setup link — confirm mail is configured
   on the panel, see `.env` `MAIL_*`).
5. **Secrets.** The `ptla_` key is admin-equivalent — store it in Paymenter's
   encrypted settings, never in the panel repo. Use a dedicated key per
   environment so it can be rotated/revoked independently.

## Pre-integration checklist (panel side)
- [ ] Panel deployed off a pinned commit (done — `1.0-develop+ih.676d25c`), not canary.
- [ ] Redis up (cache/session/queue all use it per hardened `.env.example`).
- [ ] At least one Wings node registered + allocations available.
- [ ] Nests/eggs imported for the games to be sold.
- [ ] Mail configured (so provisioned users can set passwords).
- [ ] A dedicated `ptla_` Application API key minted for Paymenter.
- [ ] Sanctum pin (`~4.2.0`) still in place — see [`FORK.md`](./FORK.md).

## References
- Application API docs: https://dashflo.net/docs/api/pterodactyl/v1/
- Verified routes: `routes/api-application.php` in this repo.
