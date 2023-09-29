# 升級PostgreSQL版本 #

## 備份 ##
- 以PostgreSQL 15.4.0 Container為例
- 使用`pg_dumpall`指令。
    - Password詢問次數會依包含的DB個數為準, 包含內建DB
    - `pg_dumpall`可以備份含*Roles*, 但似乎無法包含額外設定的*Access privileges*
    - 輸出檔案其實是將DB內容轉為sql指令
```bash
$ pg_dumpall -U postgres -f /demo/dball.sql
Password:
Password:
Password:
Password:
Password:
```
## 還原 ##
- 以PostgreSQL 16為例。先`pull` image
- 單獨建立執行Container
    - 指定*Volume*, 包含剛才dump出來的檔案
    - 指定`postgres`的密碼
```bash
docker run --name postgres16 -v .\pgsVol:/demo -e POSTGRES_PASSWORD=P@ssw0rd! -d bitnami/postgresql:latest
```
- 還原dump出來的檔案
```bash
$ psql -U postgres -f /demo/dball.sql
Password for user postgres:
...
```
## docker-compose ##
- 在PostgreSQL 15.4.0時需要以下Volume, 但升到PSQL 16時已不用再加此設定, yml中就不用加了
```yml
    volumes:
      - postgresql_data:/bitnami/postgresql
```