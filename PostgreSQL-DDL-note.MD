# PostgreSQL #
## Table ##
- 刪除TABLE時可判斷是否已存在
```sql
DROP TABLE IF EXISTS ...
```
- `DEFAULT`欄位值
  - `INSERT`時若沒給值, 或`INSERT`、`UPDATE`時指定`DEFAULT`會給定預設值
```SQL
CREATE TABLE mytbl(
    somecol type DEFAULT value,
    ...);
```
```sql
CREATE SEQUENCE tablename_colname_seq AS integer;
CREATE TABLE tablename (
    colname integer NOT NULL DEFAULT nextval('tablename_colname_seq')
);
ALTER SEQUENCE tablename_colname_seq OWNED BY tablename.colname;
```
- `GENERATED`欄位值, 不能直接寫入欄位值, 但可以指定`DEFAULT`為其欄位值。有2種類型, 
  - 實際儲存計算值於欄位。PostgreSQL有實做, 以`GENERATED ALWAYS AS ... STORED`表示
  - 虛擬計算欄位, 不存值, 比較像**View**。PostgreSQL沒實做
```sql
CREATE TABLE people (
    ...,
    height_cm numeric,
    height_in numeric GENERATED ALWAYS AS (height_cm / 2.54) STORED
);
```
- 預設和計算欄位比較

|   |**DEFAULT**|**GENERATED**|
|----|----|----|
|計算時機|`INSERT`時若沒指定|每次記錄異動時(含`INSERT、UPDATE`)<br>不可覆寫|
|參照欄位|不可|可|
|可用函式|可, VOLATILE函式<br>可變結果的, 如`random()`<br>(相同參數不同結果)|可, 限IMMUTABLE函式<br>不可變結果的, 如`+-*/`<br>(相同參數相同結果)|
|||僅可參照該筆記錄的欄位<br>不可跨記錄|
|||不可參照其他計算欄位|
|||不可參照系統欄位|
|||不可定義欄位預設值|
|||不可為識別欄位|
|||不可是partition key|
|||**Foreign table**可以有計算欄位|
|繼承||父欄位是計算欄位, 則其子欄位也要是計算欄位<br>父欄位是計算欄位, 則其所有父欄位必須是計算欄位

## Constraints條件約束 ##
- 可針對欄位或針對**Table**
- 可針對data type無法限制的額外再加條件約束
- 可參照其他欄位或其他記錄值, 對此欄位加以約束
- 有幾種條件約束
```sql
-- CHECK---------------------------------------------
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0),                       --column constraint
    discounted_price numeric CHECK (discounted_price > 0), --column constraint
    CHECK (price > discounted_price)                       --table constraint
);

-- NOT NULL----------------------------------------------
CREATE TABLE products (
    product_no integer NOT NULL,
    name text NOT NULL,
    price numeric NOT NULL CHECK (price > 0)
);

-- UNIQUE-----------------------------------------------
-- 1) 2筆NULL不被視為相等, 故UNIQUE不會將多筆NULL記錄視為違反約束
CREATE TABLE example (
    a integer,
    b integer,
    c integer,
    UNIQUE (a, c)
);
-- 2) 可以用NULLS NOT DISTINCT改變預設規則
CREATE TABLE products (
    product_no integer UNIQUE NULLS NOT DISTINCT,
    name text,
    price numeric
);

-- PK------------------------------------------------
-- 1) 以下2者意義相等,PRIMARY KEY = UNIQUE NOT NULL
CREATE TABLE products (
    product_no integer UNIQUE NOT NULL,
    name text,
    price numeric
);
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

-- FK------------------------------------------------
-- 1) 指定table及column名
CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products (product_no),
    quantity integer
);
-- 2) 指定table名, column為同名可忽略
CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products,
    quantity integer
);
-- 3) 指定主從column為不同名column
CREATE TABLE t1 (
  a integer PRIMARY KEY,
  b integer,
  c integer,
  FOREIGN KEY (b, c) REFERENCES other_table (c1, c2)
);
-- 4) 指定主table刪除參照記錄時, 從屬記錄處理方式。
--    還有On UPDATE、NO ACTION、SET NULL、SET DEFAUL等
CREATE TABLE order_items (
    product_no integer REFERENCES products ON DELETE RESTRICT, --從屬記錄不刪
    order_id integer REFERENCES orders ON DELETE CASCADE, --從屬記錄一起刪
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);

-- EXCLUSION------------------------------------------
-- 1) 針對table內任2筆記錄做指定的比對而做剔除
CREATE TABLE circles (
    c circle,
    EXCLUDE USING gist (c WITH &&)
);
```

