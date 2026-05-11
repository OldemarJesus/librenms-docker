# LibreNMS — Rootless Podman Quadlet Setup

> All passwords are stored in Bitwarden under the HomeLab folder.

---

## 1. Create Service User

Run as root:

```sh
useradd --system --create-home --shell /usr/sbin/nologin svc_librenms
```

Verify that subuid/subgid ranges exist (required for rootless Podman):

```sh
grep svc_librenms /etc/subuid /etc/subgid
```

If missing, add them:

```sh
usermod --add-subuids 1279648-1345183 --add-subgids 1279648-1345183 svc_librenms
```

---

## 2. Enable Persistent Session (Linger)

Run as root to ensure the user's systemd session starts at boot and stays up without login:

```sh
loginctl enable-linger svc_librenms
```

Verify linger is active:

```sh
loginctl user-status svc_librenms
```

---

## 3. Allow Binding to Privileged Ports

LibreNMS sidecars need to bind to ports 514 (syslog) and 162 (SNMP trap). Allow unprivileged users to bind these ports by lowering the kernel limit.

Create `/etc/sysctl.d/80-librenms-ports.conf`:

```
net.ipv4.ip_unprivileged_port_start = 162
```

Apply immediately:

```sh
sysctl --system
```

> **Security note:** This allows all local users to bind ports ≥ 162. Acceptable on a dedicated host. On a shared system, use a firewall DNAT redirect instead (redirect 514/162 → 10514/10162 with firewalld and adjust `PublishPort` in the container files accordingly).

---

## 4. Create Data Directories and Place Environment File

Run as root:

```sh
install -d -o svc_librenms -g svc_librenms \
  /home/svc_librenms/app_librenms_data \
  /home/svc_librenms/app_dispatcher_data \
  /home/svc_librenms/app_syslogng_data \
  /home/svc_librenms/app_snmptrapd_data
```

Place the environment file at `/home/svc_librenms/librenms.env` and ensure ownership:

```sh
chown svc_librenms:svc_librenms /home/svc_librenms/librenms.env
chmod 600 /home/svc_librenms/librenms.env
```

---

## 5. Prepare the Database

```sql
CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'librenms'@'%' IDENTIFIED BY '3Qma5PsU7tzcze3dVA76yFEut4Z4M';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'%';
FLUSH PRIVILEGES;
```

---

## 6. Create Podman Secret

Switch to the service user first — the secret must be owned by `svc_librenms`:

```sh
machinectl shell svc_librenms@
```

Then create the secret:

```sh
printf 'your_db_password' | podman secret create DB_PASS_SECRET -
```

> The secret is stored in `~/.local/share/containers/storage/secrets/` and is only accessible to this user.

---

## 7. Install Quadlet Container Files

As root, copy the container unit files into the service user's quadlet directory:

```sh
install -d -o svc_librenms -g svc_librenms \
  /home/svc_librenms/.config/containers/systemd

cp labsetup/librenms/librenms.container     /home/svc_librenms/.config/containers/systemd/
cp labsetup/dispatcher/dispatcher.container /home/svc_librenms/.config/containers/systemd/
cp labsetup/syslogng/syslogng.container     /home/svc_librenms/.config/containers/systemd/
cp labsetup/snmptrapd/snmptrapd.container   /home/svc_librenms/.config/containers/systemd/

chown svc_librenms:svc_librenms \
  /home/svc_librenms/.config/containers/systemd/*.container
```

---

## 8. Start and Enable Services

Switch to the service user:

```sh
machinectl shell svc_librenms@
```

Reload systemd and enable the services:

```sh
systemctl --user daemon-reload
systemctl --user start librenms.service
systemctl --user start dispatcher.service
systemctl --user start syslogng.service
systemctl --user start snmptrapd.service
```

---

## 9. Verify

Check linger and session:

```sh
loginctl user-status svc_librenms
```

Check service status (as `svc_librenms`):

```sh
systemctl --user status librenms.service
systemctl --user status dispatcher.service
systemctl --user status syslogng.service
systemctl --user status snmptrapd.service
```

Check running containers:

```sh
podman ps
```

---

## 10. Caddy Reverse Proxy

Add the following block to the Caddyfile:

```
librenms.techboystore.uk {
        reverse_proxy http://localhost:8000
}
```

Then reload Caddy:

```sh
caddy reload --config /etc/caddy/Caddyfile
```

---

## Security Notes

| Concern | Detail |
|---|---|
| No shell or sudo | `svc_librenms` cannot log in interactively. Use `machinectl shell svc_librenms@` (requires root) for admin access. |
| Lingering | The user's systemd session is always active. No additional attack surface beyond running containers. |
| `ip_unprivileged_port_start=162` | All local users can bind ports ≥ 162. Acceptable on a dedicated host. |
| `NET_ADMIN` / `NET_RAW` capabilities | Granted to containers for SNMP/syslog workloads. They apply within the container's network namespace only. |
| Podman secrets | Stored in user home, readable only by `svc_librenms`. More secure than plain `.env` files. |
| Home directory permissions | Ensure `/home/svc_librenms` has mode `700`. |
