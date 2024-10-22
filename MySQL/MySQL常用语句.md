#### 创建数据库，指定字符集和排序规则
```
CREATE DATABASE database_name
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```
#### 重命名表
```
RENAME TABLE old_table_name TO new_table_name;
```
#### 复制表结构
```
CREATE TABLE new_table LIKE old_table;
```
#### 复制表结构，并插入数据
```
CREATE TABLE new_table AS SELECT * FROM old_table;
```

#### 查询所有表的大小，显示字段：Database，Table，Size in GB
```
SELECT table_schema AS "Database",
       table_name AS "Table",
       round(((data_length + index_length) / 1024 / 1024 / 1024), 2) AS "Size in GB"
FROM information_schema.TABLES
WHERE table_schema IN (SELECT schema_name FROM information_schema.schemata)
ORDER BY "Size in GB " DESC
```
#### 查询所有数据库的大小，显示字段：Database，Size in GB
```
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS "Size in GB"
FROM 
    information_schema.TABLES
GROUP BY 
    table_schema;
```
#### 新安装的MySQL5.7实例修改root密码
```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password';
```
#### 新安装的MySQL8.0实例修改root密码
```
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'new_password';
```
#### 修改用户权限
```
GRANT privileges ON database_name.* TO 'username'@'host' IDENTIFIED BY 'password';
```
