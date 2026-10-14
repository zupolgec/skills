# UniFi internal API

The official Integration API (`/proxy/network/integration/v1`) does not cover NAT rules, port forwarding or WAN IP aliases. The UniFi Network web app uses an internal API for these. It accepts the same `X-API-KEY`, both locally and through the cloud connector, so restish can call it with the same profile:

```bash
restish get unifi/<path> -p <profile> -o json </dev/null
```

`unifi/<path>` is resolved against the profile's `base_url` (`.../proxy/network`).

This API is undocumented and can change between UniFi Network versions. Field names below were observed on Network 10.6. Always GET first and copy the shape you see.

## Conventions

- The site is its **name** (`default` for the first site), i.e. the site's `internalReference` in the Integration API, not its UUID.
- Two API generations:
  - `api/s/<site>/rest/<collection>`: responses wrapped as `{"meta": {"rc": "ok"}, "data": [...]}`. Errors have `rc: "error"` and a `msg`.
  - `v2/api/site/<site>/<collection>`: bare JSON arrays and objects.
- IDs are `_id` (24 hex chars), not UUIDs.
- Interfaces in NAT rules (`in_interface`, `out_interface`) are `_id`s from `networkconf`.

## Networks and WANs: `api/s/<site>/rest/networkconf`

Read-only use here: map `_id` to names, find VLANs, subnets and WAN IP aliases.

```bash
restish get unifi/api/s/default/rest/networkconf -p $P -o json </dev/null \
  | jq -r '.data[] | "\(._id)\t\(.name)\t\(.purpose)\tvlan \(.vlan // 1)\t\(.ip_subnet // "")"'

# WANs and their extra public IPs
restish get unifi/api/s/default/rest/networkconf -p $P -o json </dev/null \
  | jq '.data[] | select(.purpose=="wan") | {_id, name, wan_networkgroup, wan_type, wan_ip_aliases}'
```

`purpose` is `corporate`, `guest`, `remote-user-vpn`, `wan`, etc. `wan_networkgroup` is `WAN`, `WAN2`, … and matches `pfwd_interface` in port forwards (`wan`, `wan2`).

## Port forwarding: `api/s/<site>/rest/portforward`

```bash
restish get unifi/api/s/default/rest/portforward -p $P -o json </dev/null
```

Object:

```json
{
  "_id": "…",
  "name": "web server",
  "enabled": true,
  "pfwd_interface": "wan",
  "destination_ip": "203.0.113.10",
  "src": "any",
  "proto": "tcp",
  "dst_port": "80,443",
  "fwd": "10.0.84.20",
  "fwd_port": "80,443",
  "log": false
}
```

- `destination_ip`: `any` (the WAN's primary IP) or one specific public IP, including a WAN IP alias.
- `proto`: `tcp`, `udp` or `tcp_udp`.
- `dst_port` / `fwd_port`: single port, comma list or range, same format on both sides.
- `src`: `any` or a source restriction.

Create:

```bash
restish post unifi/api/s/default/rest/portforward -p $P -o json < pf.json
```

UniFi automatically adds a matching firewall policy, "Allow Port Forward <name>" (visible with `get-firewall-policies`, origin `SYSTEM_DEFINED`). Update and delete follow the usual REST pattern on `api/s/<site>/rest/portforward/<_id>` (PUT with the full object, DELETE).

## Custom NAT: `v2/api/site/<site>/nat`

```bash
restish get unifi/v2/api/site/default/nat -p $P -o json </dev/null
```

DNAT rule:

```json
{
  "_id": "…",
  "type": "DNAT",
  "description": "web IN (WAN1)",
  "enabled": true,
  "rule_index": 1,
  "in_interface": "<networkconf _id of the WAN or LAN the traffic enters from>",
  "destination_filter": {"address": "203.0.113.11", "filter_type": "ADDRESS_AND_PORT",
    "firewall_group_ids": [], "invert_address": false, "invert_port": false},
  "source_filter": {"filter_type": "NONE", "firewall_group_ids": [], "invert_address": false, "invert_port": false},
  "ip_address": "10.0.84.21",
  "protocol": "all",
  "ip_version": "IPV4",
  "exclude": false,
  "logging": false,
  "is_predefined": false,
  "pppoe_use_base_interface": false,
  "setting_preference": "manual"
}
```

SNAT rule (make a host leave the internet with a specific public IP):

```json
{
  "type": "SNAT",
  "description": "web OUT",
  "enabled": true,
  "out_interface": "<networkconf _id of the WAN>",
  "source_filter": {"address": "10.0.84.21", "filter_type": "ADDRESS_AND_PORT",
    "firewall_group_ids": [], "invert_address": false, "invert_port": false},
  "destination_filter": {"filter_type": "NONE", "firewall_group_ids": [], "invert_address": false, "invert_port": false},
  "ip_address": "203.0.113.11",
  "protocol": "all",
  "ip_version": "IPV4",
  "exclude": false,
  "logging": false,
  "is_predefined": false,
  "pppoe_use_base_interface": false,
  "setting_preference": "manual"
}
```

Create, then set `rule_index` (rules created through the API get `rule_index: null`; the web UI numbers them sequentially):

```bash
restish post unifi/v2/api/site/default/nat -p $P -o json < snat.json          # returns the new _id
restish get unifi/v2/api/site/default/nat -p $P -o json </dev/null \
  | jq -c --arg id <_id> '.[] | select(._id==$id) | .rule_index = <next index>' > rule.json
restish put unifi/v2/api/site/default/nat/<_id> -p $P -o json < rule.json
```

The easiest way to build a new rule is to copy an existing one of the same type with `jq`, delete `_id`, `site_id` and `rule_index`, and change the addresses.

A DNAT rule with an `in_interface` on the WAN also gets an automatic "Allow DNAT {<description>}" firewall policy (origin `DERIVED`). With `protocol: all` and no port, it exposes every port of the internal host.

## Hairpin NAT: reaching a public IP from inside

Typical need: a site is hosted inside the network, its public DNS points to a public IP of the console, and people on the internal networks must reach it through that same name.

- **Port forwarding**: UniFi applies hairpin NAT automatically on all internal networks, including networks added later. This also works when `destination_ip` is a WAN IP alias. One rule per set of ports is enough.
- **Custom DNAT**: each rule has a single `in_interface`, so hairpin needs one extra DNAT rule per internal network and VPN, and one more every time a network is added. Networks without a rule cannot reach the site through its public IP.
- **Split DNS** is the other option: a DNS policy (Integration API, `create-dns-policy`) that resolves the name to the internal IP. No NAT is needed, but it only works for clients that use the console as their DNS server.

Prefer port forwarding plus an SNAT rule (for a matching outbound IP) over per-network DNAT rules, unless the host really needs every port and protocol forwarded.
