# PHP

## Installation

### Install PHP on Windows with Chocolatey

```powershell
choco install php --version 7.2
```

### Install PHP on Ubuntu

```bash
#!/bin/bash
sudo apt install php7.2-common php7.2-dev
```

### Install PHP common extensions on Ubuntu

```bash
#!/bin/bash
sudo apt install php-pear php7.2-bcmath php7.2-json php7.2-mbstring php7.2-mcrypt php7.2-curl php7.2-zip php7.2-xml php7.2-xmlrpc php7.2-xsl php7.2-intl php7.2-gd php7.2-imagick php7.2-ldap
```

### Install PHP-MySQL on Ubuntu

```bash
#!/bin/bash
sudo apt install php7.2-mysql
```

### Install PHP-PostreSQL on Ubuntu

```bash
#!/bin/bash
sudo apt install php7.2-pgsql
```

### Install PHP-Firebird on Ubuntu

```bash
sudo apt install php7.2-interbase
```

## Usage

### Basic commands

#### Get PHP version

```bash
php --version
```

#### Get PHP info

```bash
php -i
```

#### Get php.ini location

```bash
php -i | grep php.ini
```

#### Run interactive shell

```bash
php -a
```

### 


## Debug

### xDebug

#### Remote debugging of cli-script

```bash
#!/bin/bash
php -dxdebug.remote_enable=1 -dxdebug.remote_mode=req -dxdebug.remote_port=9000 -dxdebug.remote_host=127.0.0.1 -dxdebug.remote_autostart=1 /path/to/script
```

## Package managers

### Composer

#### Installation

`https://getcomposer.org/download`

## Troubleshooting

### Fatal error: Maximum execution time of 30 seconds exceeded

Try to encrease parameter "max_execution_time" in php.ini:

```text
max_execution_time = 30
```

