# Networking, Firewall & Reverse Proxy Setup

This document defines the networking foundation of the homelab, including:

* Static IP assignment
* SSH hardening
* Firewall configuration
* Internal Certificate Authority (CA)
* Local domain mapping
* Nginx Proxy Manager deployment
* Reverse proxy routing
* Current access model
* Future DNS architecture

The objective is to provide secure administration, centralized service routing, and a clean migration path toward AdGuard Home and Cloudflared.

---

## 1. Static IP Assignment

To ensure the homelab and client desktop/laptop are always reachable at the same address, configure a DHCP reservation on the router.

### Steps

1. Open the router administration page.

2. Locate:

   * DHCP Reservation
   * Static Lease
   * IP Reservation

3. Bind the server MAC address to:

   ```text
   # Homelab
   192.168.100.199

   # Desktop/Laptop
   192.168.100.190
   ```

4. Save the configuration.

5. Reboot the router if required.

---

## 2. Server Hardening

### 2.1 SSH Configuration

The default SSH port (22) is changed to reduce automated scanning and brute-force attempts.

#### Edit SSH Configuration

```bash
sudo nano /etc/ssh/sshd_config
```

Find:

```text
#Port 22
```

Replace with:

```text
Port 9999
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

#### Verify

Using hostname alias:

```bash
ssh <username>@homelab -p 9999
```

Or directly:

```bash
ssh <username>@192.168.100.199 -p 9999
```

---

### 2.2 UFW Firewall

The homelab follows a minimal-exposure model:

* Deny all incoming traffic by default
* Allow all outgoing traffic
* Open only required ports

#### Default Policy

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

#### Required Rules

```bash
# SSH
sudo ufw allow 9999/tcp comment 'SSH'

# Reverse Proxy
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# NPM Admin
sudo ufw allow from 192.168.100.190 to any port 81 comment 'NPM Admin'

# Allow Nginx Proxy Manager (inside Docker proxy network) to reach Cockpit (host on OS)
sudo ufw allow in on br-+ to any port 9090 comment 'Allow Docker Bridges to Cockpit'

# NFS Share
sudo ufw allow from 192.168.100.190 to any port 2049 comment 'NFS Client'

# Temporary Android Access
sudo ufw allow 2283/tcp comment 'Immich'
sudo ufw allow 4533/tcp comment 'Navidrome'
```

#### Enable UFW

```bash
sudo ufw enable
```

#### Verify

```bash
sudo ufw status verbose
```

Example:

```text
9999/tcp   ALLOW IN   Anywhere
80/tcp     ALLOW IN   Anywhere
443/tcp    ALLOW IN   Anywhere
81         ALLOW IN   192.168.100.190
2049       ALLOW IN   192.168.100.190
2283/tcp   ALLOW IN   Anywhere
4533/tcp   ALLOW IN   Anywhere
```

#### Security Note

Only Immich and Navidrome are exposed directly.

Administrative services remain protected behind Nginx Proxy Manager and are intended for trusted desktop/laptop clients only.

Exposed ports:

```text
2283 -> Immich
4533 -> Navidrome
```

Not publicly exposed through dedicated service ports:

```text
9090 -> Cockpit
9443 -> Portainer
```

NPM Admin remains accessible on:

```text
81 -> Nginx Proxy Manager Administration
```

These direct media ports are temporary and will be removed after internal DNS deployment.

---

## 3. Local Domain Mapping

A dedicated DNS server has not been deployed yet.

Desktop and laptop clients resolve homelab services through local hosts entries.

### Ubuntu Client

Edit:

```bash
sudo nano /etc/hosts
```

Add:

```text
# Homelab Host
192.168.100.199 homelab

# Homelab Services
192.168.100.199 system.homelab.local
192.168.100.199 portainer.homelab.local
192.168.100.199 music.homelab.local
192.168.100.199 gallery.homelab.local
192.168.100.199 npm.homelab.local
```

### Verify

```bash
ping music.homelab.local
```

Expected:

```text
192.168.100.199
```

---

## 4. Internal Certificate Authority (CA) (Optional)

To eliminate browser certificate warnings, the homelab uses a private Certificate Authority.

All internal HTTPS certificates are signed by this CA.

### Architecture

```text
MyHomelabCA
      │
      └── signs
             │
             └── homelab.crt
                    ├── *.homelab.local
                    ├── homelab.local
                    └── 192.168.100.199
```

---

### Install OpenSSL

```bash
sudo apt update && sudo apt install openssl -y
```

Verify:

```bash
openssl version
```

---

### Create Working Directory

```bash
mkdir -p ~/certs && cd ~/certs
```

---

### Generate Root CA Key

```bash
openssl genrsa \
  -out MyHomelabCA.key \
  4096
```

---

### Generate Root CA Certificate

```bash
openssl req \
  -x509 \
  -new \
  -nodes \
  -key MyHomelabCA.key \
  -sha256 \
  -days 3650 \
  -out MyHomelabCA.crt
```

Example:

```text
Common Name: HomelabCA
```

---

### Generate Server Key

```bash
openssl genrsa \
  -out homelab.key \
  2048
```

---

### Generate CSR

```bash
openssl req \
  -new \
  -key homelab.key \
  -out homelab.csr
```

Example:

```text
Common Name: homelab.local
```

---

### Create SAN Extension

```bash
nano homelab.ext
```

Content:

```ini
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature,nonRepudiation,keyEncipherment,dataEncipherment
subjectAltName=@alt_names

