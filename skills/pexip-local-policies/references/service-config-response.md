# Service configuration response — reference

The response to a service-configuration policy script has the basic shape:

```jinja2
{
    "action": "reject|redirect|continue",
    "result": { <service_configuration_data> }
}
```

- `"reject"` — Pexip rejects the call (returns "conference not found").
- `"redirect"` — Pexip sends a SIP `302 Redirect` to the requesting endpoint.
  `result` **must** be `{"new_alias": "sip:alias@example.com"}`.
- `"continue"` — use the data supplied in `result`.

The fields required in `result` depend on `service_type`:

- `"conference"` — VMR
- `"lecture"` — Virtual Auditorium
- `"gateway"` — Infinity Gateway call
- `"two_stage_dialing"` — Virtual Reception
- `"media_playback"` — Media Playback Service
- `"test_call"` — Test Call Service

> Note: the response configures the **service**, so a given service should return the same
> configuration on every request. The only fields that can vary per participant are:
> `guest_identity_provider_group`, `host_identity_provider_group`, `pin`, `guest_pin`,
> `max_callrate_in`, `max_callrate_out`. To vary other participant-level properties, use
> **participant** policy.

---

## VMR (`service_type: "conference"`) / Virtual Auditorium (`service_type: "lecture"`)

| Field | Req | Type | Description |
|---|:---:|---|---|
| `name` | ✓ | string | Service name. Pexip uses this as `service_name` in subsequent requests. For a breakout, this is the **main VMR name**. |
| `service_tag` | ✓ | string | Unique tracking identifier. |
| `service_type` | ✓ | string | `"conference"` or `"lecture"`. |
| `allow_guests` |   | bool | If true, distinguish Host vs Guest. Requires `pin`; `guest_pin` optional. Default `false`. |
| `automatic_participants` |   | list | Auto-dial participants. See [ADP fields](#adp). |
| `breakout_rooms` |   | bool | Whether breakout rooms are enabled. Default `false`. |
| `bypass_proxy` |   | bool | If true and signalling went through a Proxying Edge Node, media is sent direct to the Transcoding node. Default `false`. |
| `call_type` |   | string | `"video"` (default), `"video-only"`, `"audio"`. |
| `crypto_mode` |   | string | `null` (global default), `"besteffort"`, `"on"`, `"off"`. |
| `denoise_enabled` |   | bool | Per-VMR Denoise. Default `false`. |
| `description` |   | string | Description. |
| `direct_media` |   | string | `"best_effort"`, `"always"`, `"never"`. Default `"never"`. Not supported with breakout rooms. |
| `direct_media_notification_duration` |   | int | 0–30 seconds. |
| `enable_active_speaker_indication` |   | bool | Default `false`. |
| `enable_chat` |   | string | `"default"`, `"yes"`, `"no"`. |
| `enable_overlay_text` |   | bool | Show participant name overlays. Default `false`. |
| `force_presenter_into_main` *(lecture only)* |   | bool | Lock presenting Host into main video. Default `false`. |
| `guest_pin` |   | string | Guest PIN. |
| `guest_view` *(lecture only)* |   | string | Layout for Guests (see Layout names below). Default `"one_main_seven_pips"`. |
| `guests_can_present` |   | bool | Default `true`. |
| `guests_can_see_guests` |   | string | `"no_hosts"`, `"always"`, `"never"`. Auditorium only. Default `"no_hosts"`. |
| `guest_identity_provider_group` |   | string | IdP set offered to Guests. |
| `host_identity_provider_group` |   | string | IdP set offered to Hosts. |
| `host_view` *(lecture only)* |   | string | Layout for Hosts. Default `"one_main_seven_pips"`. |
| `ivr_theme_name` |   | string | Theme. Default = Pexip default theme. |
| `live_captions_enabled` |   | string | `"default"`, `"yes"`, `"no"`. |
| `live_captions_language` |   | string | Caption language. |
| `local_display_name` |   | string | Display name of the calling alias. |
| `locked` |   | bool | Lock on creation. No effect once running. Default `false`. |
| `max_callrate_in` |   | int | 128 000–8 192 000 bps. |
| `max_callrate_out` |   | int | 128 000–8 192 000 bps. |
| `max_pixels_per_second` |   | string | `null`, `"sd"`, `"hd"`, `"fullhd"`. |
| `mute_all_guests` |   | bool | Default `false`. |
| `non_idp_participants` |   | string | `"disallow_all"` (default) or `"allow_if_trusted"`. |
| `participant_limit` |   | int | Max participants. |
| `pin` |   | string | Host PIN. |
| `pinning_config` |   | string | Pinning configuration name from the theme. |
| `prefer_ipv6` |   | string | `"default"`, `"yes"`, `"no"`. |
| `primary_owner_email_address` |   | string | VMR owner email. |
| `softmute_enabled` |   | bool | Per-VMR Softmute. Default `false`. |
| `source_language` |   | string | AIMS source-audio language. |
| `view` *(conference only)* |   | string | Layout for all participants. Default `"one_main_seven_pips"`. |

### Layout names (`view`, `host_view`, `guest_view`)

`one_main_zero_pips`, `one_main_seven_pips`, `one_main_twentyone_pips`,
`two_mains_twentyone_pips`, `one_main_thirtythree_pips`, `four_mains_zero_pips`,
`nine_mains_zero_pips`, `sixteen_mains_zero_pips`, `twentyfive_mains_zero_pips`,
`five_mains_seven_pips` (Adaptive Composition), `one_main_one_pip`, `one_main_nine_around`,
`one_main_twelve_around`, `two_mains_eight_around`, `teams` (Teams-only),
`teams_focus` (Teams-only).

### Breakout-room fields (only when the result represents a breakout)

| Field | Req | Type | Description |
|---|:---:|---|---|
| `breakout` | ✓ | bool | `true` to mark this result as a breakout. When `true`, `breakout_rooms` must be `false`. `name` must be the main VMR name. |
| `breakout_uuid` | ✓ | string | UUID string `XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`. Use `pex_hash` of a stable input for deterministic per-caller breakouts. |
| `breakout_name` | ✓ | string | Display name of the breakout room. |
| `breakout_uuids` |   | list | (Main-room responses) list of breakout UUIDs to be created — Pexip then triggers another policy lookup per breakout with `service_name = "<main_room_name>_breakout_<breakout_uuid>"`, `unique_service_name = service_name`, and `breakout_uuid = breakout_uuid`. Each such lookup must return a breakout configuration block. |
| `breakout_description` |   | string | Long description. |
| `breakout_guests_allowed_to_leave` |   | bool | Default `false`. |
| `end_action` |   | string | `"transfer"` (default) or `"disconnect"` when the timer expires / breakout closes. |
| `end_time` |   | int | Seconds since epoch. `0` = no automatic end (default). |

A breakout block can also override any of the standard VMR fields (theme, overlays, chat, etc.).

### <a id="adp"></a>Automatically-dialed participants (`automatic_participants[]`)

| Field | Req | Type | Description |
|---|:---:|---|---|
| `local_alias` | ✓ | string | "From" alias the recipient would return the call to. |
| `protocol` | ✓ | string | `"h323"`, `"sip"`, `"mssip"`, `"rtmp"`. |
| `remote_alias` | ✓ | string | Endpoint to call. |
| `role` | ✓ | string | `"chair"` or `"guest"`. |
| `call_type` |   | string | `"video"` (default), `"video-only"`, `"audio"`. |
| `description` |   | string | Informational only. |
| `dtmf_sequence` |   | string | DTMF to send after the call is up. |
| `keep_conference_alive` |   | string | `"keep_conference_alive"`, `"keep_conference_alive_if_multiple"` (default), `"keep_conference_alive_never"`. |
| `local_display_name` |   | string | Display name. |
| `presentation_url` |   | string | RTMP only — separate destination for presentation. |
| `remote_display_name` |   | string | Friendly name. |
| `routing` |   | string | `"manual"` (default) or `"routing_rule"`. |
| `streaming` |   | bool | Mark as streaming/recording device. Default `false`. |
| `system_location_name` |   | string | Location of the Conferencing Node placing the call. |

---

## Infinity Gateway (`service_type: "gateway"`)

| Field | Req | Type | Description |
|---|:---:|---|---|
| `local_alias` | ✓ | string | Calling "from" alias. |
| `name` | ✓ | string | Unique service name (avoid collisions between calls). |
| `outgoing_protocol` | ✓ | string | `"h323"`, `"sip"`, `"mssip"`, `"rtmp"`, `"gms"`, `"teams"`. |
| `remote_alias` | ✓ | string | Endpoint to call. |
| `service_tag` | ✓ | string | Unique tracking identifier. |
| `service_type` | ✓ | string | `"gateway"`. |
| `bypass_proxy` |   | bool | See above. Default `false`. |
| `call_type` |   | string | `"video"` (default), `"video-only"`, `"audio"`, `"auto"`. |
| `called_device_type` |   | string | `"external"` (default), `"registration"`, `"mssip_conference_id"`, `"mssip_server"`, `"gms_conference"`, `"teams_conference"`, `"teams_user"`, `"telehealth_profile"`. |
| `crypto_mode` |   | string | `null`, `"besteffort"`, `"on"`, `"off"`. |
| `denoise_audio` |   | bool | Google Meet integrations only. Default `true`. |
| `description` |   | string |   |
| `dtmf_sequence` |   | string | DTMF after the call starts. |
| `enable_active_speaker_indication` |   | bool | Default `false`. |
| `enable_overlay_text` |   | bool | Default `false`. |
| `external_participant_avatar_lookup` |   | string | Teams only: `"default"`, `"yes"`, `"no"`. |
| `gms_access_token_name` |   | string | Access token name for Google Meet ID resolution. Pick trusted/untrusted appropriately. |
| `h323_gatekeeper_name` |   | string | Gatekeeper name (DNS if omitted). |
| `ivr_theme_name` |   | string |   |
| `live_captions_enabled` |   | string | `"default"`, `"yes"`, `"no"`. |
| `live_captions_language` |   | string |   |
| `local_display_name` |   | string |   |
| `max_callrate_in` |   | int | bps. |
| `max_callrate_out` |   | int | bps. |
| `max_pixels_per_second` |   | string | `null`, `"sd"`, `"hd"`, `"fullhd"`. |
| `mssip_proxy_name` |   | string | SfB server name. |
| `outgoing_location_name` |   | string | Conferencing-Node location to place the call from. |
| `prefer_ipv6` |   | string | `"default"`, `"yes"`, `"no"`. |
| `sip_proxy_name` |   | string | SIP proxy name. |
| `source_language` |   | string |   |
| `stun_server_name` |   | string | STUN server name for ICE-enabled endpoints. |
| `teams_fit_to_frame` |   | string | Teams only: `"yes"`/`"no"`. Default `"no"`. |
| `teams_proxy_name` |   | string | Teams Connector name. |
| `transcoding_enabled` |   | bool | SIP-to-SIP transcoding. Default `true`. Non-transcoded calls are H.264 only and Pexip dictates bandwidth. |
| `treat_as_trusted` |   | bool | Indicates the target may treat the caller as part of the target organisation. |
| `turn_server_name` |   | string | TURN server name for ICE-enabled endpoints. |
| `view` |   | string | Layout (see VMR `view` for valid names). |

> The `name` value for gateway services **must** match a configured name in Pexip (location,
> gatekeeper, SIP proxy, etc.) when used as such — name validation is strict.

---

## Virtual Reception (`service_type: "two_stage_dialing"`)

| Field | Req | Type | Description |
|---|:---:|---|---|
| `name` | ✓ | string | Service name. |
| `service_tag` | ✓ | string |   |
| `service_type` | ✓ | string | `"two_stage_dialing"`. |
| `bypass_proxy` |   | bool | Default `false`. |
| `call_type` |   | string | `"video"` (default), `"video-only"`, `"audio"`. |
| `crypto_mode` |   | string | `null`, `"besteffort"`, `"on"`, `"off"`. |
| `description` |   | string |   |
| `gms_access_token_name` |   | string | Trusted/untrusted does not matter for Virtual Receptions. |
| `ivr_theme_name` |   | string |   |
| `local_display_name` |   | string |   |
| `match_string` |   | string | Regex for the entered alias (blank = any). |
| `max_callrate_in` |   | int | bps. |
| `max_callrate_out` |   | int | bps. |
| `max_pixels_per_second` |   | string | `null`, `"sd"`, `"hd"`, `"fullhd"`. |
| `mssip_proxy_name` |   | string | SfB server for Conference ID resolution. |
| `post_match_string` |   | string | Post-lookup regex. Typically `(.*)`. |
| `post_replace_string` |   | string | Pair with `post_match_string` to rewrite the meeting code into a routable alias. |
| `replace_string` |   | string | Regex replace for the entered alias (paired with `match_string`). |
| `system_location_name` |   | string | Where to perform SfB/Teams/Meet code lookups. Recommended in non-`regular` types. |
| `teams_proxy_name` |   | string | Teams Connector name. |
| `two_stage_dial_type` |   | string | `"regular"` (default), `"mssip"`, `"gms"`, `"teams"`. |

---

## Media Playback (`service_type: "media_playback"`)

| Field | Req | Type | Description |
|---|:---:|---|---|
| `name` | ✓ | string |   |
| `media_playlist_name` | ✓ | string | Playlist name. |
| `service_type` | ✓ | string | `"media_playback"`. |
| `service_tag` | ✓ | string |   |
| `allow_guests` |   | bool | Default `false`. |
| `description` |   | string |   |
| `guest_pin` |   | string |   |
| `guest_identity_provider_group` |   | string |   |
| `host_identity_provider_group` |   | string |   |
| `ivr_theme_name` |   | string |   |
| `max_callrate_in` |   | int |   |
| `max_callrate_out` |   | int |   |
| `max_pixels_per_second` |   | string |   |
| `non_idp_participants` |   | string | `"disallow_all"` (default) or `"allow_if_trusted"`. |
| `on_completion` |   | object | `{"disconnect": true}` or `{"transfer": {"conference": "<alias>", "role": "<role>"}}`. |
| `pin` |   | string |   |

---

## Test Call (`service_type: "test_call"`)

| Field | Req | Type | Description |
|---|:---:|---|---|
| `name` | ✓ | string |   |
| `service_tag` | ✓ | string |   |
| `service_type` | ✓ | string | `"test_call"`. |
| `bypass_proxy` |   | bool | Default `false`. |
| `call_type` |   | string | `"video"` (default), `"video-only"`, `"audio"`. |
| `crypto_mode` |   | string | `null`, `"besteffort"`, `"on"`, `"off"`. |
| `description` |   | string |   |
| `ivr_theme_name` |   | string |   |
| `local_display_name` |   | string |   |
| `max_callrate_in` |   | int |   |
| `max_pixels_per_second` |   | string |   |

---

## Worked examples

### `continue` with a VMR

```json
{
  "action": "continue",
  "result": {
    "service_type": "conference",
    "name": "Alice Jones",
    "service_tag": "abcd1234",
    "description": "Alice Jones personal VMR",
    "pin": "123456",
    "allow_guests": true,
    "guest_pin": "567890",
    "view": "one_main_zero_pips",
    "enable_overlay_text": true,
    "automatic_participants": [
      {
        "remote_alias": "sip:alice@example.com",
        "remote_display_name": "Alice",
        "local_alias": "meet.alice@example.com",
        "local_display_name": "Alice's VMR",
        "protocol": "sip",
        "role": "chair",
        "system_location_name": "London"
      },
      {
        "remote_alias": "rtmp://example.com/live/alice_vmr",
        "local_alias": "meet.alice@example.com",
        "local_display_name": "Alice's VMR",
        "protocol": "rtmp",
        "role": "guest",
        "streaming": true
      }
    ]
  }
}
```

### `reject`

```json
{ "action": "reject" }
```

### `redirect`

```json
{ "action": "redirect", "result": { "new_alias": "sip:newtarget@example.com" } }
```

---

## Dial-out (outbound service-configuration requests)

When Pexip dials out from a conference to invite a participant, it issues a service-config
request with `call_info.call_direction == "dial_out"`. The response **must match the existing
service** (same `name`, `service_type`, etc.) — local policy can only influence:

- `prefer_ipv6` in the service-configuration response.
- The media location, via the media-location request that follows (see `media-location-response.md`).
