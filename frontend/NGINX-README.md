# Nginx Configuration Guide

A comprehensive guide to nginx configuration for serving a React frontend with an Express backend in a VM/Docker environment.

---

## Table of Contents

- [Basic HTTP Configuration](#basic-http-configuration)
- [Understanding `server_name`](#understanding-server_name)
- [Understanding `proxy_pass`](#understanding-proxy_pass)
- [server_name vs proxy_pass (VM Setup)](#server_name-vs-proxy_pass-vm-setup)
- [Building and Deploying React to Nginx](#building-and-deploying-react-to-nginx)
- [Proxy Headers Explained](#proxy-headers-explained)
- [Verifying Proxy Headers in Express](#verifying-proxy-headers-in-express)
- [HTTPS with Self-Signed SSL](#https-with-self-signed-ssl)
- [HTTPS Nginx Configuration](#https-nginx-configuration)
- [Load Balancing](#load-balancing)

---

## Basic HTTP Configuration

A minimal nginx config that serves a React SPA and proxies API requests to an Express backend:

```nginx
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server {
        listen 80;
        server_name localhost;

        # Serve React static files from the build directory
        root /usr/share/nginx/html;
        index index.html;

        # API endpoints - proxy to backend
        location /api {
            proxy_pass http://localhost:5001;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # React SPA - serve static files and handle client-side routing
        location / {
            try_files $uri $uri/ /index.html;
        }
    }
}
```

### What each block does:

| Block | Purpose |
|---|---|
| `events` | Controls how nginx handles connections (1024 simultaneous connections) |
| `http` | Main HTTP configuration block |
| `server` | A virtual server definition |
| `listen 80` | Listen on port 80 (HTTP) |
| `server_name` | Match requests based on the `Host` header |
| `root` | Directory where static files are served from |
| `index` | Default file to serve |
| `location /api` | Reverse proxy — forwards API requests to the Express backend |
| `location /` | Serves static files; falls back to `index.html` for React SPA client-side routing |

---

## Understanding `server_name`

`server_name` tells nginx **which requests to handle** by matching the `Host` header sent by the browser.

### Valid values for `server_name`:

| Value | Example | When it matches |
|---|---|---|
| **Exact hostname** | `server_name example.com;` | `Host: example.com` |
| **IP address** | `server_name 172.16.115.128;` | `Host: 172.16.115.128` |
| **`localhost`** | `server_name localhost;` | `Host: localhost` |
| **Wildcard** | `server_name *.example.com;` | `Host: api.example.com`, `Host: www.example.com` |
| **Regex** | `server_name ~^(?<sub>.+)\.example\.com$;` | Any subdomain |
| **`_` (underscore)** | `server_name _;` | **Catch-all** — matches anything (default/fallback) |
| **Multiple names** | `server_name example.com www.example.com;` | Either one |

### How matching works:

When a request arrives, nginx looks at the `Host` header and matches it against `server_name` values:

```
Browser request: GET / HTTP/1.1
                 Host: 172.16.115.128    ← nginx matches this against server_name
```

### What if nothing matches?

If no `server_name` matches the `Host` header, nginx uses the **first server block** as the default (or whichever has `default_server`):

```nginx
server {
    listen 80 default_server;    # ← this becomes the fallback
    server_name _;               # ← catch-all convention
}
```

> **Note**: If you only have ONE `server` block, nginx uses it as the default regardless of `server_name`, so even a "wrong" value would still work — but it's best practice to set it correctly.

---

## Understanding `proxy_pass`

`proxy_pass` tells nginx **where to forward requests** internally. This is the address nginx uses to connect to your backend service.

```nginx
location /api {
    proxy_pass http://localhost:5001;
}
```

When a request hits `/api/users`, nginx forwards it to `http://localhost:5001/api/users`.

---

## server_name vs proxy_pass (VM Setup)

When both nginx and your backend run on the **same VM**, and you access the app from an external browser:

```
Browser (your laptop)                          VM (172.16.115.128)
┌─────────────┐     vm-ip:80          ┌──────────────────────────────┐
│             │ ──────────────────→    │  nginx (port 80)             │
│  Browser    │                       │    │                          │
│             │ ←──────────────────    │    │ proxy_pass               │
└─────────────┘                       │    ↓ localhost:5001           │
                                      │  Express backend (port 5001) │
                                      └──────────────────────────────┘
```

### Rule of thumb:

| Directive | Should be | Why |
|---|---|---|
| **`server_name`** | **VM IP** (e.g. `172.16.115.128`) or `_` | This is what the **browser sends** as the `Host` header |
| **`proxy_pass`** | **`localhost`** (e.g. `http://localhost:5001`) | Nginx talks to the backend **internally** via loopback — faster, no firewall issues |

### Why `localhost` for proxy_pass?

- `localhost:5001` → stays inside the VM via the loopback interface (127.0.0.1) — fast, private, no firewall needed
- `vm-ip:5001` → would also work, but unnecessarily goes out through the network interface and back in

### Example for a VM with IP `172.16.115.128`:

```nginx
server {
    listen 80;
    server_name 172.16.115.128;          # ← what the browser sends as Host

    location /api {
        proxy_pass http://localhost:5001;  # ← internal connection to backend
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## Building and Deploying React to Nginx

Build the React app and copy the static files to nginx's serving directory:

```bash
# Build React production bundle
npm run build

# Copy build output to nginx html directory
cp -r build/* /usr/share/nginx/html/
```

---

## Proxy Headers Explained

When nginx proxies a request to the backend, it can forward important client information via headers:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

| Header | Value | Purpose |
|---|---|---|
| `Host` | Original `Host` header from client | Backend knows the original hostname |
| `X-Real-IP` | Client's actual IP address | Backend knows who the real client is |
| `X-Forwarded-For` | Chain of proxy IPs | Tracks the full proxy chain |
| `X-Forwarded-Proto` | `http` or `https` | Backend knows if the original request was secure |

---

## Verifying Proxy Headers in Express

Add this middleware to your Express backend to log the proxy headers:

```javascript
// In your src/server.js or wherever you configure Express
app.use((req, res, next) => {
  console.log('📨 Request Headers:');
  console.log('   Host:', req.headers.host);
  console.log('   X-Real-IP:', req.headers['x-real-ip']);
  console.log('   X-Forwarded-For:', req.headers['x-forwarded-for']);
  console.log('   X-Forwarded-Proto:', req.headers['x-forwarded-proto']);
  console.log('   User-Agent:', req.headers['user-agent']);
  next();
});
```

---

## HTTPS with Self-Signed SSL

### Step 1: Generate self-signed certificates

```bash
# Create directory for certificates
sudo mkdir -p /etc/nginx/ssl

# Generate self-signed certificate (valid for 365 days)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx-selfsigned.key \
  -out /etc/nginx/ssl/nginx-selfsigned.crt \
  -subj "/C=NP/ST=Province/L=City/O=MyOrg/CN=localhost"

# Set proper permissions
sudo chmod 600 /etc/nginx/ssl/nginx-selfsigned.key
sudo chmod 644 /etc/nginx/ssl/nginx-selfsigned.crt
```

### Step 2: (Optional) Generate Diffie-Hellman parameters for stronger security

```bash
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```

---

## HTTPS Nginx Configuration

Full config with HTTP→HTTPS redirect, SSL, security headers, and reverse proxy:

```nginx
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Redirect HTTP to HTTPS
    server {
        listen 80;
        server_name 172.16.115.128;

        location / {
            return 301 https://$host$request_uri;
        }
    }

    # HTTPS server
    server {
        listen 443 ssl http2;
        server_name 172.16.115.128;

        # SSL certificate files
        ssl_certificate /etc/nginx/ssl/nginx-selfsigned.crt;
        ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;
        ssl_dhparam /etc/nginx/ssl/dhparam.pem;

        # SSL configuration
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA384;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        ssl_session_tickets off;

        # Security headers
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Root directory for React app
        root /usr/share/nginx/html;
        index index.html;

        # API endpoints - proxy to backend
        location /api {
            proxy_pass http://localhost:5001;
            proxy_http_version 1.1;

            # Proxy headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;
        }

        # React SPA - handle client-side routing
        location / {
            try_files $uri $uri/ /index.html;
        }
    }
}
```

### What each SSL directive does:

| Directive | Purpose |
|---|---|
| `ssl_certificate` | Path to the public SSL certificate |
| `ssl_certificate_key` | Path to the private key |
| `ssl_dhparam` | Diffie-Hellman parameters for key exchange |
| `ssl_protocols` | Only allow TLS 1.2 and 1.3 (disable older insecure versions) |
| `ssl_ciphers` | Specify which encryption ciphers to use |
| `ssl_prefer_server_ciphers` | Server chooses the cipher (more secure) |
| `ssl_session_cache` | Cache SSL sessions for better performance |
| `ssl_session_tickets` | Disable session tickets (more secure) |

### Security headers:

| Header | Purpose |
|---|---|
| `Strict-Transport-Security` | Forces browsers to always use HTTPS |
| `X-Frame-Options` | Prevents clickjacking by blocking iframes |
| `X-Content-Type-Options` | Prevents MIME type sniffing |
| `X-XSS-Protection` | Enables browser's XSS filter |

---

## Load Balancing

### Step 1: Run multiple backend instances

```bash
# Terminal 1 - Backend instance 1
PORT=5001 npm run dev

# Terminal 2 - Backend instance 2
PORT=5002 npm run dev
```

### Step 2: Configure upstream (Round Robin — default)

```nginx
# Define upstream backend servers (load balancing pool)
upstream backend_servers {
    server localhost:5001;
    server localhost:5002;
}
```

### Step 3: Point proxy_pass to the upstream

```nginx
location /api {
    proxy_pass http://backend_servers;
}
```

### Weighted Load Balancing

Send more traffic to specific servers:

```nginx
# Weighted load balancing
upstream backend_servers {
    server localhost:5001 weight=3;  # Gets 75% of traffic (3 out of 4 requests)
    server localhost:5002 weight=1;  # Gets 25% of traffic (1 out of 4 requests)
}
```

### Custom Logging for Load Balancing

Track which backend handled each request:

```nginx
# Custom log format to see which backend handled the request
log_format upstreamlog '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $body_bytes_sent '
                       '"$http_referer" "$http_user_agent" '
                       'upstream: $upstream_addr '
                       'response_time: $upstream_response_time';

# Access log with custom format
access_log /var/log/nginx/access.log upstreamlog;
error_log /var/log/nginx/error.log warn;
```

### Other Load Balancing Strategies:

```nginx
# Least connections - sends to the server with fewest active connections
upstream backend_servers {
    least_conn;
    server localhost:5001;
    server localhost:5002;
}

# IP hash - same client always goes to the same server (sticky sessions)
upstream backend_servers {
    ip_hash;
    server localhost:5001;
    server localhost:5002;
}
```
