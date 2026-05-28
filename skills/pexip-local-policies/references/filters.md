# Local policy filters — reference

Local policy supports a **subset** of standard Jinja2 filters and a set of custom Pexip
filters. Anything not listed below is disabled.

> **Source:** docs.pexip.com — *Using filters in local policy scripts*.

## Supported standard Jinja2 filters

`abs`, `capitalize`, `default`, `first`, `float`, `format`, `int`, `join`, `last`, `length`,
`lower`, `range`, `replace`, `round`, `striptags`, `trim`, `truncate`, `upper`.

Syntax is the normal Jinja form: `{{ <source>|<filter>(<args>) }}` and filters can be chained
(`{{ sn | truncate(5) | upper }}`). The two heavyweights you will see all over local policy
scripts are:

- `trim` — `{{ title | trim }}` removes leading/trailing whitespace.
- `replace` — `{{ department | replace("Personnel", "HR") }}` (optional 3rd arg = max count).
  For anything more complex, use `pex_regex_replace`.

For richer text/list manipulation see the upstream Jinja2 docs:
<https://jinja.palletsprojects.com/en/stable/templates/#list-of-builtin-filters>.

## Custom Pexip filters

| Filter | Description |
|---|---|
| `pex_base64` | Base64-encode the input. |
| `pex_clean_phone_number` | Keep only `+0123456789`, strip everything else (letters, punctuation, etc.). |
| `pex_debug_log(message, ...)` | Write a message (literal text + any variables) to the Pexip support log. **Remove from production scripts** — fills the log and causes rotation. View results under **History & Logs > Support log**, search for `pex_debug_log`. |
| `pex_find_first_match(list, 'regex')` | Return the first element of `list` matching the regex. |
| `pex_hash` | Hash the input field. Deterministic — same input always gives same output (used for stable per-caller breakout UUIDs). |
| `pex_head(n)` | Return at most the first `n` characters. |
| `pex_in_subnet(address, [cidr, ...])` | Boolean. Works for IPv4 and IPv6. |
| `pex_md5` | MD5 hash of the input. |
| `pex_now(timezone)` | Current datetime in the given timezone (`"UTC"` by default, e.g. `"Asia/Tokyo"`, `"US/Eastern"`). Attributes: `year`, `month`, `day`, `hour`, `minute`, `second`, `microsecond`. |
| `pex_random_pin(length)` | Generate a random PIN of `length` digits. Takes no input — `{{ ""|pex_random_pin(6) }}` style or `{% set p = pex_random_pin(6) %}`. |
| `pex_regex_replace('find', 'replace')` | Regex find-and-replace. |
| `pex_regex_search('pattern', 'string')` | Search; returns the captured groups (use `{% set groups = pex_regex_search(...) %}` then index `groups[0]`, `groups[1]`, …). |
| `pex_require_min_length(n)` | Validate the input string has at least `n` characters. **If it fails, the local policy is not applied and the existing configuration is kept unchanged.** Useful as a safety check. |
| `pex_reverse` | Reverse the characters in the input. |
| `pex_safelinks_decode` | Convert a Microsoft Defender Safe Links URL back to the original. |
| `pex_strlen` | Length of a string. `{% set n = "Example"|pex_strlen %}` ⇒ `7`. |
| `pex_tail(n)` | Return at most the last `n` characters. |
| `pex_to_json` | **Convert a Python dict variable into JSON.** Required for `service_config` and `suggested_media_overflow_locations` when embedded in a response. |
| `pex_to_uuid` | Convert a base64 string into a UUID. |
| `pex_update({...})` | Return a copy of a dict with the supplied keys overridden/added. Pairs with `pex_to_json`. |
| `pex_url_decode` | URL-decode a previously URL-encoded string. |
| `pex_url_encode` | URL-encode safely for use in URL parameters. |
| `pex_urldefense_decode` | Convert a Proofpoint URL Defense URL back to the original. |
| `pex_uuid4()` | Generate a UUID. No input. |

## Worked examples

### `pex_update` + `pex_to_json` (the canonical mutate-then-emit)

```jinja2
{{ service_config | pex_update({"pin": "123456", "guest_pin": "", "allow_guests": false}) | pex_to_json }}
```

### `pex_debug_log`

```jinja2
{{ pex_debug_log("Media location policy ", call_info.location, " protocol ", call_info.protocol) }}
```

Produces a support-log entry like:

```
Name="support.jinja2" pex_debug_log Detail="Media location policy ,Oslo, protocol ,sip"
```

### `pex_in_subnet`

```jinja2
{{ pex_in_subnet("10.47.0.1", ["10.47.0.0/16", "10.147.0.0/16"]) }}        ⇒ True
{{ pex_in_subnet("10.44.0.1", ["10.47.0.0/16", "10.147.0.0/16"]) }}        ⇒ False
{{ pex_in_subnet("2001:658:22a:cafe:ffff:ffff:ffff:ffff", ["2001::/16"]) }} ⇒ True
{{ pex_in_subnet(call_info.remote_address, ["10.44.0.0/16"]) }}            (typical real usage)
```

For multi-location subnet routing, see the example in `example-scripts.md`.

### `pex_now`

```jinja2
{% set now = pex_now("Europe/London") %}
{% if now.month == 2 and now.day == 29 %}
    {# leap-day special #}
{% endif %}
```

### `pex_regex_search`

```jinja2
{% set groups = pex_regex_search("([a-z0-9.-]+)@([a-z0-9.-]+.com)", "example string with someone@example.com") %}
{% if groups %}
    {{ groups[0] }}@{{ groups[1] }}
{% endif %}
```

### `pex_regex_replace`

```jinja2
{{ mail | pex_regex_replace('@.+', '@otherdomain.com') }}
```

Transforms `user1@domainA.com` → `user1@otherdomain.com`.

### `pex_require_min_length`

```jinja2
{{ some_string | pex_require_min_length(2) }}
```

If `some_string` is shorter than 2 chars, the local policy is *not* applied and the original
configuration is retained — a one-line safety guard.

---

## Notes & gotchas

- **`pex_to_json` is only required when you embed a Python dict variable.** If you are
  hand-building the JSON literal yourself in the template (`"result": {"name": "..."}`),
  you do not need it.
- **`pex_update` does not mutate in place** — it returns a new dict. Chain it as
  `{{ service_config | pex_update({...}) | pex_to_json }}` rather than
  `{% set service_config = service_config | pex_update({...}) %}`.
- **Hyphenated / space-containing keys** in `idp_attributes` need `.get("X-wing pilot")`
  rather than dot access (Jinja2 / Python attribute syntax does not allow hyphens).
- **Remove all `pex_debug_log` calls** before going to production — they write to the support
  log on every call.
