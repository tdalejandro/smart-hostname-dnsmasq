# smart-hostname-dnsmasq

Script para OpenWrt/dnsmasq que genera hostnames inteligentes desde DHCP, mDNS, cache local y pistas de tipo/OUI.

## Prioridad de nombres

1. Respeta hostnames estaticos ya configurados en OpenWrt/dnsmasq.
2. Usa mDNS visto pasivamente con `tcpdump` en UDP/5353.
3. Reusa el ultimo mDNS valido guardado para la misma MAC.
4. Usa hostname DHCP si es valido.
5. Detecta tipo por mDNS/DHCP/OUI cuando no hay nombre confiable.
6. Si no hay pistas, genera `Device-xxxx`.

Ejemplos anonimizados:

```txt
iPhone-de-XXXX.local + AA:BB:CC:EE:33:44 -> iPhone-de-XXXX-3344
Printer.local + AA:BB:CC:DD:EE:FF -> Printer-eeff
Sin pistas + AA:BB:CC:DD:EE:FF -> Device-eeff
```

## Alcance

- No crea reservas DHCP.
- No cambia IPs.
- No toca `odhcpd`.
- No reescribe `/tmp/dhcp.leases`.
- Mantiene nombres DNS en `/tmp/smart-hostname/hosts`.
- Envía `SIGHUP` a `dnsmasq` solo si cambia el archivo de hosts generado.
- El reinicio automatico de `dnsmasq` queda desactivado por defecto.

## Archivos

- `smart-hostname-hook-safe`: script principal.
- `test-*`: fixtures de prueba anonimizados.

## Instalacion rapida

En OpenWrt:

```sh
apk update
apk add tcpdump
mkdir -p /usr/libexec /tmp/smart-hostname
```

Copiar el script:

```sh
scp smart-hostname-hook-safe root@192.168.1.1:/usr/libexec/smart-hostname
ssh root@192.168.1.1 'chmod 755 /usr/libexec/smart-hostname'
```

Crear hotplug:

```sh
cat >/etc/hotplug.d/dhcp/95-smart-hostname <<'EOF'
#!/bin/sh
[ -x /usr/libexec/smart-hostname ] || exit 0
/usr/libexec/smart-hostname "$@"
EOF
chmod 755 /etc/hotplug.d/dhcp/95-smart-hostname
```

Configurar `dnsmasq`:

```sh
uci add_list dhcp.@dnsmasq[0].addnhosts='/tmp/smart-hostname/hosts'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

## Verificacion

```sh
sh -n /usr/libexec/smart-hostname
logread | grep smart-hostname
cat /tmp/smart-hostname/hosts
nslookup Device-xxxx.lan 127.0.0.1
```
