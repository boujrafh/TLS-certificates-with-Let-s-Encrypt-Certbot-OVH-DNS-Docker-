Here’s a clean, copy-pasteable **README** you can drop in your repo to document how you generate and use Let’s Encrypt certificates with **Certbot + OVH DNS** (dockerized) and **NGINX**.

---

# TLS certificates with Let’s Encrypt (Certbot + OVH DNS, Docker)

This guide explains how to issue and renew HTTPS certificates using Certbot’s **dns-ovh** plugin in a Docker container, store them under `/opt/certbot/conf`, and mount them into your NGINX reverse proxy.

## Overview

* **ACME client**: `certbot/dns-ovh` (Docker image)
* **DNS provider**: OVH (API credentials required)
* **Cert storage (on host)**: `/opt/certbot/conf/...`
* **NGINX mounts**: `/etc/nginx/ssl/<name>-fullchain.pem` and `/etc/nginx/ssl/<name>-privkey.pem`

---

## 1) One-time setup

### 1.1 Prepare directories on the host

```bash
sudo mkdir -p /opt/certbot/{conf,work,log,secrets}
sudo chown -R root:root /opt/certbot
sudo chmod 700 /opt/certbot/secrets
```

### 1.2 Create OVH API credentials

Create `/opt/certbot/secrets/ovh.ini` with:

```ini
# OVH API credentials used by Certbot
dns_ovh_endpoint = ovh-eu
dns_ovh_application_key = <APP_KEY>
dns_ovh_application_secret = <APP_SECRET>
dns_ovh_consumer_key = <CONSUMER_KEY>
```

Permissions **must** be strict:

```bash
sudo chmod 600 /opt/certbot/secrets/ovh.ini
```

> ✅ Your domain’s nameservers must be at OVH (check: `dig NS example.com +short`).
> ✅ Your OVH API keys must have permission on the **zone** you’re issuing for.

---

## 2) Issue a certificate

### 2.1 Single domain

```bash
docker run --rm -it \
  -v /opt/certbot/conf:/etc/letsencrypt \
  -v /opt/certbot/work:/var/lib/letsencrypt \
  -v /opt/certbot/log:/var/log/letsencrypt \
  -v /opt/certbot/secrets:/secrets \
  certbot/dns-ovh certonly --dns-ovh \
  --dns-ovh-credentials /secrets/ovh.ini \
  --dns-ovh-propagation-seconds 300 \
  -m you@example.com --agree-tos --no-eff-email \
  -d n8n-rp.bh-xxxx.be
```

### 2.2 Multiple SANs (same cert)

```bash
... certbot/dns-ovh certonly --dns-ovh \
  --dns-ovh-credentials /secrets/ovh.ini \
  --dns-ovh-propagation-seconds 300 \
  -m you@example.com --agree-tos --no-eff-email \
  -d service1.example.com \
  -d service2.example.com
```

### 2.3 Wildcard (requires DNS challenge)

```bash
... certbot/dns-ovh certonly --dns-ovh \
  --dns-ovh-credentials /secrets/ovh.ini \
  --dns-ovh-propagation-seconds 300 \
  -m you@example.com --agree-tos --no-eff-email \
  -d example.com -d *.example.com
```

> ⏱ If validation fails due to slow DNS, increase `--dns-ovh-propagation-seconds` (e.g. 500).

---

## 3) Where the files live

After a successful run, Certbot places (on the **host**):

```
/opt/certbot/conf/live/<domain>/
  ├─ fullchain.pem  # certificate + chain
  ├─ privkey.pem    # private key
  ├─ cert.pem
  ├─ chain.pem
  └─ README
```

These are the files you mount into NGINX.

---

## 4) NGINX: mount and reference the certs

### 4.1 docker-compose (snippet)

```yaml
services:
  nginx:
    image: nginx:alpine
    restart: always
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro

      # n8n-rp.bh-xxxx.be
      - /opt/certbot/conf/live/n8n-rp.bh-xxxx.be/fullchain.pem:/etc/nginx/ssl/n8n-rp-fullchain.pem:ro
      - /opt/certbot/conf/live/n8n-rp.bh-xxxx.be/privkey.pem:/etc/nginx/ssl/n8n-rp-privkey.pem:ro

      # (repeat for other domains you issued)
      # - /opt/certbot/conf/live/support.bh-xxxx.be/fullchain.pem:/etc/nginx/ssl/support-fullchain.pem:ro
      # - /opt/certbot/conf/live/support.bh-xxxx.be/privkey.pem:/etc/nginx/ssl/support-privkey.pem:ro
    ports:
      - "80:80"
      - "443:443"
    networks:
      - proxy
```

