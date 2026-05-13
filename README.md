# OpenBSDFileShare

This guide shows how to turn an OpenBSD host into an internet-accessible shared file server with mostly built-in components.

## 1) Recommended architecture

Use **built-in SFTP (`sshd`)** for file sharing and put it behind a **built-in VPN (`iked`/IPsec)**.

- File sharing: `sshd` + chrooted SFTP users
- Network protection: `pf` firewall + IKEv2 road-warrior VPN (`iked`)
- Optional alternative: mTLS front-end for HTTPS file distribution

## 2) Base system prep

```sh
doas syspatch
#doas pkg_add -u   # optional for installed packages
```

Create a storage area and group:

```sh
doas groupadd fileshare
doas mkdir -p /srv/share/common
doas chown root:fileshare /srv/share/common
doas chmod 2770 /srv/share/common
```

## 3) SFTP-only users (built-in)

Create users who can only use SFTP:

```sh
doas useradd -m -d /home/alice -s /sbin/nologin -G fileshare alice
doas passwd alice
```

Configure `/etc/ssh/sshd_config`:

```text
# Keep normal SSH for admins; add a restricted SFTP block for share users
Subsystem sftp internal-sftp

Match Group fileshare
    ChrootDirectory /srv/share
    ForceCommand internal-sftp
    X11Forwarding no
    AllowTcpForwarding no
    PermitTunnel no
    PasswordAuthentication no
    PubkeyAuthentication yes
```

Prepare chroot layout (required ownership model):

```sh
doas mkdir -p /srv/share/home/alice/upload
doas chown root:wheel /srv/share /srv/share/home /srv/share/home/alice
doas chmod 755 /srv/share /srv/share/home /srv/share/home/alice
doas chown alice:fileshare /srv/share/home/alice/upload
doas chmod 770 /srv/share/home/alice/upload
```

Load Alice's key:

```sh
doas mkdir -p /home/alice/.ssh
doas vi /home/alice/.ssh/authorized_keys
doas chown -R alice:alice /home/alice/.ssh
doas chmod 700 /home/alice/.ssh
doas chmod 600 /home/alice/.ssh/authorized_keys
```

Restart SSH:

```sh
doas rcctl restart sshd
```

## 4) Expose safely with built-in IKEv2 VPN (`iked`)

Enable forwarding and filter traffic so SFTP is reachable only from VPN clients.

`/etc/sysctl.conf`:

```text
net.inet.ip.forwarding=1
```

Minimal `/etc/iked.conf` skeleton (replace with your certs/subnets):

```text
# Define certificate and trust settings (certpath/CA) per iked.conf(5).
# Example: set cert "/etc/iked/certs/vpn.example.com.fullchain.pem"
#          set key  "/etc/iked/private/vpn.example.com.key"
ikev2 "roadwarrior" passive esp \
    from 10.20.0.0/24 to 10.20.0.1/32 \
    peer any \
    srcid vpn.example.com \
    config address 10.20.0.0/24
```

Firewall example in `/etc/pf.conf`:

```text
set skip on lo
vpn_if = "enc0"
wan_if = "egress"

block all
pass out quick on $wan_if inet from ($wan_if)
pass in on $wan_if proto udp to ($wan_if) port {500,4500}   # IKEv2/IPsec
pass in on $vpn_if proto tcp from 10.20.0.0/24 to 10.20.0.1 port 22
```

Enable services:

```sh
doas rcctl enable iked pf sshd
doas rcctl restart iked
doas pfctl -f /etc/pf.conf && doas pfctl -e
```

## 5) Optional mTLS pattern (when VPN is not possible)

If clients cannot use VPN, publish files over HTTPS and require client certificates (mTLS).

- Use OpenBSD `httpd` to serve a read-only export (for example `/var/www/htdocs/files`).
- Put `relayd` in front, terminate TLS, and enforce client cert validation against your internal CA.
- Keep SFTP for write access, and keep upload paths off public HTTPS.

High-level relayd design:

1. Server cert/key on relayd
2. Trusted client CA bundle configured for verification
3. `relayd` denies connections without valid client certs
4. Proxy only to local `httpd` backend
5. Restrict inbound 443 in `pf` to expected source networks where possible

## 6) Hardening checklist

- Disable password auth for share users (`PasswordAuthentication no` in `Match Group fileshare`)
- Use per-user SSH keys; rotate keys regularly
- Keep OpenBSD updated (`syspatch`)
- Limit exposed ports with `pf`
- Log and review `/var/log/authlog` and `sftp-server` activity
- Back up `/srv/share` and encryption keys/certificates

## 7) Validation

From a VPN-connected client:

```sh
sftp -P 22 alice@10.20.0.1
```

If mTLS HTTPS is enabled:

```sh
curl --cert client.crt --key client.key --cacert ca.crt https://files.example.com/files/
```

---

This approach keeps the core file service in OpenBSD base (`sshd`, `pf`, `iked`) and adds mTLS as an optional distribution layer where VPN is unavailable.
