---
name: unifi
description: >-
  Manage UniFi Network consoles (UDM, UDR, UCG, Cloud Key) through the UniFi
  Network API using restish. Use when the user wants to inspect or change
  devices, clients, networks/VLANs, WiFi, firewall zones and policies, ACL and
  DNS policies, vouchers, WANs, port forwarding or NAT rules on a UniFi
  console, locally or remotely through unifi.ui.com.
license: MIT
metadata:
  author: zupolgec
  version: "0.0.1"
---

# UniFi Network

UniFi has no official CLI. The UniFi Network application publishes an OpenAPI spec, and [restish](https://rest.sh) turns that spec into a CLI at runtime: every operation becomes a command, with `--help` showing parameters and request bodies.

Some features (NAT rules, port forwarding, WAN IP aliases) are not in the official API. The same API key also works on the console's internal API, which restish can call directly. See [references/internal-api.md](references/internal-api.md).

Docs: https://developer.ui.com/network/

## Prerequisites

```bash
restish --version          # install with: brew install restish
jq --version
restish api list           # is the "unifi" API configured?
restish api inspect unifi  # which profiles exist?
```

If `unifi` is configured, skip to [Usage](#usage). Otherwise follow [Setup](#setup).

## Two ways to reach a console

| | Cloud (recommended) | Local |
|---|---|---|
| Base | `https://api.ui.com/v1/connector/consoles/<consoleId>/proxy/network` | `https://<console-ip>/proxy/network` |
| Key created in | unifi.ui.com → Settings → API | the Integrations page of the UniFi Network settings, on that console |
| Reach | any console of the UI account, from anywhere | only that console, only from its network or VPN |
| TLS | valid certificate | self-signed: add `--rsh-insecure` to every call |

Both use the `X-API-KEY` header, but they are different keys: use a cloud key with cloud profiles and a key created on the console with local profiles.

## Setup

### 1. Store the API key

Never paste keys in chat or in commands that end up in logs. On macOS keep them in the Keychain, one entry per key:

```bash
# prompts for the key
security add-generic-password -U -s unifi-api -a <key-name> -l "UniFi API (<key-name>)" -w

# or copy it from 1Password without showing it
security add-generic-password -U -s unifi-api -a <key-name> -l "UniFi API (<key-name>)" \
  -w "$(op read 'op://<vault>/<item>/credential')"
```

restish reads it at every call with `command:security find-generic-password -s unifi-api -a <key-name> -w`. On Linux use `command:secret-tool lookup service unifi-api account <key-name>`, or `env:UNIFI_API_KEY` with the key in an environment variable.

### 2. Find the consoles (cloud key)

```bash
KEY=$(security find-generic-password -s unifi-api -a <key-name> -w)
curl -s -H "X-API-KEY: $KEY" -H 'Accept: application/json' https://api.ui.com/v1/hosts \
  | jq -r '.data[] | select(.type=="console") | "\(.id)\t\(.reportedState.name // .reportedState.hostname)\t\(.ipAddress)"'
unset KEY
```

The first column is the `consoleId`.

### 3. Download the spec matching the console version

```bash
mkdir -p ~/.config/restish/specs
curl -sfL https://developer.ui.com/network/v10.6.106/openapi.json \
  -o ~/.config/restish/specs/unifi-network-10.6.106.json
```

After the first profile works, check the version with `restish unifi get-info -p <profile> </dev/null`. If it differs, download the spec for that version (the URL takes the exact version) and update `spec_files`.

### 4. Configure restish

Config file: `restish config path` (usually `~/.config/restish/restish.json`, keep it `chmod 600`). One profile per console. Name profiles after the console; do not create a `default` profile, so every command must say which console it targets.

```json
{
  "apis": {
    "unifi": {
      "spec_files": ["/Users/<you>/.config/restish/specs/unifi-network-10.6.106.json"],
      "profiles": {
        "office": {
          "base_url": "https://api.ui.com/v1/connector/consoles/<consoleId>/proxy/network",
          "operation_base": "/v1/connector/consoles/<consoleId>/proxy/network/integration",
          "auth": {"type": "api-key", "params": {"in": "header", "name": "X-API-KEY",
            "value": "command:security find-generic-password -s unifi-api -a <key-name> -w"}}
        },
        "office-local": {
          "base_url": "https://192.168.1.1/proxy/network",
          "operation_base": "/proxy/network/integration",
          "auth": {"type": "api-key", "params": {"in": "header", "name": "X-API-KEY",
            "value": "command:security find-generic-password -s unifi-api -a <local-key-name> -w"}}
        }
      }
    }
  }
}
```

Both fields are required:

- `operation_base` routes the generated commands to `/integration/v1/...`. Without it restish takes the path of the first server in the spec, a cloud URL with a placeholder console ID.
- `base_url` (ending in `/proxy/network`) is what `restish get unifi/<path>` uses for the internal API.

To add a profile to an existing file:

```bash
F=$(restish config path); C=<consoleId>; P=<profile>; K=<key-name>
jq --arg c "$C" --arg p "$P" --arg k "$K" '.apis.unifi.profiles[$p] = {
  base_url: ("https://api.ui.com/v1/connector/consoles/" + $c + "/proxy/network"),
  operation_base: ("/v1/connector/consoles/" + $c + "/proxy/network/integration"),
  auth: {type: "api-key", params: {in: "header", name: "X-API-KEY",
    value: ("command:security find-generic-password -s unifi-api -a " + $k + " -w")}}}' \
  "$F" > "$F.tmp" && mv "$F.tmp" "$F" && chmod 600 "$F"
```

Test it:

```bash
restish unifi get-info -p <profile> </dev/null                     # {"applicationVersion": "..."}
restish unifi get-site-overview-page -p <profile> -o json </dev/null
```

## Usage

### Rules for every call

- **Always pass `-p <profile>`.** If the user did not say which console, list the profiles and ask.
- **Always redirect stdin**: `</dev/null` when there is no body, `< body.json` when there is one. restish waits for a body on stdin otherwise and hangs.
- Use `-o json` and pipe to `jq` for anything you need to parse.
- Local profiles need `--rsh-insecure` (self-signed certificate).
- Most endpoints need the **site ID** (a UUID): get it from `get-site-overview-page`. Two consoles can have the same site ID, so the profile is what identifies the console.

### Discover commands

```bash
restish unifi --help -p <profile>                 # all operations, grouped by area
restish unifi <command> --help -p <profile>       # parameters, request schema, example body
```

Command names come from the spec's operation IDs, e.g. `get-adopted-device-overview-page`, `create-firewall-policy`, `update-network`.

### Lists, pagination, filters

List commands return `{offset, limit, count, totalCount, data}`. Default `--limit` is 25, maximum 200: page with `--offset` when `totalCount` is higher.

`--filter` takes UniFi's filter syntax:

```text
name.like('U7*')                                  # devices
not(name.like('guest*'))
ipAddress.eq('10.0.0.5')                          # clients
and(type.eq('WIRELESS'), connectedAt.gt(2026-01-01))
or(a.isNull(), b.in('x', 'y'))
```

Functions: `eq ne gt ge lt le like in notIn isNull isNotNull`, plus `isEmpty contains containsAny containsAll containsExactly` for sets. Strings go in single quotes; dates and numbers don't. Spec: https://developer.ui.com/network/v10.6.106/filtering

Only some fields can be filtered, and they differ per endpoint (clients can filter on `type` and `ipAddress` but not `name`). The command's `--help` lists them under "Filterable properties". An unknown property returns `400 api.request.invalid-filter`. When a field is not filterable, fetch with `--limit 200` and filter with `jq`.

### Common reads

```bash
S=<siteId>; P=<profile>
restish unifi get-adopted-device-overview-page $S --limit 200 -p $P -o json </dev/null \
  | jq -r '.data[] | "\(.name)\t\(.model)\t\(.ipAddress)\t\(.state)\t\(.firmwareVersion)"'
restish unifi get-connected-client-overview-page $S --limit 200 -p $P -o json </dev/null | jq '.totalCount'
restish unifi get-networks-overview-page $S -p $P -o json </dev/null | jq -r '.data[] | "\(.name)\tvlan \(.vlanId)"'
restish unifi get-wifi-broadcast-page $S -p $P -o json </dev/null | jq -r '.data[].name'
restish unifi get-firewall-zones $S -p $P -o json </dev/null
restish unifi get-firewall-policies $S --limit 200 -p $P -o json </dev/null \
  | jq '[.data[] | select(.metadata.origin != "SYSTEM_DEFINED")]'    # user-defined and derived only
restish unifi get-wans-overview-page $S -p $P -o json </dev/null
```

Other areas: ACL rules, DNS policies, traffic matching lists (named sets of IPs or ports used by firewall policies), hotspot vouchers, switching (LAG, stacks), VPN servers and site-to-site tunnels, device and port actions.

### Writes

Request bodies come from a file on stdin, or as restish shorthand arguments for simple objects:

```bash
restish unifi create-network $S -p $P -o json < network.json
restish unifi create-network $S 'name: Guests, enabled: true, vlanId: 30' -p $P -o json
```

Read the body schema in `--help` first. Some fields are polymorphic (e.g. a firewall policy `action.type` of `ALLOW`/`BLOCK`/`REJECT`) and `--help` only shows them as `string`. The variants are in the spec as a discriminator mapping:

```bash
jq '.components.schemas["Firewall policy action"].discriminator.mapping' \
  ~/.config/restish/specs/unifi-network-*.json
```

`update-*` operations (PUT) replace the whole object: GET it, change the fields, send it all back. Firewall policies also support `patch-firewall-policy`.

## Safety

These are production network devices. A wrong rule can cut off a whole office, including your own access to the console.

- **Reads are free.** Before any create, update, delete, device action or reorder, show the user exactly what will change (object, console, before/after) and wait for explicit confirmation. Approval of one change does not cover the next.
- Re-read the current object right before writing it, and verify the result right after.
- Prefer disabling (`enabled: false`) over deleting when the user just wants something off.
- Be extra careful with firewall policies, ACLs, zones, WAN and networks used by the management VLAN or the VPN you are connected through: changes take effect immediately.
- Never print API keys. Pass them through the Keychain or environment, never on the command line.

## Internal API: NAT, port forwarding, WAN aliases

Port forwarding, custom NAT (DNAT/SNAT), WAN IP aliases and the network/interface IDs they reference are only available through the console's internal API. Call it with the same profile:

```bash
restish get unifi/v2/api/site/default/nat -p <profile> -o json </dev/null
restish get unifi/api/s/default/rest/portforward -p <profile> -o json </dev/null
```

Here the site is its **name** (`default`, the `internalReference` of the site), not the UUID. Endpoints, object shapes, write examples and how hairpin NAT behaves are in [references/internal-api.md](references/internal-api.md).

## Troubleshooting

| Symptom | Cause |
|---|---|
| Command hangs | stdin not redirected: add `</dev/null` |
| `401 unauthorized` from api.ui.com | key revoked or expired (create a new one on unifi.ui.com), or not a cloud key. Check it with `GET https://api.ui.com/v1/hosts` |
| Requests go to `.../consoles/942A6FF...:9999999999/...` | profile is missing `operation_base` |
| `404` on `restish get unifi/...` | `base_url` does not end in `/proxy/network`, or wrong site name |
| `requires an API short name or --rsh-server` | no `-p` given and no default profile (intended) |
| TLS error on a local profile | add `--rsh-insecure` |
| Connection timeout on a local profile | not on the console's network or VPN: use a cloud profile |
| `-v` | shows the exact URL and headers sent (the key is redacted) |