[alt_names]
DNS.1 = homelab.local
DNS.2 = *.homelab.local
IP.1 = 192.168.100.199
```

---

### Sign Certificate

```bash
openssl x509 \
  -req \
  -in homelab.csr \
  -CA MyHomelabCA.crt \
  -CAkey MyHomelabCA.key \
  -CAcreateserial \
  -out homelab.crt \
  -days 825 \
  -sha256 \
  -extfile homelab.ext
```

Generated files:

```text
MyHomelabCA.crt
MyHomelabCA.key

homelab.crt
homelab.key

MyHomelabCA.srl
homelab.csr
homelab.ext
```

---

### Install Root CA on Ubuntu Clients

```bash
sudo cp MyHomelabCA.crt \
/usr/local/share/ca-certificates/

sudo update-ca-certificates
```

**Note for Firefox users: Firefox maintains its own independent certificate store. If it still reports security warnings, manually import MyHomelabCA.crt via Firefox Settings -> Privacy & Security -> Manage Certificates -> Authorities -> Import.**

---

### Configure NPM SSL Certificate

Open:

```text
SSL Certificates
```

Upload:

```text
homelab.crt
homelab.key
```

Assign this certificate to:

```text
system.homelab.local
portainer.homelab.local
music.homelab.local
gallery.homelab.local
npm.homelab.local
```

---

## 5. Deploy Nginx Proxy Manager

Nginx Proxy Manager acts as the central reverse proxy.

All services communicate internally through Docker networks while NPM provides a single entry point for desktop and laptop users.

### Create Workspace

```bash
mkdir -p ~/docker/nginx-proxy-manager
cd ~/docker/nginx-proxy-manager
```

### Create Docker Compose

Create:

```text
docker-compose.yml
```

Reference:

```text
/docker/nginx-proxy-manager/docker-compose.yml
```

### Start NPM

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

Expected:

```text
nginx-proxy-manager
```

---

## 6. Configure Proxy Hosts

### Access Dashboard

```text
http://192.168.100.199:81
```

### Default Credentials

```text
Email: admin@example.com
Password: changeme
```

Change them immediately.

---

### Proxy Host Mapping

| Domain                  | Service   | Target                 |
| ----------------------- | --------- | ---------------------- |
| system.homelab.local    | Cockpit   | 192.168.100.199:9090   |
| portainer.homelab.local | Portainer | portainer:9443         |
| npm.homelab.local       | NPM       | nginx-proxy-manager:81 |
| music.homelab.local     | Navidrome | navidrome:4533         |
| gallery.homelab.local   | Immich    | immich_server:2283     |

Example:

```text
Domain Names:
music.homelab.local

Scheme:
http

Forward Hostname/IP:
navidrome

Forward Port:
4533
```

---

### ⚠️ Special Note for Cockpit Integration (system.homelab.local)

Cockpit runs directly on the Bare-Metal Host OS and enforces strict security mechanisms regarding HTTPS and Origin validation (CORS). To successfully route it through Nginx Proxy Manager, you must apply specific configurations on both the proxy and the host:

#### 1. NPM Configuration

* **Scheme:** Must be set to `https` (Cockpit rejects standard unencrypted HTTP requests at port 9090).
* **Forward Hostname / IP:** Use the Docker network gateway IP (e.g., `172.18.0.1`) or the server's static LAN IP (`192.168.100.199`).

#### 2. Host OS Configuration (CORS Whitelisting)

By default, Cockpit will drop connections coming from unfamiliar hostnames like `system.homelab.local`. You must manually authorize the domain.

1. Open (or create) the Cockpit configuration file on the server:

   ```bash
   sudo nano /etc/cockpit/cockpit.conf
   ```

2. Add the following lines to trust your local homelab domains:

    ```ini
    [WebService]
    Origins = https://system.homelab.local http://system.homelab.local
    AllowUnencrypted = true
    ```

3. Save and exit (Ctrl+O, Enter, Ctrl+X).

4. Restart the Cockpit service to immediately apply the security policy:

    ```bash
    sudo systemctl restart cockpit
    ```

Once restarted, Systemd Socket Activation will automatically wake Cockpit up whenever NPM forwards requests to port 9090, giving you a seamless, secure green-lock UI.

---

## 7. Current Access Model

### Desktop / Laptop

All services are accessed through Nginx Proxy Manager:

```text
https://system.homelab.local
https://portainer.homelab.local
https://music.homelab.local
https://gallery.homelab.local
https://npm.homelab.local
```

SSH:

```bash
ssh <username>@homelab -p 9999
```

---

### Android

Android devices currently do not use the homelab's internal name resolution.

Only media applications are exposed directly:

#### Immich

```text
http://192.168.100.199:2283
```

#### Navidrome

```text
http://192.168.100.199:4533
```

Administrative services are intentionally unavailable through direct ports:

* Cockpit
* Portainer
* Nginx Proxy Manager

This minimizes exposure while still allowing media applications to function.

---

## Future Improvements

Planned networking components:

* AdGuard Home
* Cloudflared Tunnel
* Public Domain Names
* Internal DNS Resolution

After deployment:

* Android devices will use domain names
* /etc/hosts entries can be removed
* Ports 2283 and 4533 can be removed from UFW
* All services will be accessed through Nginx Proxy Manager
* Internal and external access paths become unified
