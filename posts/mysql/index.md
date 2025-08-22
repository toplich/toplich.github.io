# MySQL Basics Cheat Sheet


# MySQL Basics Cheat Sheet

This article contains essential commands for working with MySQL: creating users, databases, tables, managing access, and viewing information.

---

## 1. Connect to MySQL

```bash
mysql -u root -p
```

Or with host specified:
```bash
mysql -u username -p -h server_ip
```

---

## 2. Database Management

Create a database:
```sql
CREATE DATABASE mydb;
```

Show all databases:
```sql
SHOW DATABASES;
```

Select a database:
```sql
USE mydb;
```

Drop a database:
```sql
DROP DATABASE mydb;
```

---

## 3. Table Management

Create a table:
```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50),
  email VARCHAR(100)
);
```

Show tables:
```sql
SHOW TABLES;
```

Describe table structure:
```sql
DESCRIBE users;
```

Drop a table:
```sql
DROP TABLE users;
```

---

## 4. Basic Queries

Insert data:
```sql
INSERT INTO users (name, email) VALUES ('John Doe', 'john@example.com');
```

Select data:
```sql
SELECT * FROM users;
```

Update data:
```sql
UPDATE users SET email = 'new@example.com' WHERE id = 1;
```

Delete data:
```sql
DELETE FROM users WHERE id = 1;
```

---

## 5. Users and Permissions

Create a user:
```sql
CREATE USER 'username'@'%' IDENTIFIED BY 'password';
```

Grant all privileges on a database:
```sql
GRANT ALL PRIVILEGES ON mydb.* TO 'username'@'%';
```

Grant read-only access to a specific table:
```sql
GRANT SELECT ON mydb.mytable TO 'username'@'%';
```

Check user privileges:
```sql
SHOW GRANTS FOR 'username'@'%';
```

Revoke all privileges:
```sql
REVOKE ALL PRIVILEGES ON mydb.* FROM 'username'@'%';
```

Drop a user:
```sql
DROP USER 'username'@'%';
```

---

## 6. Administration

Show running processes:
```sql
SHOW PROCESSLIST;
```

Reload privileges:
```sql
FLUSH PRIVILEGES;
```

Exit MySQL:
```sql
EXIT;
```


