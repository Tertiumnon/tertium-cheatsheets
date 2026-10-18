# Firebird

## Installation

### Install on Ubuntu

- Install.

```bash
sudo apt install firebird3.0-server
```

- Check.

```bash
sudo service firebird3.0 status
```

## Usage

### Basic commands

- Enter the shell

```bash
isql-fb
```

- Create connection

```bash
SQL> connect "localhost:/var/lib/firebird/2.5/data/employee.fdb" user 'SYSDBA' password '{masterkey}';
```

- Show tables

```bash
SQL> show tables;
```

- Quit

```bash
SQL> quit;
```

## Migration

### Database migration (2.5 -> 3.0)

- Backup previous DB.

```bash
gbak -user sysdba -pas {masterkey} -b {path}/{db_name}.fdb {db_name}.fbk
```

- Restore DB from backup to a new location, for example to "/var/lib/firebird/3.0/data".

```bash
gbak -user sysdba -pas {masterkey} -c {db_name}.fbk {path}/{db_name}.fdb
```
