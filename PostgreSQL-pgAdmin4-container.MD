# 基本觀念及環境資訊 #
## PostgreSQL ##
- PostgreSQL基本上Linux的資料庫, 可以使用Container安裝使用
  - 取用bitnami/postgresql的image, 已包含安全性修正

## pgAdmin4 ##
- pgAdmin4是針對PostgreSQL最常見的管理工具, web界面, 也可使用container安裝
  - 取用dpage/pgadmin4的image
- 此container的工作目錄在`/var/lib/pgadmin`,若這裡使用對應到`driver: local`類型的Volumes, 會發現在管理界面中要存取外部檔案(.sql;.csv)時都會是`Access denied`, 故此處建議應對應到host的明確目錄, 以便在建立container時有存取權限
- 此container在啟用後的工作目錄看到的會是對應到`/var/lib/pgadmin/your_email`的user目錄

## docker-compose ##
- 可採用以下docker-compose集中啟用2個container
> [docker-compose.yml](docker-compose.yml)

