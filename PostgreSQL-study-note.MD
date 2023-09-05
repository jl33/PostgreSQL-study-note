# PostgreSQL

> 以下撰寫內容可能混雜在*psql*或是*pgAdmin4*的操作使用
> 例如有`{db name}=>`就是在*psql*下做的, 否則為*pgAdmin4*。

> 部份 code 區段是*psql*執行結果的呈現, 這部份非 sql 語法

## SQL 指令特性

- 在 psql 裡面, 可寫成多行指令, 直到`;`號為止才算是一句

```sql
mydb=> CREATE TABLE sometable(
mydb(> w1 int,
mydb(> w2 varchar(10),
mydb(> w3 varchar(50));
```

```bash
CREATE TABLE # 此回應代表沒有錯誤, 執行成功
```

- 關鍵字不分大小寫, 但習慣用法是關鍵字全部大寫, 物件及欄位名稱用全部小寫
- 雙引號包住的一律視為物件及欄位名稱, 而不是關鍵字, 例如`"SELECT"`會當成名稱為 _SELECT_ 的物件而非關鍵字
- 要表達字串中的單引號要用 2 個單引號(此點同 SQL Server)。但若是有換行字元, 則會被當成連接在一起的字串

```sql
mydb=> SELECT 'foo''bar';
```

```txt
 ?column?
----------
 foo'bar
(1 row)
```

```sql
mydb=> SELECT 'foo'
mydb-> 'bar';
```

```bash
 ?column?
----------
 foobar
(1 row)
```

- 支援溢脫字元表示, 如`\n \r \t \o \x \u`等等
  - 前面要加上`e`或`E`才會以溢脫字元處理

```sql
mydb=> SELECT 'abc\ndef';
```

```bash
 ?column?
----------
 abc\ndef
(1 row)
```

```sql
mydb=> SELECT e'abc\ndef';
```

```bash
 ?column?
----------
 abc     +
 def
(1 row)
```

- `$`字符的字串表示法。
  - 以雙`$$`字符或包含 tag 的雙`$tag$`標示法, 達到免用溢脫字元或特別標示特定符號之寫法
  - `$tag$`中的*tag*有區分大小寫, 前後要一致
  - `$$`無巢狀用法, 只有最外層有效, 內層會被解譯為純文字

```sql
mydb=> SELECT $$sss' rrr $ddd"nn$$;
```

```txt
     ?column?
------------------
 sss' rrr $ddd"nn
(1 row)
```

```sql
-- $tag$的使用--------------
mydb=> SELECT $sometag$sss' rrr $ddd"nn$sometag$;
```

```txt
     ?column?
------------------
 sss' rrr $ddd"nn
(1 row)
```

```sql
-- 無巢狀使用---------------
mydb=> SELECT $function$
BEGIN
    RETURN ($1 ~ $q$[\t\r\n\v\\]$q$);
END;
$function$;
```

```
               ?column?
---------------------------------------
                                      +
 BEGIN                                +
     RETURN ($1 ~ $q$[\t\r\n\v\\]$q$);+
 END;                                 +

(1 row)
```

- 資料型態指定(轉換)

```sql
-- type 'string'
mydb=> SELECT real '1.23';
```

```bash
 float4
--------
   1.23
(1 row)
```

```sql
-- 'string'::type
mydb=> SELECT '1.23'::real;
```

```bash
 float4
--------
   1.23
(1 row)
```

```sql
-- CAST('string' AS type)
mydb=> SELECT cast('1.23' AS real);
```

```
 float4
--------
   1.23
(1 row)
```

## 函數

- 參數索引(Positional Parameters): 在函數內部以`$1 $2 $3`等代表參數的索引位置

```sql
CREATE OR REPLACE FUNCTION public.dept(
	c1 character varying,
	c2 character varying)
    RETURNS character varying
    LANGUAGE 'sql'
    COST 100
    VOLATILE PARALLEL UNSAFE
AS $BODY$  --使用$$定義函數內的程式區塊
SELECT c2 FROM demo WHERE c1=$1 and c2=$2
$BODY$;
```

