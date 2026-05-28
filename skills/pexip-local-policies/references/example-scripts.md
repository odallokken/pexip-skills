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

---

## Field-tested patterns from production deployments

The next batch of examples come from real customer deployments rather than the Pexip
documentation, and cover more advanced situations: SfB/Teams integration, deterministic
PINs, media-playback chaining, time-based capacity protection, anti-overwrite ADP
patterns, etc. Each has a standalone copy/paste file in `../examples/`.

## 19. Enable overlay text for gateway calls by service tag

Set `enable_overlay_text=true` on gateway services whose tag begins with `overlaytext`.
Primarily useful for Skype/Teams gateway rules. See
[`overlay-text-for-gateway-by-tag.jinja2`](../examples/overlay-text-for-gateway-by-tag.jinja2).

## 20. Force overlay text everywhere

Single-line override that flips `enable_overlay_text` on for every resolved service —
no per-VMR config needed. See
[`force-text-overlay-everywhere.jinja2`](../examples/force-text-overlay-everywhere.jinja2).

## 21. Overlay text only for scheduled meetings

Same idea as §19 but gated by an alias regex (here `77NNNNNN`) so only scheduled
conferences get the overlay. See
[`text-overlay-for-scheduled.jinja2`](../examples/text-overlay-for-scheduled.jinja2).

## 22. Live captions for scheduled meetings

Sets `live_captions_enabled="yes"` on conferences whose alias starts with `88`
(scheduled-meeting prefix). Leaves other service types untouched. See
[`live-captions-for-scheduled.jinja2`](../examples/live-captions-for-scheduled.jinja2).

## 23. Auto-dial the owner's email as an MS-SIP ADP for scheduled meetings

When a scheduled meeting (alias prefix `88`) is owned by an `@pexip.com` user, append an
ADP that dials the owner's email over MS-SIP at chair role. Demonstrates reading
`service_config.primary_owner_email_address` populated from the AD-linked schedule. See
[`adp-mssip-for-scheduled-by-email.jinja2`](../examples/adp-mssip-for-scheduled-by-email.jinja2).

## 24. Append an ADP without overwriting existing ones

`pex_update({"automatic_participants": [...]})` REPLACES the list. To *add* to it without
disturbing ADPs already configured on the VMR, mutate `service_config.automatic_participants`
in place with `list.extend([...])`. The Jinja2 idiom is `{{ adps.extend([...]) or "" }}` —
the `or ""` is required because `extend` returns `None` and Jinja2 would otherwise emit
literal `None` into the output and break JSON. See
[`add-adp-without-overwriting.jinja2`](../examples/add-adp-without-overwriting.jinja2).

## 25. Dial out to the owner over MS-SIP on first guest entry only

Differentiate the host's own endpoint from guest joiners using a whitelist of
`(remote_alias, local_alias)` tuples (`exec`). Hosts join silently; guests trigger an MS-SIP
ADP towards `service_config.primary_owner_email_address`. Clear any pre-existing ADPs on
the VMR first. See
[`dial-out-to-s4b-on-guest.jinja2`](../examples/dial-out-to-s4b-on-guest.jinja2).

## 26. Ad-hoc VMR for Skype-for-Business Focus GRUU dial-in

When the dialed alias matches `sip:user@domain;gruu;opaque=app:conf:focus:id:<gruu>` (an
SfB meeting Focus URI), synthesise a brand-new VMR on the fly and add an MS-SIP ADP that
dials back into the SfB meeting. Video endpoints join the VMR (and see a real Pexip
mixed layout); SfB sees a single merged stream. Trade-offs documented inline in the
script (lower HW cost vs. no SfB-side roster control of VC endpoints). See
[`skype-meeting-adhoc-vmr.jinja2`](../examples/skype-meeting-adhoc-vmr.jinja2).

## 27. Override the "from" alias and display name on outbound gateway B-legs

Useful when interop with the far end needs a particular caller identity that does not
match the dial-in alias. Works only for straight point-to-point gateway calls — not VMRs
nor AVCMU-escalated calls. See
[`override-outbound-gateway-from.jinja2`](../examples/override-outbound-gateway-from.jinja2).

## 28. Rewrite a VMR into an outbound gateway leg by prefix

E.164-only systems can dial a numeric extension and reach the user's Skype URI: when the
dialed alias starts with `55`, flip `service_type` from `conference` to `gateway`, set
the MS-SIP proxy, and use `service_config.description` (the user's SfB URI populated by
AD provisioning) as the outbound `remote_alias`. See
[`vmr-as-gateway-by-prefix.jinja2`](../examples/vmr-as-gateway-by-prefix.jinja2).

## 29. PIN-less join only for a specific (endpoint, VMR) pair

