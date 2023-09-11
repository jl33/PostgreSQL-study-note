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