- 回傳型態
  - `returns`若指定單一型態, 如`int, char`, 函數內查詢結果就只能取得相同型態的單一欄位值
  - 若是結果為多欄位, 可以用`record`做回傳,
  - 或是指定某一*table*名做為回傳, 但查詢結果的欄位數量及各欄位型態要符合`returns`所指定*table*的各欄位型態

```sql
CREATE OR REPLACE FUNCTION public.dept5(
	c1 character)
    RETURNS sometable
    LANGUAGE 'sql'
    COST 100
    VOLATILE PARALLEL UNSAFE
AS $BODY$
-- 以下回傳欄位及型態要符合上面returns指定的table定義的內容
SELECT 999 w1,c1 w2,c2 w3 FROM demo WHERE c1=$1
$BODY$;
```

## 陣列的使用

- 建立 table 時可以宣告欄位為 Array 型態, 也可加入索引數值, 但 PostgreSQL 不會限制 Array 內容個數
- 可建立多維 Array

```sql
-- 以[ ]方式宣告
CREATE TABLE demoArray(c1 varchar(5),ar1 text[2]);
-- 以ARRAY方式宣告, 但僅限一維陣列可用
CREATE TABLE demoArray2(c1 varchar(5),ar1 integer array);
```

```sql
INSERT INTO demoArray VALUES ('2','{aaa,bbb,ccc}');
SELECT * FROM demoArray;
```

```
 c1 |      ar1
----+---------------
 1  | {123,456}
 2  | {aaa,bbb,ccc}
(2 rows)
```

- array 寫入格式:
  - 以`{ }`表示維度範圍, 以`,`區別個別項目, 如`{{1,2,3},{4,5,6},{7,8,9}}`
  - 也可以用`ARRAY[ ]`做表示, 如`ARRAY[[1,2],[3,4],[9,9]]`
- null 的使用及索引查詢

```sql
INSERT INTO demoArray VALUES ('3','{a1,NULL,"NULL",hello world}');
-- 以下2句會查到同一筆記錄, 但是依不同索引內容分別查到的
SELECT * FROM demoArray WHERE ar1[2] is null;
SELECT * FROM demoArray WHERE ar1[3]='NULL';
```

```
 c1 |              ar1
----+--------------------------------
 3  | {a1,NULL,"NULL","hello world"}
(1 row)
```

- Array 索引進階查詢,

```sql
mydb=> SELECT ar1[2:3] FROM demoArray WHERE c1='3';
mydb=> SELECT ar1[2:] FROM demoArray WHERE c1='3';
mydb=> SELECT ar1[:2] FROM demoArray WHERE c1='3';
-- 使用ANY找任一個符合
mydb=> SELECT * FROM demoarray WHERE 'a1' = ANY(ar1);
```

```
      ar1
---------------
 {NULL,"NULL"}
(1 row)

             ar1
-----------------------------
 {NULL,"NULL","hello world"}
(1 row)

    ar1
-----------
 {a1,NULL}
(1 row)

 c1 |              ar1
----+--------------------------------
 3  | {a1,NULL,"NULL","hello world"}
 3  | {a2,a1,"NULL",hello}
(2 rows)
```

- 使用陣列函式查詢索引資訊

```sql
-- 查起、迄邊界
mydb=> SELECT array_dims(ar1) FROM demoArray;
-- 查上邊界索引
mydb=> SELECT array_upper(ar1,1) FROM demoArray;
```

```
 array_dims
------------
 [1:2]
 [1:3]
 [1:4]
(3 rows)

 array_upper
-------------
           2
           3
           4
(3 rows)
```