> ❌ Don’t map the key onto the cert path! (That causes `PEM_read_bio_X509_AUX() failed`.)

### 4.2 nginx.conf (server block)

```nginx
server {
  listen 443 ssl;
  http2 on;
  server_name n8n-rp.bh-xxxx.be;

  ssl_certificate     /etc/nginx/ssl/n8n-rp-fullchain.pem;
  ssl_certificate_key /etc/nginx/ssl/n8n-rp-privkey.pem;

  client_max_body_size 100M;

  location / {
    proxy_pass http://n8n:5678;
    proxy_http_version 1.1;

    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_read_timeout  600s;
    proxy_send_timeout  600s;
    proxy_buffering     off;
  }
}
```

### 4.3 Apply changes

```bash
docker compose up -d nginx
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload
```

---

## 5) Renewals

Let’s Encrypt certs are valid for **90 days**. Renewal is easy.

### 5.1 Manual renewal

```bash
docker run --rm \
  -v /opt/certbot/conf:/etc/letsencrypt \
  -v /opt/certbot/work:/var/lib/letsencrypt \
  -v /opt/certbot/log:/var/log/letsencrypt \
  -v /opt/certbot/secrets:/secrets \
  certbot/dns-ovh renew --dns-ovh \
  --dns-ovh-credentials /secrets/ovh.ini \
  --dns-ovh-propagation-seconds 300
# then reload nginx to pick up renewed files:
docker compose exec nginx nginx -s reload
```

### 5.2 Test a renewal (no changes)

```bash
... certbot/dns-ovh renew --dns-ovh \
  --dns-ovh-credentials /secrets/ovh.ini \
  --dry-run
```

### 5.3 Automate with cron

Edit root’s crontab:

```bash
sudo crontab -e
```

Add (runs daily at 03:15):

```
15 3 * * * docker run --rm \
  -v /opt/certbot/conf:/etc/letsencrypt \
  -v /opt/certbot/work:/var/lib/letsencrypt \
  -v /opt/certbot/log:/var/log/letsencrypt \
  -v /opt/certbot/secrets:/secrets \
  certbot/dns-ovh renew --dns-ovh \
  --dns-ovh-credentials /secrets/ovh.ini \
  --dns-ovh-propagation-seconds 300 \
  --quiet && docker compose -f /home/n8n/docker-compose.yml exec -T nginx nginx -s reload
```

> Adjust the `-f` path to your actual compose file.

---

## 6) Troubleshooting

* **403 from OVH API**
  Your keys don’t have permission on the zone, or the zone isn’t under your OVH account. Recreate the keys or fix access.

* **`Remote end closed connection without response` during challenge**
  Usually DNS not propagated yet. Increase `--dns-ovh-propagation-seconds` (e.g., 500) and retry.

* **`PEM_read_bio_X509_AUX() failed (no start line)` in NGINX**
  You mounted the private key on the certificate path by mistake. Fix the mounts:

  ```
  fullchain.pem  -> /etc/nginx/ssl/<name>-fullchain.pem
  privkey.pem    -> /etc/nginx/ssl/<name>-privkey.pem
  ```

* **Nameservers not managed by OVH**
  Check `dig NS yourdomain.tld +short`. If not OVH, dns-ovh won’t work; use your provider’s certbot DNS plugin instead.

* **Verify cert on the host**

  ```bash
  sudo openssl x509 -noout -text -in /opt/certbot/conf/live/n8n-rp.bh-xxxx.be/fullchain.pem | head -n 20
  ```

---

## 7) Add a new domain later

Repeat the **Issue a certificate** step with the new `-d <host>` values, then:

1. Add corresponding NGINX volume mounts for `fullchain.pem` and `privkey.pem`.
2. Reference them in a `server {}` block.
3. `docker compose up -d nginx && docker compose exec nginx nginx -s reload`.

---

## 8) Safety tips

* Never commit `ovh.ini` to Git. Keep it at `600` permissions.
* Don’t share `privkey.pem`. It’s sensitive.
* Prefer **separate certs** per hostname unless you specifically need SANs (simplifies rotation and NGINX mounts).

---

**That’s it!** This doc covers issuance, storage, NGINX wiring, renewals, and common pitfalls.
