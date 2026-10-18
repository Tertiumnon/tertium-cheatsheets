# PostgreSQL

## Installation

## Access

### Check database status

- Check database status (Ubuntu).

```bash
sudo service postgresql status
```

### Switch over to the postgres account

- Switch over to the postgres account.

```bash
sudo -i -u postgres
psql
```

### Accessing a Postgres Prompt Without Switching Accounts

```bash
sudo -u postgres psql
```

### Exit Postgres Account

```sql
\q
```

## Usage

### Help

- Switch to the postgres account.

```bash
sudo -u postgres psql
```

- List postgres commands.

```sql
\?
```

### Create database

- Create database.

```bash
sudo -i -u postgres
createdb {database_name}
```

- Create database (psql).

```sql
CREATE DATABASE {database};
```

### Create user

- Create user.

```bash
sudo -i -u postgres
createuser --interactive
```

### Change user password

- Switch to Account

```bash
sudo -u postgres psql
```

- Change user password

```sql
ALTER USER {user} WITH PASSWORD '{password}';
```

### Grant privileges to database

```sql
grant all privileges on database {database} to {user};
```

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/12/reference.html)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/11/reference.html)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/10/reference.html)