## 系統欄位 ##
- 每個table內建的欄位: `tableoid、xmin、cmin、xmax、cmax、ctid`
- column命名不可與這些相同

## 變動table ##
- 增加欄位: 從**PostgreSQL 11**起, 增加欄位時若是有加入預設值, 並不會立即更新table中的所有記錄, 所以對很大的table也會很快。直到下次存取記錄時預設值才會生效, 例如`SELECT`時
  - 新指定的預設值會呈現在所有記錄的新欄位(在`SQL Server`上則不會自動在新增欄位上呈現新的預設值)
  - 但若新欄位的預設值是變動的, 如系統時間, 則必須要`UPDATE`才會呈現出來
```SQL
ALTER TABLE products ADD COLUMN description text CHECK (description <> '');
```
- 刪除欄位
```SQL
ALTER TABLE products DROP COLUMN description CASCADE; -- 若有FK關連, 可加上CASCADE做一併刪除
```
- 增加條件約束
```SQL
ALTER TABLE products ADD CHECK (name <> '');
ALTER TABLE products ADD CONSTRAINT some_name UNIQUE (product_no);
ALTER TABLE products ADD FOREIGN KEY (product_group_id) REFERENCES product_groups;
-- NOT NULL是沒Constraint名稱, 所以是用異動做新增
ALTER TABLE products ALTER COLUMN product_no SET NOT NULL;
```
- 刪除條件約束
```sql
-- 刪除要指定名稱, 必須要先查得名稱
ALTER TABLE products DROP CONSTRAINT some_name;
-- 針對NOT NULL也是要用異動做處理(再變為NULL)
ALTER TABLE products ALTER COLUMN product_no DROP NOT NULL;
```
- 改變欄位預設值
```SQL
ALTER TABLE products ALTER COLUMN price SET DEFAULT 7.77;
```
- 改變欄位型態
```SQL
-- 必須是要可以轉換的型別
ALTER TABLE products ALTER COLUMN price TYPE numeric(10,2);
```
- 改欄位名稱
```SQL
ALTER TABLE products RENAME COLUMN product_no TO product_number;
```
- 改TABLE名稱
```SQL
ALTER TABLE products RENAME TO items;
```

## Privilege 授權 ##
>https://www.postgresql.org/docs/current/ddl-priv.html
- 建立者預設就是該物件的**owner**
- 非**superuser**的一般**role**必須要是該物件的現有**owner**, 又是新**owner role**的成員之一, 才能更改物件權限
```sql
ALTER TABLE table_name OWNER TO new_owner;
```
```sql
-- 授與accounts的UPDATE權限給joe
GRANT UPDATE ON accounts TO joe;
```
- `PUBLIC`可以指包含所有**role**的意思
- `ALL`可是指包含物件的所有權限種類
```sql
-- 以下表示取消所有role在accounts的所有權限設定
REVOKE ALL ON accounts FROM PUBLIC;
```
- `WITH GRANT OPTION`可授與給**A**可授權其他人有**xxx**的權限
  - 若其後`WITH GRANT OPTION`權限被取消, 則所有**A**因此獲得的權限, 以及A再授與其他人的權限, 都將被取消
  - `WITH GRANT OPTION`不可授與給`PUBLIC`角色
