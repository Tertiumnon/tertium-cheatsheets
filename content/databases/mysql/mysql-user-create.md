# MySQL User Creation

## Create a New User

```sql
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
```

- Replace `username` with the desired username.
- Replace `localhost` with the host (use `%` for any host).
- Replace `password` with a strong password.

## Grant Privileges

Grant all privileges on a specific database:

```sql
GRANT ALL PRIVILEGES ON database_name.* TO 'username'@'localhost';
```

Grant specific privileges:

```sql
GRANT SELECT, INSERT, UPDATE ON database_name.* TO 'username'@'localhost';
```

## Apply Privileges

```sql
FLUSH PRIVILEGES;
```

## Change User Password

```sql
ALTER USER 'username'@'localhost' IDENTIFIED BY 'new_password';
```

## Remove User

```sql
DROP USER 'username'@'localhost';
```