```sql
CREATE TABLE demoArray3(c1 varchar(5),ar1 integer[2][4]);
INSERT INTO demoArray3 VALUES ('3',ARRAY[[1,2],[3,4],[5,6]]);
INSERT INTO demoArray3 VALUES ('4',ARRAY[[1,2],[5,6]]);
-- 查指定維度之索引長度
mydb=> SELECT array_length(ar1,1) FROM demoArray3;
-- 查所有維度的所有項目總數
mydb=> SELECT cardinality(ar1) FROM demoArray3;
```

```
 array_length
--------------
            3
            2
(2 rows)

 cardinality
-------------
           6
           4
(2 rows)
```

```sql
-- 查在Array中的索引位置
-- 其結果可以不只一個(等於也是一個Array)
mydb=> SELECT array_position(ar1,'a1') FROM demoarray;
```

```bash
 array_position
----------------
        # psql這裡空白表示null, 查無符合記錄
        # 參考psql-note.MD中的\pset設定可改變呈現效果
              1
              2
(4 rows)
```

- Array 的組合修改

```sql
UPDATE sal_emp SET pay_by_quarter = '{25000,25000,27000,27000}';
UPDATE sal_emp SET pay_by_quarter = ARRAY[25000,25000,27000,27000];
UPDATE sal_emp SET pay_by_quarter[4] = 15000;
UPDATE sal_emp SET pay_by_quarter[1:2] = '{27000,27000}';

-- 串接, 使用 ||
ARRAY[1,2] || ARRAY[3,4]
ARRAY[5,6] || ARRAY[[1,2],[3,4]]

-- 接在前面
SELECT array_prepend(1, ARRAY[2,3]);

-- 接在後面
SELECT array_append(ARRAY[1,2], 3);

-- 串接2個
SELECT array_cat(ARRAY[1,2], ARRAY[3,4]);
```

## 用欄位組合於查詢中

- 使用**alias**在`SELECT`中代表把所有欄位當成像是**tuple**

```sql
SELECT c FROM demoarray c; --return like a tuple
SELECT * FROM demoarray c;
```

```
                    c
------------------------------------------
 (1,"{123,456}")
 (2,"{aaa,bbb,ccc}")
 (5,"{a2,a1,""NULL"",hello}")
 (7,"{a1,NULL,""NULL"",""hello world""}")
(4 rows)

 c1 |              ar1
----+--------------------------------
 1  | {123,456}
 2  | {aaa,bbb,ccc}
 5  | {a2,a1,"NULL",hello}
 7  | {a1,NULL,"NULL","hello world"}
(4 rows)
```

- ORDER BY: 以下 3 句、後 2 句是相同的意思

```sql
SELECT c.* FROM demoarray c ORDER BY c
SELECT c.* FROM demoarray c ORDER BY c.*
SELECT c.* FROM demoarray c ORDER BY ROW(c.*)
```

```sql
SELECT * FROM demoarray c ORDER BY (c.c1,c.ar1)
SELECT * FROM demoarray c ORDER BY ROW(c.c1,c.ar1)
```

- alias 別名與欄位的寫法: 以下 2 句是相同的意思

```sql
SELECT c.ar1 FROM demoarray c
SELECT ar1(c) FROM demoarray c
```

## Aggregation 彙總函式表示法

- https://www.postgresql.org/docs/15/functions-aggregate.html
- Aggregate 是彙總後輸出單一結果或群組後的結果, 與 Window Function 不同
- 5 種表示樣式

```sql
aggregate_name (expression [ , ... ] [ order_by_clause ] ) [ FILTER ( WHERE filter_clause ) ]
aggregate_name (ALL expression [ , ... ] [ order_by_clause ] ) [ FILTER ( WHERE filter_clause ) ]
aggregate_name (DISTINCT expression [ , ... ] [ order_by_clause ] ) [ FILTER ( WHERE filter_clause ) ]
aggregate_name ( * ) [ FILTER ( WHERE filter_clause ) ]
aggregate_name ( [ expression [ , ... ] ] ) WITHIN GROUP ( order_by_clause ) [ FILTER ( WHERE filter_clause ) ]
```