- 現有**owner**可以`REVOKE`自己的權限, 例如變成唯讀不可異動, 但他還是可以再次把自己`GRANT`成有可異動權限
- 授權種類
  - `SELECT`: r
    1. 可指定所有欄位或特定欄位
    1. 同時授與`COPY TO`權限
    1. 在`UPDATE、DELETE、MERGE`的使用欄位也必須給與此權限
    1. 同時授與`currval`函式的使用
  - `INSERT`: a
    1. 可指定只可寫入特定欄位, 未指定的欄位因為不能寫在`insert`語法中, 則會以預設值寫入
    1. 同時授與`COPY FROM`權限
  - `UPDATE`: w
    1. 可指定只可更新特定欄位
    1. 更新的欄位或參照的欄位, 要一併給與`SELECT`權限
    1. `SELECT ... FOR UPDATE`及`SELECT ... FOR SHARE`則需要給`UPDATE`權限
    1. 同時授與`nextval、setval`函式的使用
  - `DELETE`: d
    1. 給與刪除記錄權限, 要一併給與`SELECT`權限
  - `TRUNCATE`: D
    1. 給與清空table權限
  - `REFERENCES`: x
    1. 給與建立**FK** index, 參照其他table或其table中的欄位的權限
  - `TRIGGER`: t
    1. 給與建立trigger的權限
  - `CREATE`: C
    1. 給與建立DB、schema等的權限
    1. revoke此權限並不會異動到現有物件
  - `CONNECT`: c
    1. 給與連線到DB的權限
    1. 預設授與給`PUBLIC`
  - `TEMPRARY`: T
    1. 給與建立暫存table的權限
    1. 預設授與給`PUBLIC`
  - `EXECUTE`: X
    1. 給與執行函式及預存的權限
    1. 僅可針對函式及預存使用
    1. 預設授與給`PUBLIC`
  - `USAGE`: U
    1. 針對預存, 指定建立預存可用的語言。預設授與給`PUBLIC`
    1. 針對`schema`, 指定在schema中可存取的物件, 同時給與允許可找尋(**look up**)的權限。revoke此權限後, **現存session**仍可能可以存取schema中的物件
    1. 針對`sequences`, 則允許使用`currval、nextval`函式
  - `SET`: s
    1. 可在當下session中設定系統參數值
  - `ALTER SYSTEM`: A
    1. 可設定系統參數值
- 以上各類型名稱後的單一字母為簡寫, 會顥示在`psql`的指令查詢結果中, 如下**Access privileges**欄位
  - Access privileges若為空白, 表示是其**owner**的**default privileges**
```bash
pagila=> \l
```
```
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 mydb      | pguser   | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/pguser           +
           |          |          |             |             |            |                 | pguser=CTc/pguser
 pagila    | pguser   | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(5 rows)
```

## Row的安全政策 ##
>https://www.postgresql.org/docs/current/ddl-rowsecurity.html
- table一般沒有安全政策限制, 因此只要有存取權限, 其內的所有記錄都是可存取的
- 以下語法可設定**row-level**安全政策
  - 設定後要符合安全政策才能存取, 否則預設的**default-deny**政策就會生效, 表示看不到記錄也無法存取。
```sql
 ALTER TABLE ... ENABLE ROW LEVEL SECURITY
```
- **owner**仍是有所有權限, **superusers**及具有`BYPASSRLS`屬性的角色, 都不受row安全政策的限制影響
```sql
-- owner可以用以下指令強制套用row安全政策限制
 ALTER TABLE ... FORCE ROW LEVEL SECURITY
```
- `TRUNCATE、REFERENCES、UNIQUE、PK、FK`也不受row安全政策影響
- 此政策可套用於**指令**或**role**
- Row安全政策會在任何條件式或函式運作之前先行運行, 以確認是否有存取權限。其判斷結果是boolean值, 若回傳false, 則不會進行其後的指定運作
- 增刪修政策及啟用停用政策指令
```sql
-- 增刪修
CREATE POLICY ...
DROP POLICY ...
ALTER POLICY ...
-- 啟用/停用
ALTER TABLE ...
```
- `POLICY`必須有名稱
  - 一個TABLE可以有多個`POLICY`, 每個的名稱都要是唯一
  - 不同TABLE可以有相同名稱的`POLICY`
