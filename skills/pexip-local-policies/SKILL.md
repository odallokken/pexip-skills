---
name: pexip-local-policies
description: >
  Expert knowledge for writing Pexip Infinity local policy scripts — the Jinja2 templates
  configured in Call control > Policy profiles that run on each Conferencing Node to
  modify service configuration, participant properties, or media location data. Use this
  skill whenever the user is writing, debugging, or reviewing a Pexip local policy script,
  mentions a "local policy", a "policy profile", a "jinja2 policy", `pex_update`,
  `pex_to_json`, `pex_debug_log`, `pex_in_subnet`, `pex_now`, `service_config`,
  `call_info`, `suggested_media_overflow_locations`, or `preauthenticated_role`. Also
  triggers for tasks like rejecting calls from suspect User-Agents, removing the PIN for
  internal/registered devices, anonymizing PSTN callers, routing an alias into a breakout
  room, pinning media to a specific location/overflow set, time-of-day theme switching,
  locking VMRs by service tag, ABAC via IdP attributes inside a Jinja2 script, or using
  the built-in "Test local …​ policy" facility under a policy profile. Prefer this skill
  over the broader `pexip-external-policy` skill whenever the user is staying inside
  Pexip Infinity (no external HTTP server) and only needs a Jinja2 template — local
  policy is simpler, runs on each Conferencing Node, and has a different (smaller) set
  of variables and response fields than the external policy API.
---

# Pexip Infinity Local Policies — Expert Skill

Comprehensive knowledge for writing Pexip Infinity **local policy** scripts: the Jinja2
templates configured per *policy profile* under **Call control > Policy profiles** that
run on every Conferencing Node and can transform service configuration, participant
properties, and media location data before Pexip Infinity acts on them.

Local policy is the lightweight cousin of external policy:

- **In-process Jinja2** on each Conferencing Node — no external HTTP server, no network round-trip.
- **Stateless** — has no view of running conferences or participants beyond the single in-flight request.
- **Three request types only**: service configuration, participant properties, media location.
  (Local policy can **not** do avatar, directory, or registration policy — those are external-only.)
- Runs **after** external policy when both are enabled, so it sees the merged result.

For background on the external policy API (six request types, HTTP, ABAC patterns, etc.) use
the `pexip-external-policy` skill instead.

---

## Quick Decision Tree

| Goal | Read this first |
|---|---|
| Understand the basic script shape and `action`/`result` envelope | §1 below |
| Find the right variable name (`call_info.*`, `participant.*`, `service_config.*`) | `references/variables.md` |
| Find a filter (`pex_update`, `pex_to_json`, `pex_in_subnet`, `pex_now`, `pex_regex_*`, …) | `references/filters.md` |
| Build a service configuration response (VMR, gateway, virtual reception, breakout, ADP) | `references/service-config-response.md` |
| Build a participant policy response (role, display name, reject reason) | `references/participant-response.md` |
| Build a media location response (principal + overflow locations) | `references/media-location-response.md` |
| See a complete, working script you can copy/paste | `examples/` and `references/example-scripts.md` |
| Debug a script that isn't doing what you expect | §6 below |
| Configure the policy profile in the admin UI | §2 below |

Reference files contain the **complete** field tables (every supported field, type, default
and meaning) that the body of this `SKILL.md` only summarises.

---

## 1. The Script Shape Every Local Policy Has

Every local policy script returns a single JSON object with an `action` and a `result`:

