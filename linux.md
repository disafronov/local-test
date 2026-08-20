# Linux with systemd(-resolved)

`systemd-resolved` does not support wildcard static DNS records directly. Use `dnsmasq` to serve the `local.test` zone and configure `systemd-resolved` to route queries for that zone to it.

## 1. Configure `dnsmasq`

Create `/etc/dnsmasq.d/local-test.conf`:

```ini
listen-address=127.0.0.1
port=5353
bind-interfaces

address=/local.test/127.0.0.1
address=/local.test/::1
```

Enable and start `dnsmasq`:

```bash
sudo systemctl enable --now dnsmasq
```

## 2. Configure `systemd-resolved`

Create `/etc/systemd/resolved.conf.d/local-test.conf`:

```ini
[Resolve]
DNS=127.0.0.1:5353
Domains=~local.test
```

Restart `systemd-resolved`:

```bash
sudo systemctl restart systemd-resolved
```

`~local.test` defines a **route-only domain**. Queries for `local.test` and its subdomains are routed to `dnsmasq`, while all other DNS queries continue to use the DNS servers provided by the network interfaces.

## 3. Verify the configuration

Check the resolver configuration:

```bash
resolvectl status
```

The global configuration should contain:

```text
Global
       DNS Servers: 127.0.0.1:5353
        DNS Domain: ~local.test
```

Test wildcard resolution:

```bash
resolvectl query foo.local.test
resolvectl query bar.baz.local.test
```

Both names should resolve to:

```text
127.0.0.1
::1
```

The resulting DNS path is:

```text
application
    ↓
systemd-resolved
    ├── *.local.test → 127.0.0.1:5353 → dnsmasq → localhost
    └── everything else → regular interface DNS
```

The configuration is persistent across reboots.
