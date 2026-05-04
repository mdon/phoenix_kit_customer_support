# Claude's Review of PR #1 — Initial release: phoenix_kit_customer_support

**Verdict:** Approved post-merge — clean extraction; no blockers, but several follow-ups for `0.1.1` (LiveView `mount/3` DB reads, PubSub payload bloat, blanket `rescue _`, test coverage).

**Reviewed:** 2026-05-04
**Reviewer:** Claude (claude-opus-4-7)
**PR:** https://github.com/BeamLabEU/phoenix_kit_customer_support/pull/1
**Author:** Tymofii Shapovalov (timujinne)
**Head SHA:** 0c4fe74
**Status:** Merged

## Summary

Initial port of the Customer Support ticketing module from PhoenixKit core into a standalone Hex package (~6.4k LOC across 38 files). The mechanical rename is consistent — module, atom, settings keys, route prefixes, permission key, and Hex package name all moved over without leftovers (resolved by 67d5df6). `mix compile` and `mix format --check-formatted` are green per the PR description. README accurately distinguishes routes owned by this package from routes wired by `PhoenixKitWeb.Integration` in core (added in 1aef5fa, 0c4fe74).

The package ships with the rough edges typical of a first extraction: LiveView `mount/3` doing DB work, full-struct PubSub payloads (a pattern siblings have already moved away from), a 7-line test file, and a couple of `mix.exs` polish gaps. None block the initial release; all are tractable in `0.1.1`.

## Issues

### 1. [MEDIUM] LiveView: DB reads inside `mount/3`
**Files:**
- `lib/phoenix_kit_customer_support/web/list.ex:49` — `PhoenixKitCustomerSupport.get_stats()` in `mount/3`
- `lib/phoenix_kit_customer_support/web/details.ex:56-58` — `load_comments/1`, `load_attachments/1`, `load_status_history/1` in `mount/3`
- `lib/phoenix_kit_customer_support/web/user_details.ex:55-56` — `load_public_comments/1`, `load_attachments/1` in `mount/3`

`mount/3` runs twice per page load (HTTP request → WebSocket upgrade), so each of these queries fires twice on every navigation. Sibling packages explicitly moved DB reads into `handle_params/3` (e.g. `phoenix_kit_publishing` 0.1.5, `phoenix_kit_newsletters` 0.1.1) — same fix applies here. `Web.List` already has a populated `handle_params/3` at line 64; relocating `get_stats()` is a one-line move.

**Risk:** Doubled query volume on every admin/dashboard page load. Becomes more visible as ticket and comment volume grow.

**Fix:** Move the loads into `handle_params/3`, gated on `connected?(socket)` for the initial paint if you want to skip the dead first call entirely.

**Confidence:** 90/100

### 2. [MEDIUM] PubSub broadcasts emit full structs
**File:** `lib/phoenix_kit_customer_support/events.ex:127-160` (`broadcast_ticket_created/1`, `broadcast_ticket_updated/1`, `broadcast_ticket_status_changed/3`, `broadcast_ticket_assigned/3`, etc.)

Broadcasts pass the entire `%Ticket{}` struct (`{:ticket_created, ticket}`). `phoenix_kit_publishing` 0.1.5 explicitly moved to minimal `%{uuid:, slug:}` payloads — receivers refetch through the context. Reasons that apply here too:

1. Full structs bloat PubSub messages, especially with preloads (`:user`, `:assigned_to`).
2. Adding a sensitive field to `Ticket` later silently leaks it to all subscribers.
3. The struct is a snapshot; by the time a receiver renders it, the row may already be obsolete — refetching is the safer default.

**Fix:** Change broadcasts to `%{uuid: ticket.uuid}` (plus the `old_status`/`new_status` atoms for status changes) and document the contract in the moduledoc so host apps don't pattern-match against the old shape. LiveView consumers already need to handle re-fetch on filter changes; the same path covers broadcast hydration.

**Confidence:** 80/100 (pattern-quality issue, not a correctness bug today)

### 3. [LOW] Blanket `rescue _` in `enabled?/0`
**File:** `lib/phoenix_kit_customer_support.ex:83-87`

```elixir
def enabled? do
  Settings.get_boolean_setting("customer_support_enabled", false)
rescue
  _ -> false
end
```

`rescue _` swallows DB connection errors, missing settings tables (pre-migration boot), and bugs in `PhoenixKit.Settings` indistinguishably. Narrow to the specific exceptions `Settings.get_boolean_setting/2` can raise (likely `DBConnection.ConnectionError` and `Postgrex.Error` during boot before migrations have run). Worth auditing the rest of the context module for the same pattern.

**Confidence:** 75/100

