# smart-hostname-dnsmasq

OpenWrt/dnsmasq script that generates smart hostnames from DHCP, mDNS, local cache, and device type/OUI hints.

## Name Priority

1. Preserve static hostnames already configured in OpenWrt/dnsmasq.
2. Use passively observed mDNS from `tcpdump` on UDP/5353.
3. Reuse the last valid mDNS name cached for the same MAC address.
4. Use the DHCP hostname if it is valid.
5. Detect the device type from mDNS/DHCP/OUI when no trusted name exists.
6. If there are no hints, generate `Device-xxxx`.

Anonymized examples:

```txt
iPhone-de-XXXX.local + AA:BB:CC:EE:33:44 -> iPhone-de-XXXX-3344
Printer.local + AA:BB:CC:DD:EE:FF -> Printer-eeff
No hints + AA:BB:CC:DD:EE:FF -> Device-eeff
```

## Scope

- Does not create DHCP reservations.
- Does not change IP addresses.
- Does not touch `odhcpd`.
- Does not rewrite `/tmp/dhcp.leases`.
- Maintains DNS names in `/tmp/smart-hostname/hosts`.
- Sends `SIGHUP` to `dnsmasq` only when the generated hosts file changes.
- Automatic `dnsmasq` restart is disabled by default.

## Files

- `smart-hostname-hook-safe`: main script.
- `test-*`: anonymized test fixtures.

## Quick Install

On OpenWrt:

```sh
apk update
apk add tcpdump
mkdir -p /usr/libexec /tmp/smart-hostname
```

Copy the script:

```sh
scp smart-hostname-hook-safe root@192.168.1.1:/usr/libexec/smart-hostname
ssh root@192.168.1.1 'chmod 755 /usr/libexec/smart-hostname'
```

Create the hotplug hook:

```sh
cat >/etc/hotplug.d/dhcp/95-smart-hostname <<'EOF'
#!/bin/sh
[ -x /usr/libexec/smart-hostname ] || exit 0
/usr/libexec/smart-hostname "$@"
EOF
chmod 755 /etc/hotplug.d/dhcp/95-smart-hostname
```

Configure `dnsmasq`:

```sh
uci add_list dhcp.@dnsmasq[0].addnhosts='/tmp/smart-hostname/hosts'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

## Verification

```sh
sh -n /usr/libexec/smart-hostname
logread | grep smart-hostname
cat /tmp/smart-hostname/hosts
nslookup Device-xxxx.lan 127.0.0.1
```
