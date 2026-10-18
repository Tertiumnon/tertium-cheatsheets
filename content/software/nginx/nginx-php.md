# Nginx PHP-FPM Configuration Cheatsheet

This document covers configuring Nginx to work with PHP-FPM (FastCGI Process Manager) to serve PHP applications.

## 1. PHP-FPM Pool Configuration

Nginx serves PHP applications by passing requests to a PHP-FPM service. You need to configure a PHP-FPM pool to listen for these requests.

### a. Locate and Edit PHP-FPM Pool Configuration

The main PHP-FPM configuration file is usually `php-fpm.conf` (e.g., `/etc/php/{php_version}/fpm/php-fpm.conf`), and pool configurations are typically in a subdirectory like `pool.d/` (e.g., `/etc/php/{php_version}/fpm/pool.d/www.conf`).

- **Debian/Ubuntu:** `/etc/php/{php_version}/fpm/pool.d/www.conf` (e.g., for PHP 8.2, `/etc/php/8.2/fpm/pool.d/www.conf`)
- **Fedora/CentOS/RHEL:** `/etc/php-fpm.d/www.conf`
- **Arch Linux:** `/etc/php/php-fpm.d/www.conf` (after installing `php-fpm`)

*Replace `{php_version}` with your installed PHP version (e.g., 8.2, 8.3).*

### b. Configure PHP-FPM Listener

Ensure PHP-FPM is listening on either a TCP socket or a UNIX socket.

**Example for TCP socket (e.g., in `www.conf`):**

```ini
; Ensure this line is active and others like 'listen = /run/php/php{php_version}-fpm.sock' are commented out
listen = 127.0.0.1:9000
```

**Example for UNIX socket (e.g., in `www.conf`):**

```ini
; Ensure this line is active and others like 'listen = 127.0.0.1:9000' are commented out
listen = /run/php/php{php_version}-fpm.sock
; Common socket paths:
; Debian/Ubuntu: /run/php/php{php_version}-fpm.sock (e.g., /run/php/php8.2-fpm.sock)
; Fedora/RHEL: /run/php-fpm/www.sock
; Arch Linux: /run/php-fpm/php-fpm.sock

; For UNIX sockets, ensure Nginx's user (e.g., www-data, nginx, http) has permission to access the socket.
; Adjust listen.owner, listen.group, listen.mode if necessary:
; listen.owner = www-data
; listen.group = www-data
; listen.mode = 0660
```

*The Nginx user (`www-data`, `nginx`, `http`) varies by distribution.*

### c. Restart PHP-FPM

After making changes to the pool configuration, restart the PHP-FPM service. The service name can vary:

```bash
# For Debian/Ubuntu (replace {php_version} e.g., php8.2-fpm)
sudo systemctl restart php{php_version}-fpm

# For Fedora/RHEL/Arch Linux
sudo systemctl restart php-fpm
```

*You can check your system's specific service name using `systemctl list-units | grep fpm` or `ps aux | grep php-fpm`.*

## 2. Nginx Server Block Configuration for PHP

In your Nginx site configuration file (e.g., `/etc/nginx/sites-available/{your_site_conf_name}`), you need a `location` block to handle `.php` files and pass them to PHP-FPM.

```nginx
server {
    # ... other server directives like listen, server_name, root, index ...

    index index.php index.html index.htm; # Ensure index.php is listed

    location / {
        try_files $uri $uri/ /index.php?$query_string; # Common for frameworks like WordPress, Laravel
        # For basic sites, you might just use:
        # try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        # Option 1: Use a common snippet (often available on Debian/Ubuntu)
        # This snippet typically includes fastcgi_params and sets SCRIPT_FILENAME.
        # include snippets/fastcgi-php.conf;

        # Option 2: Manual configuration (if snippet is not available or for customization)
        include fastcgi_params; # Provides basic FastCGI parameters
        
        # Set SCRIPT_FILENAME. $document_root is the 'root' directive in your server block.
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        # fastcgi_param SCRIPT_NAME $fastcgi_script_name; # May be needed by some applications

        # Prevent Nginx from sending requests for non-existent PHP files to PHP-FPM
        # try_files $uri =404; # Uncomment if you don't use a front controller pattern like in the location / block

        # Pass PHP scripts to PHP-FPM (choose one and match your PHP-FPM 'listen' directive)
        
        # For UNIX socket (recommended for performance on a single server):
        # Ensure the path matches your PHP-FPM configuration.
        fastcgi_pass unix:/run/php/php{php_version}-fpm.sock; # e.g., unix:/run/php/php8.2-fpm.sock
        # Or a more generic path if applicable:
        # fastcgi_pass unix:/run/php-fpm/www.sock; 

        # For TCP socket:
        # fastcgi_pass 127.0.0.1:9000;

        # Mitigate CVE-2019-11043 vulnerability if not using try_files $uri =404;
        # fastcgi_split_path_info ^(.+\.php)(/.+)$;
        # if (!-f $document_root$fastcgi_script_name) {
        #    return 404;
        # }
    }

    # ... other location blocks or directives ...
}
```

*Remember to replace `{php_version}` and socket paths/ports to match your actual PHP-FPM setup.*
*After modifying Nginx configuration, always test (`sudo nginx -t`) and then reload (`sudo systemctl reload nginx`).*

## 3. Security Considerations

- **Permissions:** Ensure the Nginx worker process user (e.g., `www-data`, `nginx`) has appropriate read permissions for your web root and execute permissions for directories if needed, but restrict write access as much as possible.
- **Socket Permissions:** If using UNIX sockets, ensure the Nginx user can read/write to the socket. `listen.owner`, `listen.group`, and `listen.mode` in the PHP-FPM pool configuration are crucial.
- **`fastcgi_param SCRIPT_FILENAME`:** Ensure this is correctly set. `$document_root$fastcgi_script_name` is standard.
- **Restrict access to unnecessary PHP files:** If your application has PHP files outside the web root or in include directories that shouldn't be directly accessible via the web, ensure your Nginx configuration or application logic prevents this.
- **Keep PHP and PHP-FPM updated:** Regularly update to patch security vulnerabilities.
