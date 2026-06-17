# Edie Scavenger Hunt — `ca.blankp.io`

A single, mobile-first, hand-drawn clue page for a scavenger hunt. The page reached via QR code
just points the player up at the browser's address bar — **the domain itself is the answer**
(`ca.blankp.io` → CAMPIO). There's a click-to-reveal "Need a hint?" line for stragglers.

It's intentionally a static site: one self-contained `index.html` (CSS + inline SVG arrow), an
SVG favicon, no build step, no backend.

## Files

- `index.html` — the whole page.
- `favicon.svg` — hand-drawn arrow that sits left of the address text, pointing right at it.

## Local preview

Just open `index.html` in a browser, or serve it and load it on your phone over LAN:

```sh
python -m http.server 8000
# then visit http://<your-computer-ip>:8000 on your phone
```

## Deploying to the DigitalOcean droplet

Host nginx + Let's Encrypt (certbot), code pulled via git. Run on a fresh Ubuntu droplet.

### 1. DNS (Porkbun)

In Porkbun's DNS panel for `blankp.io`, add an **A record**:

| Type | Host | Answer            | TTL |
|------|------|-------------------|-----|
| A    | `ca` | `<droplet IPv4>`  | 600 |

(Add an **AAAA** record with the droplet's IPv6 too, if it has one.) Leave the apex records
alone. Confirm it resolves before running certbot:

```sh
dig +short ca.blankp.io     # should print the droplet IP
```

### 2. Server prep

```sh
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx certbot python3-certbot-nginx git
```

### 3. Get the code

```sh
sudo git clone <repo-url> /var/www/ca.blankp.io
```

To redeploy later: `cd /var/www/ca.blankp.io && sudo git pull`.

### 4. nginx server block

Create `/etc/nginx/sites-available/ca.blankp.io`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name ca.blankp.io;

    root /var/www/ca.blankp.io;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable it and reload:

```sh
sudo ln -s /etc/nginx/sites-available/ca.blankp.io /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 5. HTTPS (auto-renewing)

```sh
sudo certbot --nginx -d ca.blankp.io
```

Choose the redirect-HTTP-to-HTTPS option. certbot rewrites the server block and installs a
renewal timer — verify with `systemctl list-timers | grep certbot` and
`sudo certbot renew --dry-run`.

### 6. Firewall

```sh
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

### 7. QR code

Generate a QR code encoding `https://ca.blankp.io` and place it.

## Verify it's live

```sh
curl -I http://ca.blankp.io     # 301 redirect to https://
curl -I https://ca.blankp.io    # 200 OK, valid cert
```

Then scan the QR with a real phone (ideally one iOS + one Android): the page should load fast,
look hand-drawn, and clearly point at the address bar.
