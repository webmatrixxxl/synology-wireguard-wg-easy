
# wg-easy v15 Docker Compose Examples

These compose files are updated examples for running the maintained `wg-easy` image:

```txt
ghcr.io/wg-easy/wg-easy:15.2
````

They are intended for people following older tutorials that used the old and unmaintained Docker image:

```txt
weejewel/wg-easy
```

Old image reference:

```txt
https://hub.docker.com/r/weejewel/wg-easy
```

Maintained project:

```txt
https://github.com/wg-easy/wg-easy
```

Video this update relates to:

```txt
https://www.youtube.com/watch?v=v0Z1m658Xe8
```

---

## Ports

| Port    | Protocol | Purpose               |
| ------- | -------: | --------------------- |
| `51820` |      UDP | WireGuard VPN traffic |
| `51821` |      TCP | wg-easy web interface |

---

# Option 1: Bridge Mode

Use this if you want Docker to publish the required ports manually.

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15.2
    container_name: wg-easy
    restart: unless-stopped

    environment:
      - PORT=51821
      - HOST=0.0.0.0
      - INSECURE=true
      - DISABLE_IPV6=true

    volumes:
      - /volume1/docker/wg-easy:/etc/wireguard
      - /lib/modules:/lib/modules:ro

    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"

    cap_add:
      - NET_ADMIN
      - SYS_MODULE

    devices:
      - /dev/net/tun:/dev/net/tun

    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=1
      - net.ipv6.conf.all.forwarding=1
      - net.ipv6.conf.default.forwarding=1

    networks:
      - wg

networks:
  wg:
    driver: bridge
    enable_ipv6: false
    ipam:
      driver: default
      config:
        - subnet: 11.1.0.0/24
```

Important: the service must be attached to the `wg` network. Without this part:

```yaml
networks:
  - wg
```

the custom network definition will not actually be used.

---

# Option 2: Host Mode

Use this if bridge mode causes networking or access issues, especially on Synology/NAS setups.

In host mode, Docker does not need explicit `ports:` mappings because the container uses the host network directly.

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15.2
    container_name: wg-easy
    restart: unless-stopped
    network_mode: "host"

    environment:
      - PORT=51821
      - HOST=0.0.0.0
      - INSECURE=true
      - DISABLE_IPV6=true

    volumes:
      - /volume1/docker/wg-easy:/etc/wireguard
      - /lib/modules:/lib/modules:ro

    cap_add:
      - NET_ADMIN
      - SYS_MODULE

    devices:
      - /dev/net/tun:/dev/net/tun
```

---

## Access

After starting the container, open:

```txt
http://YOUR_SERVER_IP:51821
```

---

## Notes

* The `/volume1/docker/wg-easy:/etc/wireguard` volume keeps the WireGuard configuration persistent.
* The `/dev/net/tun` device is required for VPN functionality.
* `NET_ADMIN` is required so the container can configure networking.
* `SYS_MODULE` may be needed on some systems to load kernel modules.
* Make sure the Docker bridge subnet does not conflict with your LAN or WireGuard client subnet.
* Do not expose the wg-easy web interface directly to the internet without proper protection.


## Hooks / Internet Tunneling

For wg-easy v15, the firewall hooks may need to use `eth0` instead of `wg0` in the `FORWARD` rules.

```bash

#PostUp
iptables -t nat -A POSTROUTING -s {{ipv4Cidr}} -o {{device}} -j MASQUERADE; iptables -A INPUT -p udp -m udp --dport {{port}} -j ACCEPT; iptables -A FORWARD -i eth0 -j ACCEPT; iptables -A FORWARD -o eth0 -j ACCEPT;

#PostDown
iptables -t nat -D POSTROUTING -s {{ipv4Cidr}} -o {{device}} -j MASQUERADE; iptables -D INPUT -p udp -m udp --dport {{port}} -j ACCEPT; iptables -D FORWARD -i eth0 -j ACCEPT; iptables -D FORWARD -o eth0 -j ACCEPT;
```

### Why `eth0` instead of `wg0`?

Inside the Docker container, `wg0` is the WireGuard VPN interface, while `eth0` is the container network interface used to reach the outside network.

In bridge mode, VPN clients connect through `wg0`, but their traffic must leave the container through `eth0`. Because of that, the forwarding rules need to allow traffic going through `eth0`.

This is why these rules are used:

```bash
iptables -A FORWARD -i eth0 -j ACCEPT
iptables -A FORWARD -o eth0 -j ACCEPT
```

instead of only forwarding through `wg0`.


In most Docker bridge-mode setups, the interface is `eth0`.
