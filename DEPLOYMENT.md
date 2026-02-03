# Deploying Sheets-MySQL Sync to Production

This guide walks you through hosting the full stack application on a Virtual Private Server (VPS) like **DigitalOcean, AWS EC2, Linode, or Vultr**.

## 📋 Prerequisites

1.  **A VPS**: Ubuntu 22.04 LTS (Recommended). 2GB RAM / 1 vCPU is sufficient.
2.  **Domain Name**: (Optional) Required for HTTPS if you don't use a tunnel.
3.  **Google Cloud Console Project**: You need your `client_id`, `client_secret`, etc.

---

## 🚀 Step 1: Provision & Secure Server

SSH into your server:
```bash
ssh root@your-server-ip
```

Update packages and install essential tools:
```bash
apt update && apt upgrade -y
apt install -y curl git unzip
```

### Install Docker Engine & Docker Compose
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install Docker
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## 📂 Step 2: Deploy Codebase

Clone your repository (or upload files via SCP):
```bash
git clone https://github.com/your-username/sheets-mysql-sync.git
cd sheets-mysql-sync
```

Create your production `.env` file:
```bash
cp .env.example .env
nano .env
```

**Critical Environment Variables to Set:**
-   `NODE_ENV=production`
-   `MYSQL_PASSWORD` (Change this to a strong password!)
-   `GOOGLE_CLIENT_ID` / `SECRET` / `REFRESH_TOKEN`
-   `WEBHOOK_SECRET` (Generate a random string)

---

## 🐳 Step 3: Launch Services

Run the application in detached mode:
```bash
docker compose up -d --build
```

Check logs to ensure everything is running:
```bash
docker compose logs -f
```

---

## 🌐 Step 4: Expose to the Internet

Since Google Sheets needs to send webhooks to your server, it must be accessible via a public URL (HTTPS).

### Option A: Cloudflare Tunnel (Easiest & Most Secure)
This exposes your localhost `3000` to the internet without opening ports.

1.  Install `cloudflared` on the VPS.
2.  Run `cloudflared tunnel login`.
3.  Start a tunnel:
    ```bash
    cloudflared tunnel --url http://localhost:3000
    ```
4.  Copy the `https://....trycloudflare.com` URL.

### Option B: Nginx + Certbot (Standard Way)
If you have a domain (e.g., `api.yourdomain.com`).

1.  **Install Nginx**:
    ```bash
    apt install -y nginx
    ```
2.  **Configure Nginx Proxy**:
    Edit `/etc/nginx/sites-available/default`:
    ```nginx
    server {
        server_name api.yourdomain.com;
        location / {
            proxy_pass http://localhost:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
        }
    }
    ```
3.  **Restart Nginx**: `systemctl restart nginx`
4.  **Setup SSL**:
    ```bash
    apt install -y certbot python3-certbot-nginx
    certbot --nginx -d api.yourdomain.com
    ```

---

## 🔗 Step 5: Update Google Sheets

1.  Get your new public URL (e.g., `https://api.yourdomain.com`).
2.  Open your Google Sheet.
3.  Go to **Extensions > Apps Script**.
4.  Update the `API_URL` variable at the top of the script:
    ```javascript
    const API_URL = "https://api.yourdomain.com/api";
    ```
5.  Save and verify the sync works by editing a cell!

---

## 🛡️ Maintenance

-   **View Logs**: `docker compose logs -f --tail=100`
-   **Restart App**: `docker compose restart backend`
-   **Update Code**:
    ```bash
    git pull
    docker compose up -d --build
    ```
