# Apache2

## Installation

### Install (Ubuntu)

```bash
sudo apt install apache2
```

### Install PHP mod (Ubuntu)

```bash
sudo apt install libapache2-mod-php7.2
```

## Usage

### Enable a site

- Edit /etc/sites-available/example.com.conf

```apache
<VirtualHost *:80>
  ServerName example.com
  DocumentRoot /var/www/example.com
  ErrorLog ${APACHE_LOG_DIR}/example.com-error.log
  CustomLog ${APACHE_LOG_DIR}/example.com-access.log combined
</VirtualHost>
```
