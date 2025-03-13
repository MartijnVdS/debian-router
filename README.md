# Debian as a Router

> [!CAUTION]
> Make sure you can access the machine's (serial) console,
> as one small misconfiguration can cause the system to become
> unreachable over the network.

To turn a plain Debian installation on a regular (small) PC into a
router/firewall, some configuration is needed.

The configuration files in this repository set up an internal network
that's split into five parts:

  - "Management", for "device management" (think routers/switches or IPMI style stuff)
  - "Server", for servers (VMs, NAS, etc.); this is sometimes called "DMZ"
  - "Trusted", for trusted clients (desktop, laptop, maybe phones and tablets)
  - "IoT", for untrusted clients (light bulbs, "smart" TVs, etc.)
  - "Guest", for guests

It also sets up a Wireguard server that listens on the WAN
interface and allows connections to services running on my home
server from devices like my phone and laptop when outside the house.

Each of the networks gets its own subnet from `10.240.0.0/16`.
If IPv6 is available, each network also gets an IPv6 prefix.

<!-- Future: ULA and IPv6 masquerade? -->

## sysctl: IP forwarding

By default, Linux does not forward traffic between interfaces.

To enable forwarding, put [network.conf](etc/sysctl.d/network.conf) in
`/etc/sysctl.d` and reload it:

```sh
sysctl -p /etc/sysctl.d/network.conf
```

With this configured, the machine will forward traffic coming
in on one interface to another one, based to the route table(s).
You can inspect the default route table with `ip route show`.

<!-- Future: move IP forwarding configuration to systemd-networkd -->

## nftables: Firewall rules

To separate the networks and keep incoming IPv6 connections in check,
an `nftables` firewall configuration is used that allows the following
traffic.

* ✅ New connections allowed
* ❌ New connections blocked

| From ↓ To → | Management | Server | Trusted| IoT | Guest | Wireguard |
|-|-|-|-|-|-|-|
| Management |     | ❌[^1]  | ❌  | ❌  | ❌  | ❌ |
| Server     | ❌  |     | ✅  | ❌  | ❌  | ❌ |
| Trusted    | ✅  | ✅  |     | ✅  | ❌  | ❌ |
| IoT        | ❌  | ❌[^1]  | ❌  |     | ❌  | ❌ |
| Guest      | ❌  | ❌[^1]  | ❌  | ❌  |     | ❌ |
| Wireguard  | ❌  | ✅  | ✅  | ✅  | ❌  |    |

In addition, devices from every network can connect to the internet
(using NAT if using IPv4) and to a DNS server in the "Server"/DMZ
network.

[^1]: DNS traffic to DNS server in the servers network is allowed

```sh
apt install nftables
```

Copy [nftables.conf](etc/nftables.conf) to `/etc`. Before loading it,
read through and adjust things as needed.

To reload the complete firewall ruleset, run
`nft -f /etc/nftables.conf`

To show rules, use `nft list tables` then show a table with
`nft list table inet filter` etc.

For more information:

* `man nft`
* https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes

<!-- Future: better counters and named maps in nftables config? -->

## systemd-networkd: Configuring network interfaces

First, replace the default `ifupdown` network configuration utilities
with `systemd-networkd` and `systemd-resolved`.

There is a lot of documentation about how to set up complex network
devices (like a VLAN-aware bridge) using `systemd-networkd`, and
while that's also possible with `ifupdown`, documentation on how to
do that is sometimes harder to find.

```sh
apt install systemd-resolved
systemctl enable --now systemd-networkd
apt purge ifupdown
```

`systemd-networkd` is configured by files in `/etc/systemd/network`.
The files for this configuration can be found in
[etc/systemd/network](/etc/systemd/network/).

| File | Function? |
|-|-|
| `00-localbridge.netdev` | Defines a (new) network device: the local bridge. This bridge will be used for all "internal" network traffic. It is VLAN-aware, so traffic of different (separated) networks stays separate. |
| `00-localbridge.network` | Network configuration of the bridge device. Because this is a VLAN aware bridge, and we're not really using the "default" VLAN, it does not get any IP configuration, just a list of VLANs to attach. |
| `01-management.network`, `02-server.network`, `03-trusted.network`, `04-iot.network`, `06-guest.network` | These define the network configuration of each VLAN. They all get their own subnet and IPv6 prefix, and possibly some static DHCP leases. |
| `enp5s0.network` | Physical network port for the internal network (a "VLAN trunk"). This configuration contains settings for each VLAN that should be present on the trunk and how (tagged or untagged) it should come out of this port. Also contains configuration to make the interface a part of the bridge. There can be multiple of these, each with different VLAN settings. |
| `vlan_*.netdev` | These define the separate `vlan` devices used by the `network` files above to configure the internal networks. |
| `wan.network` | Configures the connection to your ISP. |

Restart `systemd-networkd` (`systemctl restart systemd-networkd`)
and check how it's doing with `networkctl`.

## Wireguard

To set up wireguard, you need to generate a keypair for the server
using the `wg` tool:

```sh
wg genkey | tee wg-private.key | wg pubkey > wg-public.key
```

Then for each connection, also generate a pre-shared key:

```sh
wg genpsk > connection1-psk.txt
```

Set ownership/permissions on the private key and PSK so they're
not readable by regular users (`chmod 600` and owner `root` should
do it).

Then generate keys and configure things in `/etc/systemd/network/wireguard.netdev`. After (re)configuring, reload the network
configuration: `networkctl reload`

Older `systemd-networkd` doesn't automatically create/remove
wireguard peers when a reload is triggered. A workaround is
described in https://github.com/systemd/systemd/issues/25547
(`ip link del wireguard; networkctl reload`)

## Other things

Autodiscovery between networks (most likely "trusted" and "iot", and
"trusted" and "server" networks) can be enabled by installing the
`mdns-reflector` package and configuring it. See https://github.com/vfreex/mdns-reflector for more information.