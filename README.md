# Certificates-in-Corporate-Networks-when-using-VM-in-AD-windows-Host

---

## Step 1: Get All Proxy Settings

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings"
```

Note down:

ProxyServer   → proxy2.corp.com:8080

---

## Step 2: Get System Level Proxy

```cmd
netsh winhttp show proxy
```

---

## Step 3: Resolve Proxy to IP

```cmd
nslookup proxy2.corp.com
```

Note down both hostname and IP.

---

## Step 4: Get Network Details

```cmd
ipconfig /all
```

Note down:

DNS Servers     : 10.10.1.1

Default Gateway : 10.10.0.1

---

## Step 5: Get Domain Information

```cmd
echo %USERDNSDOMAIN%
```

```cmd
echo %LOGONSERVER%
```

```cmd
echo %USERDOMAIN%
```

---

## Step 6: Export Corporate Root CA Certificate

Win+R → certmgr.msc

Trusted Root Certification Authorities → Certificates

Find corporate root CA → Right-click → All Tasks → Export

Format: Base-64 encoded X.509 (.CER)

Filename: corp-ca.crt → Save to Desktop

## Step 8: Export Cisco Umbrella CA Certificate (if present)

```
Same certmgr.msc window
```
```
Find: Cisco Umbrella Primary SubCA
Right-click → All Tasks → Export
```
```
Format: Base-64 encoded X.509 (.CER)
Filename: umbrella-ca.crt → Save to Desktop
```

---

## Step 9: Check if Cisco Umbrella Roaming Client is Installed

```cmd
tasklist | findstr -i umbrella
```

```cmd
tasklist | findstr -i cisco
```

Note whether it is running — this affects home network too.

---

## Step 10: Master Reference Table

| Setting | Your Value |
|---|---|
| Proxy hostname | `proxy2.corp.com` |
| Proxy IP | `10.10.1.5` |
| Proxy port | `8080` |
| Auth type | `Kerberos/Negotiate` |
| Domain name | `CORP.COM` |
| Domain controller | `dc1.corp.com` |
| DNS server | `10.10.1.1` |
| Corp CA cert | `corp-ca.crt` |
| Umbrella CA cert | `umbrella-ca.crt` |
| Cisco Umbrella | Running / Not Running |

---
---

## Step 11: Set Network Adapter to NAT

```
VirtualBox → Select Kali VM → Settings
```
```
Network → Adapter 1 → Attached to: NAT
```
```
Advanced → Promiscuous Mode: Allow All
```
```
Click OK
```

---

## Step 12: Boot Kali and Verify Network

```bash
ip a
```

```bash
ping -c 3 10.0.2.2
```

---

## Step 16: Install Corporate Root CA System-Wide

```bash
sudo cp /home/kali/Desktop/corp-ca.crt /usr/local/share/ca-certificates/
```

```bash
sudo update-ca-certificates
```

---

## Step 17: Install Cisco Umbrella CA System-Wide (if applicable)

```bash
sudo cp /home/kali/Desktop/umbrella-ca.crt /usr/local/share/ca-certificates/
```

```bash
sudo update-ca-certificates
```

---

## Step 18: Install Both CA Certs into Firefox

```bash
certutil -A \
  -n "Corporate CA" \
  -t "CT,," \
  -i /home/kali/Desktop/corp-ca.crt \
  -d ~/.mozilla/firefox/39nfjb0i.default-esr
```

```bash
certutil -A \
  -n "Cisco Umbrella CA" \
  -t "CT,," \
  -i /home/kali/Desktop/umbrella-ca.crt \
  -d ~/.mozilla/firefox/39nfjb0i.default-esr
```

```bash
sudo certutil -A \
  -n "PortSwigger" \      
  -t "CT,," \
  -i /home/kali/Desktop/burp-ca.crt \
  -d ~/.mozilla/firefox/39nfjb0i.default-esr
