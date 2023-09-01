# 基本觀念及環境資訊 #
## PostgreSQL ##
- PostgreSQL基本上Linux的資料庫, 可以使用Container安裝使用
  - 取用bitnami/postgresql的image, 已包含安全性修正

## pgAdmin4 ##
- pgAdmin4是針對PostgreSQL最常見的管理工具, web界面, 也可使用container安裝
  - 取用dpage/pgadmin4的image

## docker-compose ##
- 可採用以下docker-compose集中啟用2個container
> [docker-compose.yml](docker-compose.yml)

