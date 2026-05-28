# Example local policy scripts — annotated

All examples below come straight from the Pexip docs ("Example local policy scripts") with
brief explanations. Each one is also dropped as a standalone `.jinja2` file in the sibling
`examples/` folder so you can copy/paste them directly into a policy profile.

> All scripts assume the canonical structure: a pass-through default branch that emits the
> existing `service_config` unchanged, plus a guarded branch that adds the new behaviour.

---

## 1. Minimum pass-through service-config script

The "no changes" baseline. Allows whatever Pexip would have allowed; rejects whatever Pexip
would have rejected. Start every new service-config script from here.

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

## 2. Minimum pass-through media-location script

```jinja2
{
  "result" : {{suggested_media_overflow_locations|pex_to_json}}
}
```

## 3. Pass-through service-config with `pex_debug_log`

Adds nothing functional but writes the entire `call_info` to the support log so you can
inspect what Pexip actually delivers for different call types. *Remove the debug line after
you are done.*

```jinja2
{
    {{pex_debug_log("Call information ", call_info) }}
    {% if service_config %}
        "action" : "continue",
        "result" : {{service_config|pex_to_json}}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 4. Unconditionally nominate media locations

Forces Oslo as the principal, with London and New York as overflow, for every call regardless
of where signalling lands. Useful as a sanity-check baseline for media routing.

```jinja2
{
    "result" : {
        "location" : "Oslo",
        "primary_overflow_location" : "London",
        "secondary_overflow_location" : "New York"
    }
}
```

## 5. Nominate a media location for RTMP streaming

Pins outbound RTMP to a DMZ location; everything else uses Pexip's default.

```jinja2
{
    {% if
        call_info.protocol == "rtmp" and
        call_info.call_direction == "dial_out"
    %}
        "result" : {
            "location" : "DMZ",
            "primary_overflow_location" : "Location B",
            "secondary_overflow_location" : "Location C"
        }
    {% else %}
        "result" : {{suggested_media_overflow_locations|pex_to_json}}
    {% endif %}
}
```

## 6. Remove PIN based on inbound location

Strips PINs (and disables guest distinction) for participants whose signalling arrived at the
Conferencing Nodes in "Location A" or "Location C" — typically your internal locations.

```jinja2
{
    {% if service_config %}
        "action" : "continue",
        {% if call_info.location == "Location A" or call_info.location == "Location C" %}
            "result" : {{service_config|pex_update({"pin":"", "guest_pin":"", "allow_guests" : False})|pex_to_json}}
        {% else %}
            "result" : {{service_config|pex_to_json}}
        {% endif %}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 7. Remove PIN for registered devices

Skip the PIN gate for any device registered to a Conferencing Node.

```jinja2
{
    {% if service_config %}
        "action" : "continue",
        {% if call_info.registered %}
            "result" : {{service_config|pex_update({"pin":"", "guest_pin":"", "allow_guests" : False})|pex_to_json}}
        {% else %}
            "result" : {{service_config|pex_to_json}}
        {% endif %}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 8. Inject an Automatically Dialed Participant into every conference

Adds a single ADP to every conference. **Overrides** any existing ADPs in `service_config`
(`pex_update` replaces the value of the key wholesale).

```jinja2
{
    {% if service_config %}
        {{pex_debug_log("service_config=", service_config) }}
        "action" : "continue",
        "result" : {{service_config|pex_update({
            "automatic_participants" : [{
                "remote_alias":"participant@domain.com",
                "local_alias":"policyuser@domain.com",
                "local_display_name":"Local policy ADP",
                "description":"Dial out to an ADP",
                "protocol":"sip",
                "role":"chair",
                "system_location_name":"Location A"
            }]
        })|pex_to_json}}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 9. Reject suspicious User-Agents (anti-scanner)

Drops calls from common VOIP-scanner User-Agents (sipvicious, friendly-scanner, etc.). Adapt
the list to your environment.

```jinja2
{
    {# NOTE: not all of these UAs are always malicious — they have legitimate uses. Adapt as required. #}
    {# NOTE: "cisco" here is NOT used by genuine Cisco gear — it is an openH323 scanner pretending to be one. #}
    {% set suspect_uas = [
        "cisco", "friendly-scanner", "sipcli", "sipvicious", "sip-scan",
        "sipsak", "sundayddr", "iWar", "CSipSimple", "SIVuS", "Gulp", "sipv",
        "smap", "friendly-request", "VaxIPUserAgent", "VaxSIPUserAgent",
        "siparmyknife", "Test Agent", "PortSIP VoIP SDK"
    ] %}

    {% if service_config %}
        {% if call_info.vendor in suspect_uas %}
            "action" : "reject",
            "result" : {}
        {% else %}
            "action" : "continue",
            "result" : {{service_config|pex_to_json}}
        {% endif %}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 10. Subnet-based location routing

Apply different rules per network region. The `nl_map` declaration is reusable — drop your
own subnet ↔ region mapping in.

```jinja2
{% set nl_map = {
    "Oslo"     : ["10.47.0.0/16", "10.147.0.0/16", "10.247.200.0/24"],
    "New York" : ["10.1.0.0/16",  "10.201.5.0/23"],
    "Sydney"   : ["10.61.0.0/16"],
    "London"   : ["10.44.0.0/16"]
} %}
{% if pex_in_subnet(call_info.remote_address, nl_map["Oslo"]) %}
    {# Apply Norwegian rules #}
{% elif pex_in_subnet(call_info.remote_address, nl_map["New York"]) %}
    {# Apply American rules #}
{% else %}
    {# Default #}
{% endif %}
```

## 11. Lock a conference when the first participant joins (by service tag)

Looks at `service_config.service_tag`; if it starts with `"locked"`, the conference is
locked on creation. Set the Service tag of the VMRs you want auto-locked.

```jinja2
{
    {% if service_config %}
        "action" : "continue",
        {% if service_config.service_type == "conference" and service_config.service_tag.startswith("locked") %}
            "result" : {{service_config|pex_update({"locked":True })|pex_to_json}}
        {% else %}
            "result" : {{service_config|pex_to_json}}
        {% endif %}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 12. Time-of-day theme

Uses `pex_now()` to pick a theme based on London local time.

```jinja2
{
    {% set now = pex_now("Europe/London") %}
    {% if service_config %}
        "action" : "continue",
        {% if now.hour < 12 %}
            "result" : {{service_config|pex_to_json}}
        {% elif now.hour < 18 %}
            "result" : {{service_config|pex_update({"ivr_theme_name":"Afternoon theme"})|pex_to_json}}
        {% else %}
            "result" : {{service_config|pex_update({"ivr_theme_name":"Evening theme"})|pex_to_json}}
        {% endif %}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 13. Limit VMR access to registered devices only

Allows calls to non-conference services as normal; rejects unregistered devices for VMR
calls. Add `and call_info.location == "..."` to restrict the limitation to a single location.

```jinja2
{
    {% if service_config %}
        {% if service_config.service_type == "conference" %}
            {% if call_info.registered %}
                "action" : "continue",
                "result" : {{service_config|pex_to_json}}
            {% else %}
                "action" : "reject",
                "result" : {}
            {% endif %}
        {% else %}
            "action" : "continue",
            "result" : {{service_config|pex_to_json}}
        {% endif %}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 14. Disable encryption when dialing out to a specific device

For interop with legacy PBXs that do not support SRTP.

```jinja2
{
    {% if service_config %}
        {% if call_info.call_direction == 'dial_out' %}
            {% if 'sip:user@example.com' in call_info.remote_alias %}
                "action" : "continue",
                "result" : {{service_config|pex_update({"crypto_mode":"off"})|pex_to_json}}
            {% else %}
                "action" : "continue",
                "result" : {{service_config|pex_to_json}}
            {% endif %}
        {% else %}
            "action" : "continue",
            "result" : {{service_config|pex_to_json}}
        {% endif %}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 15. Teams-like layout for Teams gateway calls

Apply `"view": "teams"` to all Teams gateway calls. Swap to `"teams_focus"` for the
speaker-focused layout. Use `service_config.name == "<routing rule name>"` to scope to a
specific Call Routing Rule.

```jinja2
{
    {% if service_config and service_config.service_type == "gateway" and service_config.called_device_type == "teams_conference" %}
        "action" : "continue",
        "result" : {{service_config|pex_update({"view":"teams"})|pex_to_json}}
    {% elif service_config %}
        "action" : "continue",
        "result" : {{service_config|pex_to_json}}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

> Notes on Teams layouts: cannot be themed; do not honour overlay/active-speaker toggles;
> Teams-like uses Adaptive Composition resource budget, speaker-focused uses classic budget;
> only for Teams gateway calls.

## 16. Adaptive Composition + names for Google Meet gateway calls

```jinja2
{
    {% if service_config and service_config.service_type == "gateway" and service_config.called_device_type == "gms_conference" %}
        "action" : "continue",
        "result" : {{service_config|pex_update({"enable_overlay_text": true, "view":"five_mains_seven_pips"})|pex_to_json}}
    {% elif service_config %}
        "action" : "continue",
        "result" : {{service_config|pex_to_json}}
    {% else %}
        "action" : "reject",
        "result" : {}
    {% endif %}
}
```

## 17. Route incoming calls directly into a breakout room

Requires the main `"gamesroom"` VMR to exist in Pexip's database. Two extra aliases
(`teama@example.com`, `teamb@example.com`) — which must **not** be aliases of the main VMR —
synthesise a breakout-room service configuration on the fly.

```jinja2
{
    {% if service_config %}
        "action" : "continue",
        "result" : {{service_config|pex_to_json}}
    {% else %}
        {% if call_info.local_alias == "teama@example.com" %}
            "action" : "continue",
            "result" : {"name":"gamesroom", "breakout_uuid":"00000000-0000-0000-0000-000000000001",  "breakout":true, "breakout_rooms":false, "breakout_name" : "Team A", "breakout_description": "Team A room", "end_action":"transfer", "end_time":0, "service_tag":"gamesroom_teama", "service_type":"conference", "allow_guests": true, "pin": "4567"}
        {% elif call_info.local_alias == "teamb@example.com" %}
            "action" : "continue",
            "result" : {"name":"gamesroom", "breakout_uuid":"00000000-0000-0000-0000-000000000002",  "breakout":true, "breakout_rooms":false, "breakout_name" : "Team B", "breakout_description": "Team B room", "end_action":"transfer", "end_time":0, "service_tag":"gamesroom_teamb", "service_type":"conference", "allow_guests": true, "pin": "4567"}
        {% else %}
            "action" : "reject",
            "result" : {}
        {% endif %}
    {% endif %}
}
```

## 18. Mask PSTN telephone numbers (participant policy)

Anonymises the alias and display name of any caller whose user-part parses as an integer
(i.e. a phone number).

```jinja2
{% set userpart = call_info.remote_alias|lower|pex_regex_replace("^(sip:|sips:|h323:)","") |pex_regex_replace('@.+','') %}

{
    "action" : "continue",
    {% if userpart|int != 0 %}
        "result" : {
            "remote_alias": "0000",
            "remote_display_name": "Telephone User"
        }
    {% else %}
        "result" : {}
    {% endif %}
}
```
