# Local policy variables — reference

Local policy scripts run with a small set of system-supplied variables. Variable names are
case-sensitive and must be spelled exactly as below. Variables are Python dicts (very close
to, but not identical to, JSON) — use the `pex_to_json` filter when embedding them in a
response.

> **Source:** docs.pexip.com — *Writing local policy scripts > Supported variables*.

## At-a-glance

| Variable | Available in | Mutable? | Purpose |
|---|---|---|---|
| `call_info` | service config, participant, media location | No | Facts about the call/request. |
| `service_config` | service config, participant, media location | Yes (via `pex_update`) | Existing service configuration from DB or external policy. May be `None` for service-config scripts. |
| `participant` | participant only | Yes | Participant about to join. |
| `suggested_media_overflow_locations` | media location only | Yes | Existing media-location decision. |

---

## 1. `service_config`

Holds the existing service configuration. Reference fields with the `service_config.` prefix
(e.g. `service_config.pin`, `service_config.service_type`).

`service_config.service_type` is one of:

- `"conference"` — Virtual Meeting Room (VMR)
- `"lecture"` — Virtual Auditorium
- `"gateway"` — Infinity Gateway call
- `"two_stage_dialing"` — Virtual Reception
- `"media_playback"` — Media Playback Service
- `"test_call"` — Test Call Service

For service-configuration scripts, `service_config` can be `None` (e.g. the dialed alias is
not configured). Always check `{% if service_config %}` before referencing fields, *unless*
you intentionally want to synthesise a service from scratch (e.g. dynamic breakout rooms by
alias).

Full per-`service_type` field tables live in `service-config-response.md` — the same fields
appear both in the input variable and in the response object.

Example `service_config` (shown as JSON):

```json
{
  "allow_guests": true,
  "automatic_participants": [
    {
      "description": "Dial out to VMR owner",
      "local_alias": "meet.alice@example.com",
      "local_display_name": "Alice's VMR",
      "protocol": "sip",
      "remote_alias": "sip:alice@example.com",
      "role": "chair",
      "system_location_name": "London"
    }
  ],
  "description": "Alice Jones personal VMR",
  "enable_overlay_text": true,
  "guest_pin": "567890",
  "name": "Alice Jones",
  "pin": "123456",
  "service_tag": "abcd1234",
  "service_type": "conference"
}
```

> `pex_debug_log` may print additional internal fields beyond those listed in the docs. Those
> are for diagnostic interest only and may change between releases — do not depend on them.

---

## 2. `suggested_media_overflow_locations`

Only available in media-location scripts. Fields:

| Field | Description |
|---|---|
| `location` | The principal location that will handle the media. |
| `primary_overflow_location` | First fallback if `location` is at capacity. |
| `secondary_overflow_location` | Second fallback. |

Example:

```json
{
  "location": "New York",
  "primary_overflow_location": "Paris",
  "secondary_overflow_location": "Peckham"
}
```

The **response** shape is slightly different — it uses a `location` plus an `overflow_locations`
*list* (see `media-location-response.md`).

---

## 3. `call_info` — facts about the call

Read-only. Always reference with the `call_info.` prefix. The table below shows which fields
are populated for which request type. Identical fields to those in the external policy API.

| Field | Service config | Participant | Media location | Description |
|---|:---:|:---:|:---:|---|
| `bandwidth` | ✓ | ✓ | ✓ | Max requested bandwidth (meaningful only on inbound). |
| `breakout_uuid` | ✓ |  |  | UUID of a breakout room (when applicable). |
| `call_direction` | ✓ | ✓ | ✓ | `"dial_in"`, `"dial_out"`, `"non_dial"` (LRQ / SUBSCRIBE / OPTIONS). |
| `call_tag` | ✓ | ✓ | ✓ | Optional call tag. |
| `display_count` | ✓ | ✓ |  | Endpoint screens or, in service config, the screens signalled by a Teams Room joining via OTJ. |
| `external_policy_disposition` | ✓ | ✓ |  | Result of any preceding external-policy request: service config `"CONTINUE"`/`"REJECT"`/`"REDIRECT"`; participant `"CONTINUE"`/`"REJECT"`. Only present if external policy was enabled, succeeded, and returned a valid action. |
| `has_authenticated_display_name` | ✓ | ✓ | ✓ | Was display name provided & authenticated by an IdP? |
| `idp_uuid` | ✓ | ✓ | ✓ | UUID of the IdP that authenticated the participant. |
| `local_alias` | ✓ | ✓ | ✓ | The incoming/dialed alias. The primary routing input for service-config scripts. |
| `location` | ✓ | ✓ | ✓ | System location of the Conferencing Node handling the request. |
| `ms_subnet` † | ✓ | ✓ | ✓ | Sender subnet, JSON list e.g. `["10.47.5.0"]`. |
| `node_ip` | ✓ | ✓ | ✓ | IP of the Conferencing Node making the request. |
| `p_asserted_identity` † | ✓ | ✓ | ✓ | Authenticated sender identity, JSON list e.g. `['"Alice"<sip:alice@example.com>']`. |
| `previous_service_name` | ✓ | ✓ | ✓ | Service name before any call transfer. |
| `protocol` | ✓ | ✓ | ✓ | `"api"`, `"webrtc"`, `"sip"`, `"rtmp"`, `"h323"`, `"teams"`, `"mssip"`. (Pexip apps dialling in report `"api"`.) |
| `proxy_node_address` |  |  | ✓ | Proxying Edge Node address (if any). |
| `proxy_node_location` |  |  | ✓ | Proxying Edge Node location (if any). |
| `pseudo_version_id` | ✓ | ✓ | ✓ | Software build number. |
| `registered` | ✓ | ✓ | ✓ | True if the remote is a registered device. |
| `remote_address` | ✓ | ✓ | ✓ | Remote IP. |
| `remote_alias` | ✓ | ✓ | ✓ | Remote alias (may include scheme, e.g. `sip:alice@example.com`). |
| `remote_display_name` | ✓ | ✓ | ✓ | Remote display name. |
| `remote_port` | ✓ | ✓ | ✓ | Remote IP port. |
| `service_name` ‡ | (outbound + breakouts only) | ✓ | ✓ | Service name (matches `name` returned from the original service-config response). |
| `service_tag` †† | (outbound only) | ✓ | ✓ | Service tag. |
| `supports_direct_media` | ✓ | ✓ | ✓ | Whether the service supports direct media. |
| `teams_tenant_id` | ✓ | ✓ | ✓ | Microsoft Teams tenant ID (Teams Rooms SIP/H.323 inbound). |
| `telehealth_request_id` ◊ | ✓ | ✓ | ✓ | Epic telehealth call ID. |
| `third_party_passcode` | ✓ |  |  | Passcode from Teams Room calendaring (extracted from the invite). |
| `trigger` | ✓ | ✓ | ✓ | `"web"`, `"web_avatar_fetch"`, `"invite"`, `"options"`, `"subscribe"`, `"setup"`, `"arq"`, `"lrq"`, `"two_stage_dialing"`, `"teams"`, `"unspecified"`. |
| `unique_service_name` ‡ | (outbound + breakouts only) | ✓ | ✓ | Unique per-call/per-breakout name: gateway = rule name + unique suffix; breakout = `<vmr>_breakout_<uuid>`. |
| `vendor` | ✓ | ✓ | ✓ | Remote system details (User-Agent for SIP, browser/OS for soft clients). |
| `version_id` | ✓ | ✓ | ✓ | Software version number. |

