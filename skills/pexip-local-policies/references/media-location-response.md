# Media location response — reference

A media-location policy script returns just a `result` object (no `action`):

```jinja2
{
    "result": { <location_data> }
}
```

| Field | Req | Type | Description |
|---|:---:|---|---|
| `location` | ✓ | string | Principal location to use for media. A Conferencing Node assigned to this location handles the media. |
| `overflow_locations` |   | list | Ordered list of location names to fall back to if `location` is at capacity. |

> **Names must exactly match configured Pexip system-location names** (case-sensitive). If
> any name in the response does not match, **the entire response is discarded** and Pexip
> falls back to default behaviour for this request. There is no partial-apply.

Note the input vs output shape difference: the *input* variable is
`suggested_media_overflow_locations` and uses singular `primary_overflow_location` and
`secondary_overflow_location` keys, whereas the *output* uses a list field
`overflow_locations`.

---

## Worked examples

### Conditional override by inbound location

```jinja2
{
    {% if call_info.location == "Oslo" %}
        "result": {
            "location": "Oslo",
            "overflow_locations": ["London", "Paris", "New York"]
        }
    {% else %}
        "result": {
            "location": "New York",
            "overflow_locations": ["London", "Oslo"]
        }
    {% endif %}
}
```

### Pass-through (no change)

```jinja2
{
    "result": {{ suggested_media_overflow_locations | pex_to_json }}
}
```

(See the gotcha above — input has singular fields, output expects a list. In practice you
typically rebuild the response object explicitly rather than blindly emitting the input.)

### Pin outbound RTMP to a specific location

```jinja2
{
    {% if call_info.protocol == "rtmp" and call_info.call_direction == "dial_out" %}
        "result": {
            "location": "DMZ",
            "primary_overflow_location": "Location B",
            "secondary_overflow_location": "Location C"
        }
    {% else %}
        "result": {{ suggested_media_overflow_locations | pex_to_json }}
    {% endif %}
}
```

> *(This example also illustrates that pass-through of the dict works in practice — Pexip
> accepts both list-style and singular-key responses in different examples; check with the
> built-in test facility for your specific Pexip version.)*

### Subnet-based routing

```jinja2
{% set nl_map = {
    "Oslo":     ["10.47.0.0/16", "10.147.0.0/16", "10.247.200.0/24"],
    "New York": ["10.1.0.0/16",  "10.201.5.0/23"],
    "Sydney":   ["10.61.0.0/16"],
    "London":   ["10.44.0.0/16"]
} %}
{
    {% if pex_in_subnet(call_info.remote_address, nl_map["Oslo"]) %}
        "result": { "location": "Oslo", "overflow_locations": ["London", "New York"] }
    {% elif pex_in_subnet(call_info.remote_address, nl_map["New York"]) %}
        "result": { "location": "New York", "overflow_locations": ["London", "Oslo"] }
    {% elif pex_in_subnet(call_info.remote_address, nl_map["Sydney"]) %}
        "result": { "location": "Sydney", "overflow_locations": ["London"] }
    {% elif pex_in_subnet(call_info.remote_address, nl_map["London"]) %}
        "result": { "location": "London", "overflow_locations": ["Oslo", "New York"] }
    {% else %}
        "result": {{ suggested_media_overflow_locations | pex_to_json }}
    {% endif %}
}
```

---

## Interaction with external policy

If `overflow_locations` was set by an external policy response, **local policy cannot change
it** — once provided by external policy, the list is immutable for the call. Use external
policy or local policy to set this list, not both.