```jinja2
{
    {% if service_config %}
        "action" : "continue",
        "result" : {{service_config|pex_to_json}}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

This is the **canonical "no-op" service configuration script** — start every new script from
this skeleton, then add logic. The same shape applies to participant and media location
scripts (with slightly different `action` and `result` semantics — see the reference files).

**Five things to internalise:**

1. The **output is a JSON document** that Pexip Infinity parses. Jinja2 is just a templating
   layer that produces text — your job is to produce *valid JSON*. Mismatched commas, an
   unquoted key, or a stray `{% else %}` branch that emits nothing will make Pexip fall back
   to default behaviour (or in some cases reject the call). The built-in test facility shows
   you exactly what was generated.

2. **`action` controls Pexip's behaviour**, not the `result` payload alone:
    - Service config: `"continue"` (use `result`), `"reject"` (return "conference not found"),
      or `"redirect"` (send SIP 302; `result` must be `{"new_alias": "sip:..."}`).
    - Participant: `"continue"` or `"reject"` (optionally with `result.reject_reason`).
    - Media location: no `action`; just a `result` object.

3. **`service_config` (and friends) are Python dicts, not JSON.** To emit one as part of the
   response you **must** put it through the `pex_to_json` filter. To mutate it use
   `pex_update({...})` first. The idiomatic pattern is
   `{{ service_config | pex_update({...}) | pex_to_json }}`.

4. **`service_config` can be `None`** — for service configuration scripts where the dialed
   alias does not match any configured VMR/gateway rule/etc. Always guard with
   `{% if service_config %}` (or handle the `None` case to dynamically *create* a service —
   that is how breakout-room-on-the-fly works; see `references/example-scripts.md`).

5. **Maximum script length is 49,152 characters.** If you are getting close, that is the
   signal to move to external policy.

---

## 2. Configuring a Policy Profile

Local policy lives inside a *policy profile* under **Call control > Policy profiles**.
The profile has separate sections for each request type (Service configuration policy,
Participant policy, Media location policy), and each section has its own:

- **Enable external lookup** (sends to an external policy server) — optional.
- **Apply local policy** (runs your Jinja2 script) — what this skill is about.
- **Script** (the Jinja2 template) — only visible when "Apply local policy" is on.

A profile is then **assigned to one or more system locations** under
**Platform > Locations**. *A profile that is not assigned to a location is never used.*

When both external and local policy are enabled for the same data type, **external runs
first, then local sees the merged result** (and any `external_policy_disposition` it set,
exposed as `call_info.external_policy_disposition`). This lets you use local policy as a
sanity-check / override layer on top of an external policy server.

---

## 3. The Three Variables You Will Use Most

| Variable | Lives in | Mutable? | Holds |
|---|---|---|---|
| `call_info` | All three script types | Read-only | Facts about the call/request (alias, location, protocol, remote address, vendor, IdP UUID, etc.). |
| `service_config` | All three script types | Mutable (via `pex_update`) | The existing service configuration retrieved from DB or external policy. May be `None` for service-config scripts when the alias is unknown. |
| `participant` | **Participant** policy scripts only | Mutable | The participant who is about to join (role, display name, audio mix, IdP attributes). |
| `suggested_media_overflow_locations` | **Media location** policy scripts only | Mutable | The principal + overflow locations Pexip would use absent your policy. |

`call_info` is by far the most common gateway into your decisions. The two SIP-header-derived
fields are spelled differently in local policy than they are in the external policy API:
**`ms_subnet` and `p_asserted_identity`** (underscores and lowercase) — and they arrive as
JSON lists, not strings. See `references/variables.md` for the full field tables and which
fields are populated for which request types.

---

## 4. Filters: the Pexip-specific Vocabulary

Local policy uses a **subset** of Jinja2 filters (the ones Pexip enables: `abs`, `capitalize`,
`default`, `first`, `float`, `format`, `int`, `join`, `last`, `length`, `lower`, `range`,
`replace`, `round`, `striptags`, `trim`, `truncate`, `upper`) plus a set of custom
`pex_*` filters. The ones you will use constantly:

- **`pex_to_json`** — convert a Python dict (e.g. `service_config`) into the JSON Pexip needs.
- **`pex_update({...})`** — return a copy of a dict with the supplied keys overridden. Chains
  with `pex_to_json`: `{{ service_config | pex_update({"pin":""}) | pex_to_json }}`.
- **`pex_debug_log("message", var1, var2, ...)`** — writes to the Pexip support log
  (**History & Logs > Support log**, search for `pex_debug_log`). The single best debugging
  tool. **Remove from production scripts** — debug lines fill the support log and cause
  rotation.
- **`pex_in_subnet(address, [cidr, cidr, ...])`** — boolean subnet test. Works for IPv4 and
  IPv6. Pair with `call_info.remote_address` for location-routing decisions.
- **`pex_regex_search`**, **`pex_regex_replace`** — full regex support; replace is for
  patterns that go beyond what the built-in `replace` filter can do.
- **`pex_now("Europe/London")`** — current time in the given timezone (UTC by default), with
  `.year`, `.month`, `.day`, `.hour`, `.minute`, `.second` attributes. Use for time-of-day or
  date-based decisions (after-hours theme switch, holiday banners).
- **`pex_hash`**, **`pex_md5`**, **`pex_uuid4()`**, **`pex_random_pin(n)`** — deterministic and
  random IDs. `pex_uuid4()` and `pex_random_pin()` take no input (note the parens).
- **`pex_require_min_length(n)`** — if the input string is shorter than `n`, the policy is
  *not applied* and the original configuration is retained (useful for sanity checks).

Full table with every filter and worked examples: `references/filters.md`.

---

## 5. The Three Response Shapes (in one screen)

### Service configuration (`action` + `result`)

```jinja2
{
    "action": "continue",
    "result": { "service_type": "conference", "name": "...", "service_tag": "...", ... }
}
```

`service_type` selects which set of fields are valid: `"conference"` (VMR), `"lecture"`
(Virtual Auditorium), `"gateway"`, `"two_stage_dialing"` (Virtual Reception),
`"media_playback"`, `"test_call"`. Each has its own required-field list and a long tail of
optional fields (PINs, layouts, themes, encryption, ADPs, breakout fields, …). The full
per-`service_type` field tables — including the breakout fields and the ADP sub-object —
live in `references/service-config-response.md`.

### Participant properties (`action` + `result`)

```jinja2
{
    "action": "continue",
    "result": { "preauthenticated_role": "guest", "remote_display_name": "Witness A" }
}
```

All fields are *overrides* of the current participant properties — omit a field and Pexip
keeps the existing value. Common fields: `preauthenticated_role`, `remote_alias`,
`remote_display_name`, `bypass_lock`, `call_tag`, `disable_overlay_text`, `layout_group`,
`spotlight`, `rx_presentation_policy`, `send_to_audio_mixes`, `receive_from_audio_mix`.
On reject you may include `result.reject_reason` — webapp users will see it. Full table:
`references/participant-response.md`.

### Media location (`result` only — no `action`)

```jinja2
{
    "result": { "location": "Oslo", "overflow_locations": ["London", "New York"] }
}
```

If any name in the response does not match a real location configured in Pexip, **the entire
response is discarded** and Pexip falls back to default behaviour. Full details and worked
examples: `references/media-location-response.md`.

---

## 6. Debugging a Local Policy Script

### Use the built-in test facility first

Every policy profile has **Test local service configuration policy**, **Test local
participant policy**, **Test local media location policy** buttons at the bottom of the
profile page. Each opens a page with:

- **Input** — sample `call_info` plus the relevant config variable in JSON. Edit freely.
- **Script** — pre-filled with the current saved script.
- **Result** — what your script produced.
- **Diagnostics** — error detail if the script failed to parse / produce valid JSON.

Iterate here before saving the profile. **Save changes and return** persists the script;
**Cancel** throws away your edits.

### Use `pex_debug_log` on live calls

For data you cannot reproduce in the test facility (e.g. the exact `call_info` from an
SBC-fronted SIP call, or IdP attributes), drop one line at the top of the script:

```jinja2
{{ pex_debug_log("Local policy call_info=", call_info) }}
```

Place a test call, then **History & Logs > Support log** and search for `pex_debug_log`.
Remove the line as soon as you are done.

### Common failure modes

| Symptom | Cause |
|---|---|
| "Conference not found" for an alias that exists | Script's `{% else %}` falls through and emits `"action": "reject"`. |
| Policy seems ignored, original behaviour wins | Output is not valid JSON (or media-location names don't match configured locations) — Pexip silently falls back. Run it through the test facility; check Diagnostics. |
| `service_config.<field>` is `None` and the script crashes | The field is optional and not set. Use `service_config.get("field")` or `{% if service_config.field %}`. |
| Field name in `call_info` looks wrong (e.g. `ms-subnet`) | Local policy renames SIP-derived fields to `ms_subnet` and `p_asserted_identity` (lowercase + underscores). |
| `participant.idp_attributes.X-wing pilot` raises | Hyphens and spaces in attribute names need `.get()` — `participant.idp_attributes.get("X-wing pilot")`. |
| `preauthenticated_role` check never matches | Compare against `None`, not the string `"None"`. Pre-PIN SIP/H.323 participants have a real `None` here. |
| Breakout doesn't open | `name` must be the existing main VMR; `breakout: true` and `breakout_rooms: false`; `breakout_uuid` is required. |
| Script edits silently dropped on save | Length over the 49,152-character limit. Time to move to external policy. |

---

## 7. When to Reach for External Policy Instead

Local policy is the right choice for:

- Small, declarative transformations of service config / participant / media location.
- Decisions that depend only on the in-request data (alias, protocol, location, IdP
  attributes, subnet) plus optional time-of-day.
- Hard-coded look-up tables (subnet → location, alias pattern → breakout room).

You will hit a wall when you need:

- State or memory across calls (running conferences, dedup, rate-limit).
- Calls to *any* external API (LDAP, Graph, SQL, internal phonebook).
- Conditional decisions that depend on what other participants are doing.
- Avatar, directory, or registration policy (local policy cannot do these).

When you cross that wall, switch to an external policy server (the `pexip-external-policy`
skill covers all six request types, ABAC, breakout patterns and gotchas) — and consider
keeping a *thin* local policy that runs after it as a safety/override layer.

---

## 8. Checklist Before Saving a Local Policy Script

- [ ] Script starts from the pass-through skeleton (`{% if service_config %}` … `{% else %}` …)
- [ ] Every code path emits a syntactically valid JSON document
- [ ] Every dict you embed in `result` is passed through `pex_to_json`
- [ ] Every mutation is done via `pex_update({...})`, not by editing dict members directly
- [ ] You handle `service_config is None` (for service-config scripts)
- [ ] You handle `preauthenticated_role is None` (for participant scripts, SIP/H.323 path)
- [ ] Media-location names exactly match Pexip system location names (case-sensitive)
- [ ] Any `pex_debug_log` calls are removed (or you have a TODO to remove them)
- [ ] You tested at least one positive and one negative case in the built-in test facility
- [ ] You assigned the policy profile to the relevant system location(s)

---

## References

- `references/variables.md` — full `call_info`, `participant`, `service_config`,
  `suggested_media_overflow_locations` field tables (including which fields are populated
  for which request type).
- `references/filters.md` — every supported Jinja2 filter and every custom `pex_*` filter,
  with syntax and worked examples.
- `references/service-config-response.md` — every field, every `service_type`, plus
  breakout-room fields and the automatic-participants (ADP) sub-object.
- `references/participant-response.md` — participant properties response fields.
- `references/media-location-response.md` — media location response, overflow rules, and
  the location-name matching gotcha.
- `references/example-scripts.md` — annotated copy-paste examples for the most common
  patterns (pass-through, debug, reject suspicious UAs, remove PIN by location/registration,
  subnet-based media routing, time-of-day theme, lock-by-tag, breakout-by-alias, mask PSTN
  numbers, IdP-attribute-based admission, disable encryption to a specific device).
- `examples/` — the same scripts as standalone `.jinja2` files you can copy directly into
  the policy profile UI.