### 4. [LOW] Authorization order in `UserDetails`
**File:** `lib/phoenix_kit_customer_support/web/user_details.ex:28-42`

The LiveView loads the ticket from the DB *first*, then verifies ownership against `current_user.uuid`. Functionally correct (the cross-user request still 403s), but the cleaner pattern is a scoped fetch — `get_user_ticket(id, user_uuid)` returning `nil` on ownership failure — same `nil` branch as the not-found case, no extra branch, and no DB read on the wrong-user path.

**Risk:** Not a security bug today; just an order-of-checks code-smell that future refactors could turn into one if the ownership check accidentally moves below a side-effecting load.

**Confidence:** 70/100 (style + defense-in-depth, not exploitable)

### 5. [LOW] Test coverage gap
**File:** `test/phoenix_kit_customer_support_test.exs` (7 lines)

Tests only assert the module is present. The 1061-line `PhoenixKitCustomerSupport` context, four schemas (`Ticket`, `TicketComment`, `TicketAttachment`, `TicketStatusHistory`), and ten LiveViews are untested in this repo. Tests likely exist upstream in core, but the package now ships independently — at minimum, changeset tests for the four schemas and a context smoke test exercising the status workflow (`open → in_progress → resolved → closed`) would catch regressions on `mix.lock` bumps. Sibling `phoenix_kit_publishing` is a reasonable bar.

**Confidence:** 95/100

### 6. [LOW] `mix.exs` polish vs sibling pattern — FIXED
**File:** `mix.exs`

Compared against `phoenix_kit_legal`, `phoenix_kit_emails`, `phoenix_kit_newsletters`:

- Missing `homepage_url: @source_url` in `project/0`
- `defp docs/0` only set `main:` and `source_ref:`; siblings additionally include `extras: ["README.md", "CHANGELOG.md", "LICENSE"]` and `source_url:` so HexDocs renders README + CHANGELOG as navigable pages

**Fix:** Applied in this same review — `mix.exs` now includes `homepage_url`, `main: "readme"`, `source_url`, and `extras`.

`application/0` lists `:phoenix_kit` in `extra_applications`, which is consistent with `phoenix_kit_posts` but inconsistent with `phoenix_kit_legal` / `phoenix_kit_emails` / `phoenix_kit_newsletters` (which list only `:logger`). Either is defensible — `:phoenix_kit` will be started transitively as a dep regardless. Flagging the inconsistency, not changing.

**Confidence:** 95/100

### 7. [LOW] `CHANGELOG.md` minimal — FIXED
**File:** `CHANGELOG.md`

Original was 5 lines. Sibling 0.1.0 entries (e.g. `phoenix_kit_newsletters/CHANGELOG.md`) document the feature surface so Hex package users see what they're getting.

**Fix:** Applied in this same review — CHANGELOG expanded with feature list, settings-key table, and V109 migration notes (modeled on `phoenix_kit_newsletters/0.1.0`).

**Confidence:** 95/100

## Things that are good

- **Schemas** — `Ticket`, `TicketComment`, `TicketAttachment`, `TicketStatusHistory` have required-field validations and proper FK constraints.
- **Attachment XOR** — `TicketAttachment.changeset/2` enforces `ticket_uuid` xor `comment_uuid` via `validate_parent_reference/1` (`lib/phoenix_kit_customer_support/ticket_attachment.ex:122`).
- **No GenServers/Agents** — the Iron Law holds. All work is synchronous context calls and LiveView events; no process organisation-for-organisation's-sake.
- **Routing** — `routes.ex` cleanly mirrors the sibling pattern; admin vs user-dashboard split is documented in the README.
- **Settings keys** — `customer_support_*` follow a consistent naming scheme with sane defaults (false/20/true/true/true/true).
- **Module disable path** — `enabled?/0` short-circuit + `push_navigate` flash applied uniformly across all LiveViews.
- **README post-67d5df6/1aef5fa/0c4fe74** — accurately distinguishes admin routes (owned here) from user-dashboard routes (wired by `PhoenixKitWeb.Integration` in core).

## Recommended Priority

| Priority | Issue | Status |
|----------|-------|--------|
| P1 | LiveView `mount/3` DB reads (List, Details, UserDetails) | Open |
| P1 | PubSub broadcasts emit full structs | Open |
| P2 | Blanket `rescue _` in `enabled?/0` | Open |
| P2 | Test coverage gap (changesets, context, status workflow) | Open |
| P2 | `UserDetails` auth-check ordering | Open |
| P3 | `mix.exs` `homepage_url` + docs `extras` | **FIXED** in this review |
| P3 | `CHANGELOG.md` 0.1.0 expansion | **FIXED** in this review |
| P3 | README badges + Features section | **FIXED** in this review |