> † Only present if triggered by a SIP message carrying the header. Note the **local-policy
> renaming**: in local policy they are `ms_subnet` and `p_asserted_identity` (lowercase +
> underscores) — in the external policy API they are `ms-subnet` and `p_Asserted-Identity`.
> Values arrive as **JSON lists**, not bare strings.
>
> ‡ Only in outbound call requests and breakout-room requests.
>
> †† Only in outbound call requests.
>
> ◊ Only in Epic telehealth calls.

Example `call_info` (as JSON):

```json
{
  "bandwidth": 0,
  "call_direction": "dial_in",
  "call_tag": "wxyz789",
  "local_alias": "meet.alice.vmr@example.com",
  "location": "New York",
  "node_ip": "10.55.55.101",
  "protocol": "api",
  "pseudo_version_id": "36358.0.0",
  "remote_address": "10.55.55.250",
  "remote_alias": "bob@example.com",
  "remote_display_name": "Bob T. User",
  "remote_port": 64703,
  "trigger": "web",
  "vendor": "Mozilla/5.0 (X11; Linux x86_64) Chrome/54.0.2840.100 Safari/537.36",
  "version_id": "16"
}
```

---

## 4. `participant` — the joining participant (participant policy only)

| Field | Description |
|---|---|
| `auth_type` | `"credentials"` or `"sso"`. |
| `call_uuid` | Unique ID of the participant's call. |
| `display_count` | Endpoint screen count (signalled by endpoint or from theme `vendordata.json`). |
| `idp_attributes` | Dict of custom IdP attributes. Configured per IdP under **Users & Devices > Identity Providers > Advanced options**. Attribute names containing hyphens or spaces must be accessed with `.get()` — e.g. `participant.idp_attributes.get("X-wing pilot")`. |
| `participant_type` | `"standard"`, `"api"`, or `"api_host"`. WebRTC join produces two requests: first `"api"` then `"standard"`. `"api_host"` starts the conference if the participant is a Host. |
| `participant_uuid` | UUID for the participant. |
| `preauthenticated_role` | `"chair"`, `"guest"`, or **`None`** (test for `is None`, not the string `"None"`). For WebRTC, participant policy runs **after** any PIN/SSO step. For SIP/H.323 it runs **before** PIN, so the role is typically `None`. Local policy runs after any external policy, which may already have changed the role. |
| `receive_from_audio_mix` | The audio mix this participant is receiving (e.g. `"main"`). |
| `remote_alias` | May have been modified by external policy. The *original* endpoint value is in `call_info.remote_alias`. |
| `remote_display_name` | May have been modified by external policy. The *original* endpoint value is in `call_info.remote_display_name`. |
| `send_to_audio_mixes` | List of `{"mix_name": "...", "prominent": <bool>}` dicts. |

Example `participant` (as JSON):

```json
{
  "call_uuid": "69434387-e444-411c-afd4-dcc25bf328d3",
  "idp_attributes": {
    "group": "super-admin",
    "locale": "en_GB"
  },
  "participant_type": "standard",
  "participant_uuid": "91664d87-28e4-444b-b6f1-a3816547290",
  "preauthenticated_role": "chair",
  "remote_alias": "alice.jones@example.com",
  "remote_display_name": "Alice Jones"
}
```