```

---

## Step 19: Enable Firefox to Trust System Certificate Store

Open Firefox → address bar:

```
about:config
```

Search:

```
security.enterprise_roots.enabled
```

Double-click → set to `true`

Restart Firefox.

---

## Step 21: Create pip Config (Permanent)

```bash
mkdir -p ~/.config/pip
```

```bash
nano ~/.config/pip/pip.conf
```

```
[global]
cert = /etc/ssl/certs/ca-certificates.crt
trusted-host = pypi.org
               pypi.python.org
               files.pythonhosted.org
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

---

## Step 22: Create the Switch Scripts

### Corporate Profile Script

```bash
sudo nano /usr/local/bin/proxy-corp.sh
```

```bash
#!/bin/bash
echo ""
echo "================================================"
echo "     SWITCHING TO CORPORATE NETWORK PROFILE"
echo "================================================"
echo ""

# Step 1: Set corporate DNS
echo "[1/8] Setting corporate DNS..."
sudo chattr -i /etc/resolv.conf 2>/dev/null
sudo bash -c 'cat > /etc/resolv.conf << EOF
nameserver 10.10.1.1
nameserver 10.10.1.2
nameserver 8.8.8.8
EOF'
sudo chattr +i /etc/resolv.conf
echo "      Done."

# Step 2: Set system-wide proxy
echo "[2/8] Setting system-wide proxy..."
sudo bash -c 'cat > /etc/environment << EOF
http_proxy="http://proxy2.corp.com:8080"
https_proxy="http://proxy2.corp.com:8080"
ftp_proxy="http://proxy2.corp.com:8080"
HTTP_PROXY="http://proxy2.corp.com:8080"
HTTPS_PROXY="http://proxy2.corp.com:8080"
FTP_PROXY="http://proxy2.corp.com:8080"
no_proxy="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
NO_PROXY="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
EOF'
echo "      Done."

# Step 3: Set apt proxy
echo "[3/8] Setting apt proxy..."
sudo bash -c 'cat > /etc/apt/apt.conf.d/99proxy << EOF
Acquire::http::Proxy "http://proxy2.corp.com:8080";
Acquire::https::Proxy "http://proxy2.corp.com:8080";
Acquire::ftp::Proxy "http://proxy2.corp.com:8080";
EOF'
echo "      Done."

# Step 4: Set curl proxy
echo "[4/8] Setting curl proxy..."
cat > ~/.curlrc << EOF
proxy = "http://proxy2.corp.com:8080"
cacert = /etc/ssl/certs/ca-certificates.crt
EOF
echo "      Done."

# Step 5: Set wget proxy
echo "[5/8] Setting wget proxy..."
cat > ~/.wgetrc << EOF
http_proxy = http://proxy2.corp.com:8080
https_proxy = http://proxy2.corp.com:8080
ftp_proxy = http://proxy2.corp.com:8080
use_proxy = on
ca_certificate = /etc/ssl/certs/ca-certificates.crt
EOF
echo "      Done."

# Step 6: Set git proxy
echo "[6/8] Setting git proxy..."
git config --global http.proxy http://proxy2.corp.com:8080
git config --global https.proxy http://proxy2.corp.com:8080
git config --global http.sslCAInfo /etc/ssl/certs/ca-certificates.crt
echo "      Done."

# Step 7: Set pip proxy
echo "[7/8] Setting pip proxy..."
mkdir -p ~/.config/pip
cat > ~/.config/pip/pip.conf << EOF
[global]
proxy = http://proxy2.corp.com:8080
cert = /etc/ssl/certs/ca-certificates.crt
trusted-host = pypi.org
               pypi.python.org
               files.pythonhosted.org
EOF
echo "      Done."

# Step 8: Set root proxy
echo "[8/8] Setting root user proxy..."
sudo bash -c 'cat >> /root/.bashrc << EOF

# Corporate proxy
export http_proxy="http://proxy2.corp.com:8080"
export https_proxy="http://proxy2.corp.com:8080"
export HTTP_PROXY="http://proxy2.corp.com:8080"
export HTTPS_PROXY="http://proxy2.corp.com:8080"
export no_proxy="localhost,127.0.0.1,10.0.0.0/8"
export NO_PROXY="localhost,127.0.0.1,10.0.0.0/8"
EOF'
echo "      Done."

echo ""
echo "================================================"
echo "  Corporate profile active."
echo ""
echo "  NEXT STEPS:"
echo "  1. Run: source /etc/environment"
echo "  2. Run: kinit youruser@CORP.COM"
echo "  3. Run: klist  (verify ticket)"
echo "================================================"
echo ""
```

