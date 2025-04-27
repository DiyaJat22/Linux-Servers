# 🔧 Dynamic DNS (DDNS) Setup & Troubleshooting Guide for Debian (BIND)

A step-by-step guide to configure secure Dynamic DNS (DDNS) with BIND on Debian and integrate it with DHCP and Windows clients using TSIG authentication.

---

## 🧭 Overview

- **OS**: Debian (any stable release)
- **DNS Server**: BIND9
- **DHCP Server**: `isc-dhcp-server`
- **Client**: Windows (10/11)
- **Security**: TSIG key-based updates (RFC 2136)

---

## ⚙️ Step-by-Step Setup

### Step 1: Install Required Packages
```bash
apt update
```
```bash
apt install bind9 bind9-utils isc-dhcp-server dnsutils
```

## ⚙️ Step 2: Generate TSIG Key

Always show details

```bash
tsig-keygen -a hmac-sha256 win-client-key
```
`Save output to a file, e.g. /etc/bind/win-client.key`

## ⚙️ Step 3: Configure BIND
`/etc/bind/named.conf.local`

```bash
vim /etc/bind/named.conf.local
```
```bash
include "/etc/bind/win-client.key"; # or define key

zone "diya.local" {
    type master;
    file "/etc/bind/zones/db.diya.local";
    allow-update { any; }; # using any for ddns
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192";
    allow-update { any; }; # using any for ddns
};
```

```bash
vim /etc/bind/named.conf.options
```
```bash
options {
    directory "/var/cache/bind";
    allow-query { any; };
    allow-recursion { 127.0.0.1; };
    listen-on { any; };
    listen-on-v6 { any; };
};
```

## ⚙️ Step 4: Create Zone Files
```bash
vim /etc/bind/zones/diya.local.zone
```
```bash
$TTL 86400
@   IN  SOA ns1.diya.local. admin.diya.local. (
        2025042601 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum

    IN  NS  ns1.diya.local.
ns1 IN  A   192.168.1.1
```

```bash
vim /etc/bind/zones/1.168.192.zone
```
```bash
$TTL 86400
@   IN  SOA ns1.diya.local. admin.diya.local. (
        2025042601 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum

    IN  NS  ns1.diya.local.
1   IN  PTR ns1.diya.local.
```

---

## ⚙️ Step 5: Set Permissions
```bash
chown -R bind:bind /etc/bind/zones
```

---

## ⚙️ Step 6: Configure DHCP Server
```bash
vim /etc/dhcp/dhcpd.conf
```
```bash
ddns-update-style interim;
ignore client-updates;
update-static-leases on;

key "win-client-key" {
    algorithm hmac-sha256;
    secret "<YOUR_GENERATED_SECRET>";
};

zone diya.local. {
    primary 192.168.1.1;
    key win-client-key;
};

zone 1.168.192.in-addr.arpa. {
    primary 192.168.1.1;
    key win-client-key;
};

subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name "diya.local";
    option domain-name-servers 192.168.1.1;

    ddns-domainname "diya.local.";
    ddns-rev-domainname "1.168.192.in-addr.arpa.";
};
```

---

## ⚙️ Step 7: Enable and Restart Services
```bash
systemctl enable bind9 isc-dhcp-server
```
```bash
systemctl restart bind9 isc-dhcp-server
```

---

## 💻 Client-Side: Windows

### Step A: Enable DDNS
1. Open `ncpa.cpl`
2. Right-click network adapter → Properties → IPv4 → Advanced → DNS tab
3. Enable:
   - ✅ Register this connection’s addresses in DNS
   - ✅ Use this connection’s DNS suffix in DNS registration
4. Add DNS suffix: `diya.local`

### Step B: Configure DNS Server IP
Set your Debian server IP as Preferred DNS Server (e.g., 192.168.1.1)

---

## 🧪 Testing DDNS Updates

### ✅ Option 1: Windows Native
```cmd
ipconfig /registerdns
```

### ✅ Option 2: Using `nsupdate` (Linux)

1. Run:
```bash
 nsupdate -k /etc/bind/win-client.key
```

#### 🔄 Add A and PTR Records
1. Create `update.txt`:
```txt
server 192.168.1.1
zone diya.local
update delete test.diya.local. A
update add test.diya.local. 300 A 192.168.1.146
send

zone 1.168.192.in-addr.arpa
update delete 146.1.168.192.in-addr.arpa. PTR
update add 146.1.168.192.in-addr.arpa. 300 PTR test.diya.local.
send
```

---


### 🔍 Check Zone Status with `rndc`
```bash
rndc status
```

```bash
rndc zonestatus diya.local
```
```bash
rndc zonestatus 1.168.192.in-addr.arpa
```

---

## 🔍 Logs & Troubleshooting

### View Logs (Default)
```bash
tail -f /var/log/syslog | grep named
```

### With journalctl (systemd)
```bash
journalctl -u named -f
```
```bash
journalctl -u named | grep update
```

### Sample Success Log
```log
client 192.168.1.146#54321: updating zone 'diya.local/IN': adding an RR at 'win11.diya.local' A
update approved
```

### Common Errors & Fixes
#### ❌ `update denied`
- Key mismatch or missing
- Fix: Ensure TSIG key is correct on both client/server

#### ❌ `permission denied` or `journal open failed`
- Cause: Wrong file permissions or SELinux context
- Fix:
```bash
chown bind:bind /etc/bind/zones/diya.local.zone
```
```bash
restorecon -v /etc/bind/zones/diya.local.zone
```

---

## 🧰 More Troubleshooting Commands

### Check BIND Configuration
```bash
named-checkconf
```

### Check Zone File Syntax
```bash
named-checkzone diya.local /etc/bind/zones/diya.local.zone
```
```bash
named-checkzone 1.168.192 /etc/bind/zones/1.168.182.zone

### Test DNS Query
```bash
dig win-11.diya.local @192.168.1.1
```

### List DNS Records for Zone
```bash
dig diya.local AXFR @192.168.1.1
```

### Query with Debug Mode
```bash
dig diya.local @localhost +norecurse +noall +answer
```

### Manual Dynamic Update Debug
```bash
nsupdate -d update.txt
```

### Check SELinux Status (if enabled)
```bash
sestatus
```

### Check Zone Journal File
```bash
ls -l /etc/bind/zones/*.jnl
```

### Restart BIND and Monitor Logs
```bash
systemctl restart named
```
```bash
journalctl -u named -f
```

---

## 📂 Journaling in Dynamic DNS
Dynamic updates are stored in `.jnl` journal files. They are not written directly into zone files until:
- You stop named
- Run `rndc freeze`, `rndc thaw`
- Or run `rndc sync`

To view updates:
```bash
dig win-11.diya.local @localhost
```

To flush journal into zone file:
```bash
rndc sync -clean diya.local
```

```bash
rndc sync
```

---

