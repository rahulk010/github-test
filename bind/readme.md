# BIND9 DNS Server Configuration (`skynet.lab`)

Complete configuration reference for the BIND9 authoritative and caching DNS server supporting a Kubernetes cluster and `external-dns` dynamic RFC2136 updates.

---

## 1. Environment Overview

| Parameter | Value |
| :--- | :--- |
| **Server Hostname** | `dns-bind.skynet.lab` |
| **DNS Server IP** | `10.13.10.251` |
| **Primary Domain** | `skynet.lab` |
| **Ingress / LoadBalancer Pool** | `192.168.0.0/24` |
| **Reverse Lookup Zone** | `0.168.192.in-addr.arpa` |
| **Upstream Forwarder** | `10.11.10.4` |
| **TSIG Key Name** | `externaldns-key` (`hmac-sha256`) |

---

## 2. TSIG Key Generation (Dynamic Updates)

Generate the TSIG key for `external-dns` authentication:

```bash
sudo tsig-keygen -a hmac-sha256 externaldns-key | sudo tee /etc/bind/externaldns.key
sudo chown root:bind /etc/bind/externaldns.key
sudo chmod 640 /etc/bind/externaldns.key
```

> **Note:** Retrieve the generated base64 secret using `sudo cat /etc/bind/externaldns.key` for your Kubernetes `external-dns` configuration.

---

## 3. Global Options (`/etc/bind/named.conf.options`)

```bind
acl internal-network {
    10.13.10.251/32;    // DNS server's own IP
    192.168.0.0/24;     // Kubernetes LoadBalancer / Ingress range
    172.30.0.0/16;      // Kubernetes node / management subnet
};

options {
    directory "/var/cache/bind";

    allow-query { localhost; internal-network; };
    allow-recursion { localhost; internal-network; };
    allow-transfer { localhost; };

    forwarders { 10.11.10.4; };
    recursion yes;
    dnssec-validation auto;

    listen-on { any; };
};
```

---

## 4. Zone Declarations (`/etc/bind/named.conf.local`)

Zone files are stored in `/var/cache/bind/` so BIND has write permissions (under Ubuntu/Debian AppArmor) to create `.jnl` journal files for dynamic updates.

```bind
include "/etc/bind/externaldns.key";

zone "skynet.lab" IN {
    type master;
    file "/var/cache/bind/forward.skynet.lab";
    allow-update { key "externaldns-key"; };
};

zone "0.168.192.in-addr.arpa" IN {
    type master;
    file "/var/cache/bind/reverse.skynet.lab";
    allow-update { key "externaldns-key"; };
};
```

---

## 5. Zone Files (`/var/cache/bind/`)

### Forward Zone (`/var/cache/bind/forward.skynet.lab`)

```bind
\$TTL 300
@ IN SOA dns-bind.skynet.lab. admin.skynet.lab. (
    2026111602  ; Serial (YYYYMMDDNN)
    3600        ; Refresh (1 hour)
    900         ; Retry (15 minutes)
    604800      ; Expire (1 week)
    300         ; Minimum TTL (5 minutes)
)

; Name Server Record
@           IN NS   dns-bind.skynet.lab.

; DNS Server Record
dns-bind    IN A    10.13.10.251

; Optional Static Ingress / Wildcard Record (adjust IP to match LB)
ingress     IN A    192.168.0.10
*           IN A    192.168.0.10
```

### Reverse Zone (`/var/cache/bind/reverse.skynet.lab`)

```bind
\$TTL 300
@ IN SOA dns-bind.skynet.lab. admin.skynet.lab. (
    2026111601  ; Serial (YYYYMMDDNN)
    3600        ; Refresh (1 hour)
    900         ; Retry (15 minutes)
    604800      ; Expire (1 week)
    300         ; Minimum TTL (5 minutes)
)

; Name Server Record
@       IN NS   dns-bind.skynet.lab.

; PTR Records (192.168.0.x)
10      IN PTR  ingress.skynet.lab.
```

---

## 6. Permissions & Verification Commands

### Set Ownership & Permissions
```bash
sudo chown -R bind:bind /var/cache/bind/
sudo chmod 644 /var/cache/bind/forward.skynet.lab /var/cache/bind/reverse.skynet.lab
```

### Validate Syntax & Reload Service
```bash
sudo named-checkconf
sudo named-checkzone skynet.lab /var/cache/bind/forward.skynet.lab
sudo named-checkzone 0.168.192.in-addr.arpa /var/cache/bind/reverse.skynet.lab
sudo systemctl reload named
```

### Test DNS Resolution
```bash
# Verify authoritative forward lookup
dig @10.13.10.251 dns-bind.skynet.lab +short

# Verify reverse lookup
dig @10.13.10.251 -x 192.168.0.10 +short

# Verify upstream forwarding
dig @10.13.10.251 google.com +short
```

---

## 7. Operational Notes for Dynamic Zones

* **Manual Edits:** Because `allow-update` is enabled, never edit `/var/cache/bind/forward.skynet.lab` or `reverse.skynet.lab` directly while `named` is running.
* **Safe Manual Editing Workflow:**
  ```bash
  sudo rndc freeze skynet.lab
  # Edit /var/cache/bind/forward.skynet.lab and increment the Serial
  sudo rndc reload skynet.lab
  sudo rndc thaw skynet.lab
  ```