```bash
sudo chmod +x /usr/local/bin/proxy-corp.sh
```

---

### Home Profile Script

```bash
sudo nano /usr/local/bin/proxy-home.sh
```

```bash
#!/bin/bash
echo ""
echo "================================================"
echo "       SWITCHING TO HOME NETWORK PROFILE"
echo "================================================"
echo ""

# Step 1: Set home DNS
echo "[1/8] Setting home DNS..."
sudo chattr -i /etc/resolv.conf 2>/dev/null
sudo bash -c 'cat > /etc/resolv.conf << EOF
nameserver 192.168.1.1
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF'
echo "      Done."

# Step 2: Clear system-wide proxy
echo "[2/8] Clearing system-wide proxy..."
sudo bash -c '> /etc/environment'
echo "      Done."

# Step 3: Clear apt proxy
echo "[3/8] Clearing apt proxy..."
sudo bash -c '> /etc/apt/apt.conf.d/99proxy'
echo "      Done."

# Step 4: Clear curl proxy
echo "[4/8] Clearing curl proxy..."
cat > ~/.curlrc << EOF
cacert = /etc/ssl/certs/ca-certificates.crt
EOF
echo "      Done."

# Step 5: Clear wget proxy
echo "[5/8] Clearing wget proxy..."
cat > ~/.wgetrc << EOF
use_proxy = off
ca_certificate = /etc/ssl/certs/ca-certificates.crt
EOF
echo "      Done."

# Step 6: Clear git proxy
echo "[6/8] Clearing git proxy..."
git config --global --unset http.proxy 2>/dev/null
git config --global --unset https.proxy 2>/dev/null
git config --global http.sslCAInfo /etc/ssl/certs/ca-certificates.crt
echo "      Done."

# Step 7: Clear pip proxy
echo "[7/8] Clearing pip proxy..."
mkdir -p ~/.config/pip
cat > ~/.config/pip/pip.conf << EOF
[global]
cert = /etc/ssl/certs/ca-certificates.crt
trusted-host = pypi.org
               pypi.python.org
               files.pythonhosted.org
EOF
echo "      Done."

# Step 8: Clear root proxy and destroy Kerberos ticket
echo "[8/8] Clearing root proxy and Kerberos ticket..."
sudo bash -c "sed -i '/# Corporate proxy/,+6d' /root/.bashrc"
kdestroy 2>/dev/null
echo "      Done."

# Restart NetworkManager for fresh DHCP
sudo systemctl restart NetworkManager

echo ""
echo "================================================"
echo "  Home profile active."
echo ""
echo "  NEXT STEPS:"
echo "  1. Run: source /etc/environment"
echo "  2. Open new terminal for clean environment"
echo "  3. Test: curl -I https://google.com"
echo "================================================"
echo ""
```

```bash
sudo chmod +x /usr/local/bin/proxy-home.sh
```

---

### Status Check Script

```bash
sudo nano /usr/local/bin/proxy-status.sh
```

