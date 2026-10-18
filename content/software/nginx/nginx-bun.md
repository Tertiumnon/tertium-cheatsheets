# Nginx Configuration for Bun Applications

This document provides a basic Nginx configuration to serve an application built with [Bun](https://bun.sh/) as a reverse proxy.

## 1. Bun Application Setup

First, ensure your Bun application is running and listening on a specific port. Here's a simple example of a Bun HTTP server:

**`server.bun.js` (or `server.ts`)**

```typescript
// filepath: server.bun.js
// To run: bun run server.bun.js
const server = Bun.serve({
  port: 3000, // Your Bun app will listen on this port
  fetch(request) {
    return new Response("Welcome to Bun!");
  },
});

console.log(`Listening on http://localhost:${server.port} ...`);
```

You would typically run this with `bun run server.bun.js`. Make sure your application is listening on a port (e.g., `3000`, `8080`).

For production, you might use a process manager like `pm2` or `systemd` to keep your Bun application running.

**Example using `pm2`:**

```bash
# Install pm2 globally if you haven't already
npm install pm2 -g # or bun install -g pm2

# Start your Bun application with pm2
pm2 start "bun run server.bun.js" --name my-bun-app

# To make pm2 start on system boot
pm2 startup
pm2 save
```

## 2. Nginx Server Block Configuration

Nginx will act as a reverse proxy, forwarding requests from clients (e.g., on port 80 or 443) to your Bun application running on its specific port (e.g., 3000).

Create a new Nginx site configuration file, for example, `/etc/nginx/sites-available/{your_bun_app_conf}`:

```nginx
// filepath: /etc/nginx/sites-available/{your_bun_app_conf}
server {
    listen 80;
    listen [::]:80; # For IPv6

    server_name {your_domain.com} www.{your_domain.com}; # Or your server's IP address

    # Optional: Logging
    access_log /var/log/nginx/{your_bun_app_conf}.access.log;
    error_log /var/log/nginx/{your_bun_app_conf}.error.log;

    location / {
        proxy_pass http://localhost:3000; # Forward requests to your Bun app
        # Ensure '3000' matches the port your Bun app is listening on.
        # If Bun is on a different machine or in a container, use its IP:port.

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support (if your Bun app uses WebSockets)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Optional: Increase proxy buffer sizes if needed for large requests/responses
        # proxy_buffers 16 16k;
        # proxy_buffer_size 32k;

        # Optional: Adjust timeouts
        # proxy_connect_timeout 60s;
        # proxy_send_timeout 60s;
        # proxy_read_timeout 60s;
    }

    # Optional: Serve static assets directly with Nginx for better performance
    # if your Bun app doesn't handle them or you want to offload it.
    # location /static/ {
    #     alias /path/to/your/bun_app/public/static/;
    #     expires 7d; # Cache static assets for 7 days
    #     access_log off; # Optional: disable access logging for static assets
    # }

    # Optional: Deny access to hidden files
    location ~ /\. {
        deny all;
    }
}

# For HTTPS (Recommended for production):
# server {
#     listen 443 ssl http2;
#     listen [::]:443 ssl http2; # For IPv6
#
#     server_name {your_domain.com} www.{your_domain.com};
#
#     # SSL Certificate paths (e.g., from Let's Encrypt)
#     ssl_certificate /etc/letsencrypt/live/{your_domain.com}/fullchain.pem;
#     ssl_certificate_key /etc/letsencrypt/live/{your_domain.com}/privkey.pem;
#     include /etc/letsencrypt/options-ssl-nginx.conf; # Recommended SSL settings
#     ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;   # Recommended DH params
#
#     # Optional: Logging
#     access_log /var/log/nginx/{your_bun_app_conf}.ssl.access.log;
#     error_log /var/log/nginx/{your_bun_app_conf}.ssl.error.log;
#
#     location / {
#         proxy_pass http://localhost:3000; # Forward to Bun app
#         proxy_set_header Host $host;
#         proxy_set_header X-Real-IP $remote_addr;
#         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#         proxy_set_header X-Forwarded-Proto $scheme;
#
#         # WebSocket support
#         proxy_http_version 1.1;
#         proxy_set_header Upgrade $http_upgrade;
#         proxy_set_header Connection "upgrade";
#     }
# }
```

*Replace placeholders like `{your_domain.com}`, `{your_bun_app_conf}`, and the Bun application port (`3000`) with your actual values.*

## 3. Enable the Nginx Site

Create a symbolic link from `sites-available` to `sites-enabled`:

```bash
sudo ln -s /etc/nginx/sites-available/{your_bun_app_conf} /etc/nginx/sites-enabled/
```

*If you have a default site configuration enabled (`/etc/nginx/sites-enabled/default`) and it conflicts, you might want to remove its symlink.*

## 4. Test and Reload Nginx

Always test your Nginx configuration before applying changes:

```bash
sudo nginx -t
```

If the test is successful, reload Nginx:

```bash
sudo systemctl reload nginx
```

## Key Considerations

- **Port:** Ensure the `proxy_pass` directive in Nginx points to the correct host and port where your Bun application is listening.
- **HTTPS:** For production, always configure HTTPS. You can use Let's Encrypt for free SSL certificates.
- **Static Files:** For optimal performance, consider serving static files (CSS, JS, images) directly from Nginx rather than through your Bun application if possible.
- **Process Management:** Use a process manager like `pm2` or `systemd` to ensure your Bun application restarts if it crashes and runs on system startup.
- **Logging:** Check Nginx access and error logs (`/var/log/nginx/`) and your Bun application's logs for troubleshooting.
- **Firewall:** Ensure your firewall (e.g., `ufw`) allows traffic on ports 80 (HTTP) and 443 (HTTPS). The port your Bun app listens on (e.g., 3000) typically only needs to be accessible from `localhost` if Nginx is on the same server.

```bash
# Example for UFW
sudo ufw allow 'Nginx Full' # Allows HTTP and HTTPS
# If Bun is on a different server and Nginx needs to access it over the network:
# sudo ufw allow from {nginx_server_ip} to any port {bun_app_port}
```