Whitelist `(remote_alias, local_alias)` tuples in `exec`; only when both match are PINs
stripped and the conference forced host-only. Caveat: same as §6/§7 — guest-disconnect
behaviour can be quirky for PIN-less host-only conferences. See
[`pinless-by-endpoint-vmr-mapping.jinja2`](../examples/pinless-by-endpoint-vmr-mapping.jinja2).

## 30. Force audio-only for specific User-Agents

`pex_update({"call_type":"audio"})` when `call_info.vendor` is in a list of known-bad
WebRTC builds. Caveat: if a forced-audio UA is the FIRST participant on a Conferencing
Node, other participants on the same node may also be limited to audio. See
[`force-audio-only-by-vendor.jinja2`](../examples/force-audio-only-by-vendor.jinja2).

## 31. Bandwidth override for a list of dial-in endpoints

For named H.323 dial-in endpoints, set both `max_callrate_in` and `max_callrate_out`
explicitly (here, 4 Mbps). All other calls pass through unchanged. See
[`bandwidth-override-by-alias.jinja2`](../examples/bandwidth-override-by-alias.jinja2).

## 32. Trust selected domains for Teams CVI

Set `treat_as_trusted=True` on Teams CVI calls (Call Routing Rule tagged `TEAMS`) whose
SIP `remote_alias` ends with a known internal domain. Lets trusted callers skip the
Teams lobby. See
[`trust-domains-for-teams-cvi.jinja2`](../examples/trust-domains-for-teams-cvi.jinja2).

## 33. Country-coded audio steering for an Avaya gateway

Customer use-case: the calling endpoint name embeds a 2-letter country code; the policy
extracts it (two alternative regexes), looks up the matching steering code, and prepends
it to the destination alias so Avaya routes to the correct PSTN gateway. US/CA need no
prefix. Triggered only by routing rules tagged `dellAudio`. See
[`audio-steering-code-by-country.jinja2`](../examples/audio-steering-code-by-country.jinja2).

## 34. Priority-meeting capacity protection (weekday + time)

Every Monday 08:00–09:00 Helsinki time, redirect every non-priority call into a short
media-playback announcement and disconnect — preserving port-license and transcoding
capacity for the all-hands meeting (which keeps its tag `allowmeeting` and is allowed
through). The Zeller's-congruence-style block computes the weekday from `pex_now()`.
Requires a media-playback source and playlist both named `allhands_meeting_message`.
See [`priority-meeting-by-weekday.jinja2`](../examples/priority-meeting-by-weekday.jinja2).

## 35. Media playback before a Teams gateway call

For inbound dial-ins of the form `88<9-13 digits>@pex.vc` hitting a rule whose tag
contains `greg`, play the `greg_dod` playlist and on completion `transfer` to alias
`89<teams_id>` (the actual Teams gateway routing rule). Pattern is reusable for any
"play something before joining" workflow. See
[`media-playback-before-teams.jinja2`](../examples/media-playback-before-teams.jinja2).

## 36. Media playback before a VMR (preserving PIN + IDP context)

For aliases starting with `sso`, route into a media-playback service that inherits the
target VMR's PIN, guest PIN, host IDP group and `allow_guests` setting — so the caller
authenticates against the playback service, and the authentication context survives the
`on_completion` transfer into the real VMR. Assumes the VMR has a second alias dedicated
to this "playback first" entry path. See
[`media-playback-before-vmr.jinja2`](../examples/media-playback-before-vmr.jinja2).

## 37. Deterministic-PIN scheduled meeting with playback + IDP (V.2)

Combined recipe that:

- Generates two PINs deterministically from a shared secret + the numeric alias
  (`pex_hash | pex_tail` and `pex_hash | pex_head`). One acts as Host PIN, the other as
  a hidden Guest PIN (decoy). Roles are flipped depending on `call_info.location` so
  trusted-location callers get the host role and untrusted callers get the guest role.
- For untrusted callers from `VIDEODMZ`, first routes into a `media_playback` service
  that requires IDP login (for WebRTC/api) and then transfers to the real conference
  using an `88.` prefix as a "playback completed" marker.
- VMR is created `locked=true` so hosts must explicitly admit guests.

The Exchange Joining Instructions template must use the *same* `shared_secret` to render
the PIN into the meeting invitation. See
[`scheduled-meeting-pin-with-playback-idp.jinja2`](../examples/scheduled-meeting-pin-with-playback-idp.jinja2).

## 38. Do not transcode (by service tag)

One-liner: when `service_tag == "do-not-transcode"`, set `transcoding_enabled=False`.
Apply on SIP-to-SIP gateway rules where Pexip should pass media through unchanged. See
[`do-not-transcode-by-tag.jinja2`](../examples/do-not-transcode-by-tag.jinja2).
