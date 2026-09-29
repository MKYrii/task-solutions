# Domain Setup and HTTPS Configuration

This guide explains how to get a free domain using DuckDNS and secure your server with an SSL certificate from Let's Encrypt using `acme.sh`.

---

## Prerequisites

Before starting, ensure you have:
* A **VPS (Virtual Private Server)** with a public (white) IP address.
* Root or `sudo` access to your server.

---

## Step 1: Register a Domain on DuckDNS

1. Go to [duckdns.org](https://duckdns.org) and log in using any available provider.
2. In the **Domains** section, add your preferred subdomain (e.g., `qwerty`). Your full domain will be `qwerty.duckdns.org`.
3. In the **current ip** field, enter your VPS public IP address and click **update ip**.

---

## Step 2: Prepare the Server

Connect to your VPS via SSH and follow these steps:

### 1. Stop web services
Stop any service currently listening on port 80 (e.g., Nginx or Apache) to allow `acme.sh` to bind to it:
```bash
sudo systemctl stop nginx
```

### 2. Install dependencies
Update your package list and install `socat` (required by acme.sh for standalone mode) and `curl` to install acme.sh:
```bash
sudo apt update && sudo apt install -y socat curl
```

Install `acme.sh` using the official script (replace `my@email.com` with your real email):
```bash
curl https://acme.sh | sh -s email=my@email.com
source ~/.bashrc
```

### 3. Create webroot directory
Create a directory for webroot challenges and set the correct permissions:
```bash
sudo mkdir -p /var/www/acme-challenge
sudo chown -R www-data:www-data /var/www/acme-challenge
```

---

## Step 3: Issue the SSL Certificate

Run the `acme.sh` command in standalone mode to request your certificate from Let's Encrypt:
```bash
~/.acme.sh/acme.sh --issue --standalone -d gazprompt.duckdns.org --server letsencrypt
```

---

## Step 4: Install Certificates to Nginx Directory

Create a dedicated directory for your SSL certificates:
```bash
sudo mkdir -p /etc/nginx/ssl
```

Install and copy the certificates to the Nginx SSL directory:
```bash
~/.acme.sh/acme.sh --install-cert -d qwerty.duckdns.org \
--key-file       /etc/nginx/ssl/qwerty.duckdns.org.key  \
--fullchain-file /etc/nginx/ssl/fullchain.cer
```

---

## Step 5: Configure Nginx

Open your Nginx configuration file (e.g., `/etc/nginx/sites-available/default` or your specific site config) and update it with the following blocks:

```nginx
# 1. Automatic redirect from HTTP to HTTPS
server {
    listen 80;
    server_name qwerty.duckdns.org;
    return 301 https://\(host\)request_uri;
}

# 2. Main HTTPS server
server {
    listen 443 ssl;
    server_name qwerty.duckdns.org;

    ssl_certificate /etc/nginx/ssl/fullchain.cer;
    ssl_certificate_key /etc/nginx/ssl/qwerty.duckdns.org.key;

    # Optimal SSL settings (Security enhancement)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        root /var/www/html;
        index index.html index.htm;
    }
}
```

### Test and restart Nginx

Test the configuration for syntax errors:
```bash
sudo nginx -t
```

If the test is successful, start Nginx back up:
```bash
sudo systemctl start nginx
```