- Array_agg: 回傳成陣列 {a,b,c}。`c1`是`Array_agg`彙總的欄位, 並以其值組成陣列回傳

```sql
mydb=> SELECT Array_agg(c1 ORDER BY c1,ar1) FROM demoArray;
```

```
 array_agg
-----------
 {1,2,5,7}
(1 row)
```

- 同上還可以改成回傳 json 格式["a","b","c"]

```sql
mydb=> SELECT json_agg(c1) FROM demoArray;
```

```
       json_agg
----------------------
 ["1", "2", "5", "7"]
(1 row)
```

- String_agg: 以指定的字元做分隔(本例為`','`)回傳成字串。
  - 樣式為: **_String_agg(彙總表達式, 分隔符號 排序欄位)_**

```sql
mydb=> SELECT STRING_AGG(c1,',' ORDER BY c1,ar1) FROM demoArray;
```

```
 string_agg
------------
 1,2,5,7
(1 row)

```

- 其他函式

```sql
avg(num)
bit_and(num) / bit_or(num) ...
bool_and(bool) / bool_or(bool) ...
count(*)
max(n)
min(n)
range_agg(range)
sum(num)
xmlagg(xml)
percentilt_cont(double) WITHIN GROUP
rank(a) WITHIN GROUP
dence_rank(a) WITHIN GROUP
```

- **彙總函式只能出現在`SELECT`欄位或是`HAVING`表達式, 不可出現在`WHERE`(因為`WHERE`是在更早的時機做評估)**

## Window Function 視窗函式

- https://www.postgresql.org/docs/current/functions-window.html#FUNCTIONS-WINDOW-TABLE
- Window Function 可以做彙總計算, 但輸出是明細結果, 彙總結果會呈現在每筆明細上
- 視窗函式的評估順序會在任何的 Aggregation 函式、`GROUP BY`、`HAVING`之後, 所以計算時需注意引用的欄位的評估順序
- 有 4 種樣式

```sql
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
```

- 常見函式有

```sql
row_number() .. OVER
rank() .. OVER
dense_rank() .. OVER
ntile() .. OVER
lead(v,i) .. OVER
first_value(n) .. OVER
```

- 一樣有進階的範圍/群組/分頁等處理語法

```sql
RANGE | ROWS | GROUPS
UNBOUNDED PRECEDING --開頭
UNBOUNDED FOLLOWING --結尾
CURRENT ROW
EXCLUDE
```

- Window frame: 某些函式在`OVER ()`中是否有包含`ORDER BY`、`PARTITION BY`會有不同意義

```SQL
-- 這裡少了PARTITION BY,也沒有ORDER BY, 這裡的SUM等於是將所有記錄視為一組, 所以SUM是合計所有記錄
SELECT salary, sum(salary) OVER () FROM empsalary;
```

```
 salary |  sum
--------+-------
   5200 | 47100
   5000 | 47100
   3500 | 47100
   4800 | 47100
   3900 | 47100
   4200 | 47100
   4500 | 47100
   4800 | 47100
   6000 | 47100
   5200 | 47100
(10 rows)
```

```SQL
-- 這裡則是合計每一筆記錄以前的所有記錄的累計合計值(同值的會一起合計)
SELECT salary, sum(salary) OVER (ORDER BY salary) FROM empsalary;
```

```
 salary |  sum
--------+-------
   3500 |  3500
   3900 |  7400
   4200 | 11600
   4500 | 16100
   4800 | 25700
   4800 | 25700
   5000 | 30700
   5200 | 41100
   5200 | 41100
   6000 | 47100
(10 rows)
```

- **WINDOW 函式只允許在`SELECT`欄位及`ORDER BY`中使用, 不可用於`GROUP BY`、`HAVING`及`WHERE`中**
