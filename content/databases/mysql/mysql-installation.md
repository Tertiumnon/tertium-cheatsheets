# MySQL 8 Installation

## Ubuntu / Debian

```bash
sudo apt update
sudo apt install wget lsb-release gnupg
wget https://dev.mysql.com/get/mysql-apt-config_0.8.29-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.29-1_all.deb
sudo apt update
sudo apt install mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
```

## CentOS / RHEL

```bash
sudo yum install https://dev.mysql.com/get/mysql80-community-release-el7-5.noarch.rpm
sudo yum install mysql-community-server
sudo systemctl start mysqld
sudo systemctl enable mysqld
```

## Windows

1. Download the MySQL Installer from [MySQL Downloads](https://dev.mysql.com/downloads/installer/).
2. Run the installer and follow the setup wizard.
3. Configure root password and server settings as prompted.

## macOS (Homebrew)

```bash
brew update
brew install mysql
brew services start mysql
```

## Post-Installation (All Platforms)

- Secure installation:

```bash
sudo mysql_secure_installation
```

- Login as root:

```bash
mysql -u root -p
```
