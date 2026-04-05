# terraform-provider-unifi — `unifi_wlan` perpetual `mac_filter` drift + 400 on apply

After importing an existing `unifi_wlan` resource, `tofu plan` always shows computed drift on the `mac_filter` nested block. Attempting to apply this no-op change fails with `400 Invalid Payload` because the provider sends `schedule_with_duration: null` in the PUT request.

## Reproduction

Requires a UniFi controller with at least one existing WLAN.

```bash
# 1. Set up
export UNIFI_API_KEY="your-integration-api-key"
export TF_VAR_wlan_passphrase="your-wlan-passphrase"

# Edit main.tf.json: replace REPLACE_WITH_YOUR_NETWORK_ID and
# REPLACE_WITH_YOUR_USER_GROUP_ID with actual IDs from your controller.
# You can find these via:
#   curl -sk -H "X-API-KEY: $UNIFI_API_KEY" \
#     https://localhost:8443/proxy/network/api/s/default/rest/wlanconf \
#     | jq '.data[0] | {network: .networkconf_id, usergroup: .usergroup_id}'

# 2. Init + import an existing WLAN
tofu init
tofu import unifi_wlan.test <your-wlan-id>

# 3. Plan shows perpetual mac_filter drift
tofu plan
# Bug 1: mac_filter always shows as changed (enabled, policy → "known after apply")

# 4. Applying the "no-op" change fails
tofu apply
# Bug 2: 400 Invalid Payload — provider sends schedule_with_duration: null
```

## Expected

After import, `tofu plan` should show no changes when the config matches the controller state. The `mac_filter` block should be stable in state.

## Actual

### Bug 1: Perpetual `mac_filter` drift

Every `tofu plan` after import shows:

```
  # unifi_wlan.test will be updated in-place
  ~ resource "unifi_wlan" "test" {
        id = "..."
      ~ mac_filter = {
          ~ enabled = false -> (known after apply)
          + list    = (known after apply)
          ~ policy  = "allow" -> (known after apply)
        } -> (known after apply)
        name = "Test SSID"
        # (27 unchanged attributes hidden)
    }
```

This happens regardless of whether `mac_filter` is specified in the config or not. The nested block always shows as computed.

### Bug 2: Applying fails with 400

When you try to apply the change (which should be a no-op), the controller rejects it:

```
Error: Error Updating WLAN

  Could not update WLAN with ID ...:
  api.err.InvalidPayload (400) for PUT
  https://localhost:8443/proxy/network/api/s/default/rest/wlanconf/...
```

The PUT payload includes `"schedule_with_duration":null` which the controller does not accept. This field appears to be injected by the provider during serialization even when schedules are not configured.

## Versions

- Provider: `ubiquiti-community/unifi` v0.41.25
- OpenTofu: v1.11.5
- UniFi Controller: standalone (not UDM), accessed via Integration API key
- OS: macOS (aarch64-darwin)

## Notes

- The `schedule_with_duration` field issue is potentially related to [paultyng/terraform-provider-unifi#263](https://github.com/paultyng/terraform-provider-unifi/issues/263)
- Auth via Integration API key (`UNIFI_API_KEY` env var)
- Tested with both `.tf` HCL and `.tf.json` JSON config format — same result

## Related Issue

<!-- Will be filled in after filing -->