```bash
#!/bin/bash
echo ""
echo "================================================"
echo "          CURRENT NETWORK PROFILE STATUS"
echo "================================================"

echo ""
echo "[1] ACTIVE PROFILE"
PROXY=$(cat /etc/environment 2>/dev/null | grep http_proxy)
if [ -z "$PROXY" ]; then
    echo "      HOME profile active"
else
    echo "      CORPORATE profile active"
    echo "      $PROXY"
fi

echo ""
echo "[2] DNS SERVERS"
cat /etc/resolv.conf | grep nameserver

echo ""
echo "[3] PROXY ENVIRONMENT"
ENV_PROXY=$(printenv | grep -i proxy)
if [ -z "$ENV_PROXY" ]; then
    echo "      No proxy in environment"
else
    echo "$ENV_PROXY"
fi

echo ""
echo "[4] APT PROXY"
cat /etc/apt/apt.conf.d/99proxy 2>/dev/null || echo "      No apt proxy set"

echo ""
echo "[5] GIT PROXY"
GIT_PROXY=$(git config --global --list 2>/dev/null | grep proxy)
if [ -z "$GIT_PROXY" ]; then
    echo "      No git proxy set"
else
    echo "      $GIT_PROXY"
fi

echo ""
echo "[6] KERBEROS TICKET"
klist 2>/dev/null || echo "      No ticket — run: kinit user@CORP.COM"

echo ""
echo "[7] NETWORK INTERFACE"
ip a | grep -E "inet |state " | grep -v "127.0.0.1"

echo ""
echo "[8] INTERNET TEST"
curl -s -o /dev/null -w "      HTTP Status: %{http_code}\n" https://google.com

echo ""
echo "================================================"
```

```bash
sudo chmod +x /usr/local/bin/proxy-status.sh
```

---
---

# PART 4: DAILY USAGE

---

## When You Arrive at the Office

### On Windows — Enable Proxy:
```
Settings → Network & Internet → Proxy → Turn on
```

### On Kali:

```bash
sudo proxy-corp.sh
```

```bash
source /etc/environment
```

```bash
kinit youruser@CORP.COM
```

```bash
klist
```

```bash
sudo proxy-status.sh
```

---

## When You Get Home

### On Windows — Disable Proxy:
```
Settings → Network & Internet → Proxy → Turn off
```

### On Kali:

```bash
sudo proxy-home.sh
```

```bash
source /etc/environment
```

```bash
sudo proxy-status.sh
```

---

## Check Current Profile Anytime

```bash
sudo proxy-status.sh
```

---

## Manually Renew Kerberos Ticket (Corporate Only)

```bash
kinit -R
```

```bash
klist
```

---
---

# MASTER CHECKLIST

## One-Time Setup

| # | Task | Done |
|---|---|---|
| 1 | Proxy details collected from Windows | ☐ |
| 2 | Domain + DC info collected | ☐ |
| 3 | Corp CA cert exported | ☐ |
| 4 | Cisco Umbrella CA cert exported | ☐ |
| 5 | VirtualBox set to NAT | ☐ |
| 6 | Shared folder configured | ☐ |
| 7 | Packages installed (krb5, libnss3-tools) | ☐ |
| 8 | Corp CA installed system-wide | ☐ |
| 9 | Umbrella CA installed system-wide | ☐ |
| 10 | Both CAs imported into Firefox | ☐ |
| 11 | Firefox enterprise roots enabled | ☐ |
| 12 | /etc/krb5.conf configured | ☐ |
| 13 | proxy-corp.sh created and executable | ☐ |
| 14 | proxy-home.sh created and executable | ☐ |
| 15 | proxy-status.sh created and executable | ☐ |

## Every Time You Switch Networks

| # | Corporate | Home |
|---|---|---|
| 1 | Enable proxy on Windows | Disable proxy on Windows |
| 2 | `sudo proxy-corp.sh` | `sudo proxy-home.sh` |
| 3 | `source /etc/environment` | `source /etc/environment` |
| 4 | `kinit user@CORP.COM` | Not needed |
| 5 | `sudo proxy-status.sh` | `sudo proxy-status.sh` |

---

> **Four values drive everything** — replace `proxy2.corp.com`, `8080`, `CORP.COM`, `dc1.corp.com` and `10.10.1.1` with your actual values from the master reference table in Step 10. Everything else stays the same.
