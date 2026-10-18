# Nginx Cheatsheet

## 1. Installation

### Debian/Ubuntu

```bash
sudo apt update
sudo apt install nginx -y
```

### Arch Linux

```bash
sudo pacman -Syu nginx
```

### Fedora

```bash
sudo dnf install nginx -y
```

After installation, you might want to enable and start the Nginx service:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

## 2. PHP Integration

For configuring Nginx with PHP-FPM, please refer to the [Nginx PHP-FPM Configuration Cheatsheet](./nginx-php.md).

## 3. Basic Nginx Server Configuration (2025)

Nginx site configurations are typically stored in `/etc/nginx/sites-available/` and enabled by creating a symbolic link in `/etc/nginx/sites-enabled/`.

### a. Create a Site Configuration File

Create a new file, e.g., `/etc/nginx/sites-available/{your_site_conf_name}` (e.g., `mywebsite`):

```nginx
// filepath: /etc/nginx/sites-available/{your_site_conf_name}
server {
    listen 80;
    listen [::]:80; # For IPv6

    server_name {your_domain.com} www.{your_domain.com}; # Or IP address, or _ for default catch-all

    root /var/www/{your_site_directory}; # e.g., /var/www/mywebsite
    index index.html index.htm; # Add index.php here if serving PHP, see nginx-php.md

    location / {
        try_files $uri $uri/ =404;
    }

    # For PHP configuration, see nginx-php.md
    # Example placeholder for where PHP location block would go:
    # location ~ \.php$ {
    #     include snippets/fastcgi-php.conf; # Or manual config
    #     fastcgi_pass unix:/run/php/php{php_version}-fpm.sock; # Or TCP
    # }

    # Optional: Deny access to .htaccess files, if Apache's .htaccess files might be present
    location ~ /\.ht {
        deny all;
    }

    # Optional: Improve security by denying access to hidden files
    location ~ /\. {
        deny all;
    }

    # Optional: Logging
    access_log /var/log/nginx/{your_site_conf_name}.access.log;
    error_log /var/log/nginx/{your_site_conf_name}.error.log;
}
```

*Replace placeholders like `{your_domain.com}`, `{your_site_directory}`, and `{your_site_conf_name}` with your actual values.*
*Ensure the `root` directory (e.g., `/var/www/mywebsite`) exists and the Nginx user (e.g., `www-data` or `nginx`) has read permissions.*
*If serving PHP, ensure `index.php` is added to the `index` directive and refer to `nginx-php.md` for the `location ~ \.php$` block.*

### b. Enable the Site

Create a symbolic link from `sites-available` to `sites-enabled`:

```bash
sudo ln -s /etc/nginx/sites-available/{your_site_conf_name} /etc/nginx/sites-enabled/
```

*If you have a default site configuration enabled (`/etc/nginx/sites-enabled/default`) and it conflicts (e.g., also listens on port 80 as a default server), you might want to remove its symlink:*

```bash
# sudo rm /etc/nginx/sites-enabled/default # Use with caution
```

### c. Test Nginx Configuration

Always test your Nginx configuration before applying changes:

```bash
sudo nginx -t
```

### d. Reload or Restart Nginx

If the test is successful (`syntax is ok` and `test is successful`), apply the changes by reloading Nginx:

```bash
sudo systemctl reload nginx
```

*If a reload is not sufficient or if you prefer a full restart:*

```bash
# sudo systemctl restart nginx
```

## 4. Basic Nginx Commands

- **Start Nginx:** `sudo systemctl start nginx`
- **Stop Nginx:** `sudo systemctl stop nginx`
- **Restart Nginx:** `sudo systemctl restart nginx`
- **Reload Nginx configuration (graceful):** `sudo systemctl reload nginx`
- **Check Nginx status:** `sudo systemctl status nginx`
- **Enable Nginx to start on boot:** `sudo systemctl enable nginx`
- **Disable Nginx from starting on boot:** `sudo systemctl disable nginx`
- **Check
