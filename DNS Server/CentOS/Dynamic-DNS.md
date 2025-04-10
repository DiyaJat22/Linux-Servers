# Dynamic DNS (DDNS) Setup & Troubleshooting Guide for CentOS 9 BIND Server

This guide walks you through setting up and troubleshooting a secure Dynamic DNS (DDNS) configuration on a CentOS 9 server using BIND, integrated with Windows clients.

---

## 🧭 Overview
- **Server**: CentOS 9 with BIND
- **Client**: Windows (e.g., 10/11)
- **Security**: Uses TSIG keys for authenticated DDNS (RFC 2136 compliant)

---

## 🛠️ Server-Side: CentOS 9 (BIND Configuration)

### Step 1: Install BIND
```bash
dnf install bind bind-utils
```
```bash
systemctl enable --now named
```

### Step 2: Generate a TSIG Key
```bash
tsig-keygen -a hmac-sha256 win-client-key
```
Save the output block:
```conf
key "win-client-key" {
    algorithm hmac-sha256;
    secret "<YOUR_GENERATED_SECRET>";
};
```

### Step 3: Configure BIND
Edit `/etc/named.conf`:
```conf
key "win-client-key" {
    algorithm hmac-sha256;
    secret "<YOUR_GENERATED_SECRET>";
};

zone "diya.local" IN {
    type master;
    file "/var/named/diya.local.zone";
    allow-update { key win-client-key; };
};
```

### Step 4: Create Zone File `/var/named/diya.local.zone`
```dns
$TTL 86400
@   IN  SOA ns1.diya.local. admin.diya.local. (
        2025040901 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum

    IN  NS  ns1.diya.local.
ns1 IN  A   192.168.1.1
```

### Step 5: Set Permissions
```bash
chown named:named /var/named/diya.local.zone
```
```bash
restorecon -v /var/named/diya.local.zone
```

### Step 6: Configure Firewall
```bash
firewall-cmd --add-service=dns --permanent
```
```bash
firewall-cmd --reload
```

### Step 7: Restart BIND
```bash
systemctl restart named
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
Set your CentOS server IP as Preferred DNS Server (e.g., 192.168.1.1)

---

## 🧪 Testing DDNS Updates
### Option 1: Windows Native
```cmd
ipconfig /registerdns
```

### Option 2: Using `nsupdate` (Linux)
1. Create `update.txt`:
```txt
server 192.168.1.1
zone diya.local
key win-client-key <SECRET>
update delete test.diya.local. A
update add test.diya.local. 300 A 192.168.1.146
send
```
2. Run:
```bash
nsupdate -v update.txt
```

---

## 🔍 Logs & Troubleshooting

### View Logs (Default)
```bash
tail -f /var/log/messages | grep named
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
chown named:named /var/named/diya.local.zone
```
```bash
restorecon -v /var/named/diya.local.zone
```

---

## 🧰 More Troubleshooting Commands

### Check BIND Configuration
```bash
named-checkconf /etc/named.conf
```

### Check Zone File Syntax
```bash
named-checkzone diya.local /var/named/diya.local.zone
```

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
ls -l /var/named/*.jnl
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

---

## 🛡️ Security Note
- Avoid using `allow-update { any; };` in production.
- Secure the TSIG key file permissions:
```bash
chmod 640 /etc/named.conf
```
```bash
chmod 600 /var/named/win-client-key.*
```

---

Need help setting up a proxy update script or migrating to `/etc/bind`? Ping me!

