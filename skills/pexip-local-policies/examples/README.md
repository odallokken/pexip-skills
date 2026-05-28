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

To validate any of them before saving, paste into the relevant **Test local … policy**
facility under the policy profile, set a sample `call_info` / `service_config`, and check
the **Result** / **Diagnostics** output.

For more detail on every example, see [`../references/example-scripts.md`](../references/example-scripts.md).
