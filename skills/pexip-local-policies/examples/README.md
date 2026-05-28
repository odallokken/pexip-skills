# Example local policy scripts

Standalone `.jinja2` files of the most useful local policy patterns. Each file is meant to
be pasted directly into the **Script** box of a Pexip policy profile (under
**Call control > Policy profiles > Apply local policy**).

| File | Profile section | What it does |
|---|---|---|
| [`passthrough-service-config.jinja2`](passthrough-service-config.jinja2) | Service configuration | The canonical "no-op" service-config script — start every new script from this. |
| [`passthrough-media-location.jinja2`](passthrough-media-location.jinja2) | Media location | The canonical "no-op" media-location script. |
| [`debug-call-info.jinja2`](debug-call-info.jinja2) | Service configuration | Logs the entire `call_info` to the support log without changing behaviour — invaluable for seeing what Pexip actually delivers. |
| [`subnet-based-media-routing.jinja2`](subnet-based-media-routing.jinja2) | Media location | Picks a media location based on the caller's remote IP using `pex_in_subnet`. |
| [`remove-pin-for-registered-devices.jinja2`](remove-pin-for-registered-devices.jinja2) | Service configuration | Skips the PIN gate when the call comes from a registered endpoint. |
| [`reject-suspicious-user-agents.jinja2`](reject-suspicious-user-agents.jinja2) | Service configuration | Drops calls from common VOIP-scanner User-Agents. |
| [`lock-by-service-tag.jinja2`](lock-by-service-tag.jinja2) | Service configuration | Auto-locks any VMR whose service tag starts with `"locked"`. |
| [`time-of-day-theme.jinja2`](time-of-day-theme.jinja2) | Service configuration | Switches the IVR theme based on the wall-clock hour. |
| [`teams-layout-for-gateway-calls.jinja2`](teams-layout-for-gateway-calls.jinja2) | Service configuration | Forces the Teams-like layout for Teams gateway calls. |
| [`breakout-room-by-alias.jinja2`](breakout-room-by-alias.jinja2) | Service configuration | Synthesises breakout-room service config on the fly for two team aliases. |
| [`mask-pstn-numbers.jinja2`](mask-pstn-numbers.jinja2) | Participant | Anonymises PSTN callers (alias + display name). |
| [`overlay-text-for-gateway-by-tag.jinja2`](overlay-text-for-gateway-by-tag.jinja2) | Service configuration | Enables `enable_overlay_text` on gateway rules whose service tag starts with `overlaytext`. |
| [`force-text-overlay-everywhere.jinja2`](force-text-overlay-everywhere.jinja2) | Service configuration | Force-enables `enable_overlay_text` for every resolved service. |
| [`text-overlay-for-scheduled.jinja2`](text-overlay-for-scheduled.jinja2) | Service configuration | Enables overlay text only for scheduled meetings (alias matches `77NNNNNN`). |
| [`live-captions-for-scheduled.jinja2`](live-captions-for-scheduled.jinja2) | Service configuration | Enables `live_captions_enabled` for scheduled-meeting conferences (alias prefix `88`). |
| [`adp-mssip-for-scheduled-by-email.jinja2`](adp-mssip-for-scheduled-by-email.jinja2) | Service configuration | Auto-dials the VMR owner's email as an MS-SIP ADP for scheduled meetings. |
| [`add-adp-without-overwriting.jinja2`](add-adp-without-overwriting.jinja2) | Service configuration | Appends an extra ADP using `list.extend()` so existing `automatic_participants` survive. |
| [`dial-out-to-s4b-on-guest.jinja2`](dial-out-to-s4b-on-guest.jinja2) | Service configuration | Dials the VMR owner over MS-SIP only on first guest entry, not for the host's own endpoint. |
| [`skype-meeting-adhoc-vmr.jinja2`](skype-meeting-adhoc-vmr.jinja2) | Service configuration | Converts an inbound SfB Focus GRUU dial-in into an ad-hoc VMR that dials back into the SfB meeting. |
| [`override-outbound-gateway-from.jinja2`](override-outbound-gateway-from.jinja2) | Service configuration | Overrides `local_alias` / `local_display_name` on point-to-point gateway B-legs. |
| [`vmr-as-gateway-by-prefix.jinja2`](vmr-as-gateway-by-prefix.jinja2) | Service configuration | Rewrites a VMR call with a numeric prefix into an outbound MS-SIP gateway leg (extension → Skype URI). |
| [`pinless-by-endpoint-vmr-mapping.jinja2`](pinless-by-endpoint-vmr-mapping.jinja2) | Service configuration | Strips PINs only when a specific endpoint dials a specific VMR (whitelist tuple). |
| [`force-audio-only-by-vendor.jinja2`](force-audio-only-by-vendor.jinja2) | Service configuration | Forces `call_type=audio` for listed `call_info.vendor` User-Agent strings. |
| [`bandwidth-override-by-alias.jinja2`](bandwidth-override-by-alias.jinja2) | Service configuration | Sets `max_callrate_in/out` for a whitelist of H.323 dial-in endpoints. |
| [`trust-domains-for-teams-cvi.jinja2`](trust-domains-for-teams-cvi.jinja2) | Service configuration | Sets `treat_as_trusted` for Teams CVI calls (tag `TEAMS`) coming from internal domains. |
| [`audio-steering-code-by-country.jinja2`](audio-steering-code-by-country.jinja2) | Service configuration | Extracts a 2-letter country code from the calling alias and prepends an Avaya audio steering code. |
| [`priority-meeting-by-weekday.jinja2`](priority-meeting-by-weekday.jinja2) | Service configuration | During a weekly priority window, redirects non-priority calls into a media-playback announcement and disconnects. |
| [`media-playback-before-teams.jinja2`](media-playback-before-teams.jinja2) | Service configuration | Plays a playlist before transferring a `88<teams-id>@pex.vc` call into the Teams gateway rule. |
| [`media-playback-before-vmr.jinja2`](media-playback-before-vmr.jinja2) | Service configuration | Routes `sso*` aliases via a media-playback service first, then transfers to the real VMR, preserving PIN/IDP context. |
| [`scheduled-meeting-pin-with-playback-idp.jinja2`](scheduled-meeting-pin-with-playback-idp.jinja2) | Service configuration | Deterministic-PIN scheduled meeting (host/guest by location) with IDP login + media playback for untrusted callers. |
| [`do-not-transcode-by-tag.jinja2`](do-not-transcode-by-tag.jinja2) | Service configuration | Disables `transcoding_enabled` on services whose tag is exactly `do-not-transcode`. |

To validate any of them before saving, paste into the relevant **Test local … policy**
facility under the policy profile, set a sample `call_info` / `service_config`, and check
the **Result** / **Diagnostics** output.

For more detail on every example, see [`../references/example-scripts.md`](../references/example-scripts.md).
