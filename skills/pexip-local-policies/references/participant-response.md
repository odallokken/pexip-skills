# Participant policy response — reference

A participant policy script returns:

```jinja2
{
    "action": "reject|continue",
    "result": { <participant_override_fields> }
}
```

- `"continue"` — apply the overrides in `result` (any omitted field keeps its current value).
- `"reject"` — reject the call. You may include `result.reject_reason` (string), which is
  shown to Pexip app users.

All fields are **optional**. Returning an empty `result` means "no overrides".

| Field | Type | Description |
|---|---|---|
| `bypass_lock` | bool | Allow this participant into a locked conference. Default `false`. |
| `call_tag` | string | Override call tag for this participant. |
| `can_receive_personal_mix` | bool | Allow personal layout. Default `false`. |
| `display_count` | int | Override screen count for the endpoint. |
| `disable_overlay_text` | bool | Disable the participant-name overlay. Default `false`. |
| `layout_group` | string | Layout-group name (participant pinning). |
| `preauthenticated_role` | string | `"chair"`, `"guest"`, or `null`. For SIP/H.323 the original is `null` because participant policy runs before PIN entry; returning `null` keeps prompting for a PIN, returning `"chair"`/`"guest"` acts as though that PIN was entered. |
| `prefers_multiscreen_mix` | bool | Override multiscreen participant display flag. |
| `reject_reason` | string | Reason text shown when `action == "reject"`. |
| `remote_alias` | string | Override the participant's alias. Original is still logged at connection time. |
| `remote_display_name` | string | Override the display name. May be `null` if the original was `null`. Original still logged. |
| `rx_presentation_policy` | string | `"ALLOW"` (default) or `"DENY"` (uppercase). |
| `spotlight` | int | Spotlight priority — *lower numbers = higher priority*. Client-API spotlights use the current epoch time, so small numbers (1, 2, 99) override them. |
| `send_to_audio_mixes` | list | List of `{"mix_name": "...", "prominent": <bool>}` dicts. |
| `receive_from_audio_mix` | string | Audio mix this participant receives (e.g. `"main"`). |
| `wants_presentation_in_mix` | bool | Override whether the participant gets presentation as part of the layout mix. |

---

## Worked examples

### Demote to guest and anonymise

```json
{
  "action": "continue",
  "result": {
    "preauthenticated_role": "guest",
    "remote_alias": "Witness A alias",
    "remote_display_name": "Witness A",
    "bypass_lock": true
  }
}
```

### Reject with a reason

```json
{
  "action": "reject",
  "result": { "reject_reason": "Insufficient privileges" }
}
```

---

## Timing notes — when the script runs

- **WebRTC**: participant policy runs *after* any PIN/SSO step, so `preauthenticated_role` is
  meaningful and IdP attributes are available.
- **SIP / H.323**: participant policy runs *before* PIN entry. `preauthenticated_role` is
  `None`. Override it to `"chair"`/`"guest"` to skip the PIN; leave as `None` to keep it.
- Local policy runs *after* any external policy that changed the participant. The visible
  `participant.remote_alias` / `participant.remote_display_name` reflect those changes —
  the original endpoint-supplied values are in `call_info`.