- 以下指令限制只能存取屬於自己的記錄, 即看不到別人的記錄, 也無法更新別人的記錄, 甚至不能新增不屬於自己的新記錄
  - 若指定`TO`則只針對該**role**, 其他role不受限; 若要針對所有人, 則不要指定`TO`和role即可(代表針對`PUBLIC`)
  - 此處的`USING`其實隱含地同時加上`WITH CHECK`(其條件也是相同的)
```sql
CREATE TABLE accounts (manager text, company text, contact_email text);
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
-- 設定managers只能看到自己的記錄, 但沒有針對其他role設定
CREATE POLICY account_managers ON accounts TO managers
    USING (manager = current_user);
```  
- 進階設定: 所有人皆可`SELECT`到所有記錄, 但只有自己可以異動自己的記錄
  - 以下前2個`POLICY`是以`OR`(預設)寬鬆連集, 針對`SELECT`套用第1個, 若其他指令則會套用第2個
  - 在建立`POLICY`時若加上`RESTRICTIVE`則會變成`AND`限制式連集
```sql
CREATE POLICY user_sel_policy ON users
    FOR SELECT
    USING (true);
CREATE POLICY user_mod_policy ON users
    USING (user_name = current_user);

CREATE POLICY admin_local_only ON passwd AS RESTRICTIVE TO admin
    USING (pg_catalog.inet_client_addr() IS NULL);    
```
- 大型demo
```sql
-- 先使用owner或superuser建立相關物件及記錄
mydb=# CREATE TABLE passwd (
mydb(#   user_name             text UNIQUE NOT NULL,
mydb(#   pwhash                text,
mydb(#   uid                   int  PRIMARY KEY,
mydb(#   gid                   int  NOT NULL,
mydb(#   real_name             text NOT NULL,
mydb(#   home_phone            text,
mydb(#   extra_info            text,
mydb(#   home_dir              text NOT NULL,
mydb(#   shell                 text NOT NULL
mydb(# );
CREATE TABLE

mydb=# CREATE ROLE admin;  -- Administrator
CREATE ROLE
mydb=# CREATE ROLE bob;    -- Normal user
CREATE ROLE
mydb=# CREATE ROLE alice;  -- Normal user
CREATE ROLE
mydb=# INSERT INTO passwd VALUES
mydb-#   ('admin','xxx',0,0,'Admin','111-222-3333',null,'/root','/bin/dash');
INSERT 0 1
mydb=# INSERT INTO passwd VALUES
mydb-#   ('bob','xxx',1,1,'Bob','123-456-7890',null,'/home/bob','/bin/zsh');
INSERT 0 1
mydb=# INSERT INTO passwd VALUES
mydb-#   ('alice','xxx',2,1,'Alice','098-765-4321',null,'/home/alice','/bin/zsh');
INSERT 0 1
```
```sql
-- 啟用Row安全政策=========================================
mydb=# ALTER TABLE passwd ENABLE ROW LEVEL SECURITY;
ALTER TABLE
-- 建立各項POLICY==========================================
mydb=# CREATE POLICY admin_all ON passwd TO admin USING (true) WITH CHECK (true);
CREATE POLICY
-- 所有人可以SELECT(唯讀)
mydb=# CREATE POLICY all_view ON passwd FOR SELECT USING (true);
CREATE POLICY
-- 所有人可UPDATE自己, 且(AND)僅限於shell在指定值時
mydb=# CREATE POLICY user_mod ON passwd FOR UPDATE
mydb-#   USING (current_user = user_name)
mydb-#   WITH CHECK (
mydb(#     current_user = user_name AND
mydb(#     shell IN ('/bin/bash','/bin/sh','/bin/dash','/bin/zsh','/bin/tcsh')
mydb(#   );
CREATE POLICY
```
```sql
-- 授與相關權限=============================================
mydb=# GRANT SELECT, INSERT, UPDATE, DELETE ON passwd TO admin;
GRANT
-- 所有人僅可對指定欄位做SELECT
mydb=# GRANT SELECT
mydb-#   (user_name, uid, gid, real_name, home_phone, extra_info, home_dir, shell)
mydb-#   ON passwd TO public;
GRANT
-- 所有人僅可對指定欄位做UPDATE
mydb=# GRANT UPDATE
mydb-#   (pwhash, real_name, home_phone, extra_info, shell)
mydb-#   ON passwd TO public;
GRANT
```
```sql
-- 轉為admin role===========================================
mydb=# set role admin;
SET
mydb=> select current_user;
 current_user
--------------
 admin
(1 row)
--可查看所有記錄
mydb=> table passwd;
 user_name | pwhash | uid | gid | real_name |  home_phone  | extra_info |  home_dir   |   shell
-----------+--------+-----+-----+-----------+--------------+------------+-------------+-----------
 admin     | xxx    |   0 |   0 | Admin     | 111-222-3333 |            | /root       | /bin/dash
 bob       | xxx    |   1 |   1 | Bob       | 123-456-7890 |            | /home/bob   | /bin/zsh
 alice     | xxx    |   2 |   1 | Alice     | 098-765-4321 |            | /home/alice | /bin/zsh
(3 rows)
-- 轉為alice role
mydb=> set role alice;
SET
mydb=> select current_user;
 current_user
--------------
 alice
(1 row)
--可以查table, 但僅限指定可查的欄位才會成功
mydb=> table passwd;
ERROR:  permission denied for table passwd
mydb=> select user_name,real_name,home_phone,extra_info,home_dir,shell from passwd;
 user_name | real_name |  home_phone  | extra_info |  home_dir   |   shell   
-----------+-----------+--------------+------------+-------------+-----------
 admin     | Admin     | 111-222-3333 |            | /root       | /bin/dash
 bob       | Bob       | 123-456-7890 |            | /home/bob   | /bin/zsh
 alice     | Alice     | 098-765-4321 |            | /home/alice | /bin/zsh
(3 rows)
-- 可UPDATE自己的記錄, 但不能更新成非自己 ****
mydb=> update passwd set user_name = 'joe';
ERROR:  permission denied for table passwd
-- 未指定條件預設等於加上自己user_name為條件的UPDATE ****
mydb=> update passwd set real_name = 'Alice Doe';
UPDATE 1
-- 不可UPDATE非自己的記錄
mydb=> update passwd set real_name = 'John Doe' where user_name = 'admin';
UPDATE 0
-- 不可更新不符合shell條件的記錄
mydb=> update passwd set shell = '/bin/xx';
ERROR:  new row violates row-level security policy for table "passwd"
-- 未授與alice可DELETE的權限
mydb=> delete from passwd;
ERROR:  permission denied for table passwd
-- 未授與alice可INSERT的權限
mydb=> insert into passwd (user_name) values ('xxx');
ERROR:  permission denied for table passwd
-- 有授與UPDATE pwhash欄位的權限
mydb=> update passwd set pwhash = 'abc';
UPDATE 1
-- 再轉成owner 或 superuser role
mydb=> set role postgres;
SET
-- 建立限admin role只能在本機(inet_client_addr() IS NULL)才能存取的POLICY
-- 所以若在非本機時, admin也無法查到或更新記錄
mydb=# CREATE POLICY admin_local_only ON passwd AS RESTRICTIVE TO admin
    USING (pg_catalog.inet_client_addr() IS NULL);
CREATE POLICY
```
- 進行**備份**時應該設定`row_security=off`關閉, 以免因而造成記錄缺漏

## Schema ## 
>https://www.postgresql.org/docs/current/ddl-schemas.html
- **schema**無法建立巢狀
- 目前在PostgreSQL中如下寫法的`database`只能是目前所在資料庫, 這裡只是為了符合** SQL Standard**
```sql
database.schema.table
```
- 若**schema**內已沒其他物件, 用以下方式刪除
```sql
DROP SCHEMA myschema;
```
- 若要刪除包含**schema**內含的物件, 用以下方式
```sql
DROP SCHEMA myschema CASCADE;
```