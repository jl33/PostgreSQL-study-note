# psql基本概念 #

## 環境資訊 ##
- PostgreSQL是Linux常見的資料庫, 可以使用Container安裝使用
- psql: 相當於SQL Server的sqlcmd指令工具
  - 基本上是在bash下執行

## 在bash下執行 ##
- 列出資料庫清單。在以上container中, 因為權限因素需要有指定帳號才能做查詢
``` bash
$ psql -U pguser -l
Password for user pguser: *****
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 mydb      | pguser   | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/pguser           +
           |          |          |             |             |            |                 | pguser=CTc/pguser
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(4 rows)
```
- 沒有指定帳號時會有錯誤。此container要求*預設*要給ID 1001有存取權
``` bash
$ psql -l
psql: error: local user with ID 1001 does not exist
```
- 建立DB, 建立user, 刪除DB, 刪除user等等都需要給帳號才能在container的bash中執行
``` bash
$ createdb mydb2 -U postgres
Password: *****
$ psql -l -U postgres
Password for user postgres:
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 mydb      | pguser   | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/pguser           +
           |          |          |             |             |            |                 | pguser=CTc/pguser
 mydb2     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(5 rows)
$ dropdb mydb2 -U postgres
Password: *****
```
## 在psql下執行 ##
- `postgres`為預設PostgreSQL超級使用者, 只有它可以不指定資料庫做登入; 其他使用者都必須在登入時指定資料庫, 如下的`pguser`為一般帳號, 未指定連線資料庫登入會有錯誤
  - 成功連入後, 前面有提示詞為登入帳號之 *{預設資料庫}=#*
``` bash
$ psql -U pguser
Password for user pguser: 
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  database "pguser" does not exist
$ psql -U postgres
Password for user postgres: 
psql (15.4)
Type "help" for help.

postgres=#
```
- `\?`查psql指令說明
- `\h`查SQL指令說明
- `\H`切換查詢結果是html格式或表格格式顯示
- 常用查詢指令
``` bash
# \l 列出資料庫
mydb=> \l+
                                                                                   List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges   |  Size   | Tablespace |                Description
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------+---------+------------+--------------------------------------------
 mydb      | pguser   | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/pguser           +| 7461 kB | pg_default |
           |          |          |             |             |            |                 | pguser=CTc/pguser     |         |            |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |                       | 7453 kB | pg_default | default administrative connection database
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +| 7297 kB | pg_default | unmodifiable empty database
           |          |          |             |             |            |                 | postgres=CTc/postgres |         |            |
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +| 7369 kB | pg_default | default template for new databases
           |          |          |             |             |            |                 | postgres=CTc/postgres |         |            |
(4 rows)

#\d是查table,view等物件; +是增加顯示額外欄位; S是顯示系統物件
#\d,\dp,\z 是相似功能的指令
mydb=> \d+ 
                                     List of relations
 Schema |   Name    | Type  | Owner  | Persistence | Access method |  Size   | Description 
--------+-----------+-------+--------+-------------+---------------+---------+-------------
 public | demo      | table | pguser | permanent   | heap          | 0 bytes | 
 public | sometable | table | pguser | permanent   | heap          | 0 bytes | 
(2 rows)

mydb=> \d
          List of relations
 Schema |   Name    | Type  | Owner  
--------+-----------+-------+--------
 public | demo      | table | pguser
 public | sometable | table | pguser
(2 rows)

mydb=> \dS+
                                                    List of relations
   Schema   |              Name               | Type  |  Owner   | Persistence | Access method |    Size    | Description 
------------+---------------------------------+-------+----------+-------------+---------------+------------+-------------
 pg_catalog | pg_aggregate                    | table | postgres | permanent   | heap          | 56 kB      | 
 pg_catalog | pg_am                           | table | postgres | permanent   | heap          | 40 kB      | 
 pg_catalog | pg_amop                         | table | postgres | permanent   | heap          | 88 kB      |
 pg_catalog | pg_amproc                       | table | postgres | permanent   | heap          | 72 kB      |
 pg_catalog | pg_attrdef                      | table | postgres | permanent   | heap          | 8192 bytes |
 ...

mydb=> \dp
                               Access privileges
 Schema |   Name    | Type  | Access privileges | Column privileges | Policies
--------+-----------+-------+-------------------+-------------------+----------
 public | demo      | table |                   |                   |
 public | sometable | table |                   |                   |
(2 rows)

mydb=> \z
                               Access privileges
 Schema |   Name    | Type  | Access privileges | Column privileges | Policies
--------+-----------+-------+-------------------+-------------------+----------
 public | demo      | table |                   |                   |
 public | sometable | table |                   |                   |
(2 rows)

# 查詢role(PostgreSQL中role與user有很多相同意義)
# \dg, \du 是相同的指令
mydb=> \dg+
                                          List of roles
 Role name |                         Attributes                         | Member of | Description
-----------+------------------------------------------------------------+-----------+-------------
 pguser    | Create DB                                                  | {}        |
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}        |

```
- 在psql中使用變數
``` sql
mydb=> \set x 2 --使用\set在psql CLI設定變數x, 值為2
mydb=> select 4*:x; --使用:variable使用變數
 ?column?
----------
        8
(1 row)
```
- 預設在psql下null是呈現空白, 這會和真的空白一樣。可以用以下設定將null標示出來, 可以和空白區分
```sql
mydb=> select null as s;
 s 
---
 
(1 row)

mydb=> \pset null '<null>'
Null display is "<null>".
mydb=> select null as s;
   s
--------
 <null>
(1 row)
```
### 建立 pagila 範例資料庫 ###
- https://github.com/devrimgunduz/pagila
- 建立基本步驟
```bash
# 進到psql
mydb=> create database pagila;
CREATE DATABASE
mydb=> \q

# 跳出到bash, 並移到sql檔所在目錄
# 建立schema
$ cat pagila-schema.sql | psql -U pguser -d pagila
Password for user pguser:

# 建立data
$ cat pagila-data.sql | psql -U pguser -d pagila
Password for user pguser:
```