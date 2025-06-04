# How to Deploy a Zero Trust Cluster


🚀 **Teleport** is a modern open-source platform for secure access to your infrastructure: SSH, Kubernetes, databases, web apps, and even Windows desktops (RDP).

---

## Teleport Setup open-source ZTNA Platform

  ✅ Requirements
- A Linux server with a public IP address and DNS name (e.g., `teleport.<your-domain>`)
- One or more Windows and Linux machines
- Basic terminal access (Linux + Windows)

---

### 📌 Step 1: Configure DNS

Create the following A records for your domain:

```
`teleport.<your-domain>`      → your Linux server public IP  
*.`teleport.<your-domain>`    → your Linux server public IP
```

This is required for Let's Encrypt (ACME) to issue a valid TLS certificate.

---

### 📥 Step 2: Install Teleport on Linux

Install the latest stable version of Teleport:

```bash
curl https://cdn.teleport.dev/install.sh | bash -s 17.4.8
```

---

### ⚙️ Step 3: Generate Teleport Configuration

```bash
sudo teleport configure -o file   --acme   --acme-email=`teleport@yourdomain.com`   --cluster-name=`teleport.<your-domain>`
```

This creates `/etc/teleport.yaml` with TLS auto-renewal using Let's Encrypt.

My working configuration

```yaml
version: v3
teleport:
  nodename: teleport.<your-domain>
  data_dir: /var/lib/teleport
  log:
    output: stderr
    severity: INFO
    format:
      output: text
auth_service:
  enabled: true
  listen_addr: 0.0.0.0:3025
  cluster_name: teleport.<your-domain>
  proxy_listener_mode: multiplex
proxy_service:
  enabled: true
  web_listen_addr: 0.0.0.0:443
  public_addr: teleport.<your-domain>:443
  acme:
    enabled: true
    email: teleport@<your-domain>
windows_desktop_service:
  enabled: "no"
ssh_service:
  enabled: "no"
```

---

### ▶️ Step 4: Start Teleport

```bash
sudo systemctl enable teleport
sudo systemctl start teleport
```

Then open your browser and visit:

```
https://teleport.<your-domain>:443
```

---

### 👤 Step 5: Create a Teleport Admin User

```bash
sudo tctl users add teleport-admin   --roles=editor,access   --logins=root,ubuntu
```

Teleport will print an invite link like:

```
https://teleport.<your-domain>:443/web/invite/abc123...
```

Open the link to set a password and register your MFA device.

---

## 🔐 Configure Windows RDP Access

### 🪟 Step 1: Prepare Windows for RDP
1. On your **Windows machine**, open **Command Prompt (cmd.exe)** as Administrator and dowload:
```cmd
curl.exe -fo teleport.cer https://teleport.<your-domain>/webapi/auth/export?type=windows
curl.exe -fo teleport-windows-auth-setup-v17.4.8-amd64.exe https://cdn.teleport.dev/teleport-windows-auth-setup-v17.4.8-amd64.exe
```
2. Double-click the `.exe` installer to run the **Teleport Windows Auth Setup** program.
3. Choose certificat
4. Restart the computer after installation.

### 🔐 Step 2: Generate Join Token for RDP on Linux Server
1. SSH into your Teleport Auth Server and run:
```bash
tctl tokens add --type=windowsdesktop
```
2. Copy the token output to your RDP proxy server, e.g., `/tmp/token`

### ⚙️ Step 2: Configure windows_desktop_service on RDP Proxy
Update or create the relevant section in `/etc/teleport.yaml`:
```yaml
windows_desktop_service:
  enabled: "yes"
  static_hosts:
    - name: Windows
      ad: false
      addr: 192.168.1.10:3389
      labels:
        server: home
        host: Win10
```
Then restart the service:
```bash
sudo systemctl restart teleport
```

---

## 🌐 Configure Internal Web Application Access

### 🔐 Step 1: Generate Join Token for App Service
Run on the Auth server:
```bash
tctl tokens add --type=app
```
Copy the token to the target server that will proxy your internal app.

### ⚙️ Step 2: Configure app_service in teleport.yaml
Edit `/etc/teleport.yaml` on the node that will serve the app:
```yaml
app_service:
  enabled: "yes"
  apps:
    - name: opnsense
      uri: "https://192.168.1.1:8080"
      public_addr: "opnsense.teleport.<your-domain>.com"
      insecure_skip_verify: true
      labels:
        env: net
        host: OPNsense
```
Restart teleport
```bash
sudo systemctl restart teleport
```

---

## 🔍 Troubleshooting

- **ACME TLS fails?**  
  - Ensure port 80 and 443 are open  
  - Check DNS records  

- **Windows host not visible?**  
  - Check RDP is enabled on the Windows machine  
  - Check firewall rules  

- **Web UI unreachable?**  
  - Check `teleport.yaml`  
  - Check logs: `journalctl -u teleport` or `docker logs teleport`

---

💡 *Need a Docker setup? Want to automate the RDP proxy deployment? Check out my GitHub repo or contact me directly.*


