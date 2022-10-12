# Test Docs
Aafds

users表用户记录用户信息，是通用模块的基础数据表，创建的SQL语句如下：

```sql
--创建users用户信息表
CREATE TABLE IF NOT EXISTS users (
  id int8 NOT NULL PRIMARY KEY,
  iconid int8,
  name varchar(255),
  displayname varchar(255),
  password varchar(255)
);
```
