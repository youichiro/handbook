# MySQLのJSON型
MySQLはJSONデータを保存するためのJSON型をサポートしています
このドキュメントでは、MySQLのJSON型についてまとめます

## 前提
MySQL 8.0.23

## 公式ドキュメント
https://dev.mysql.com/doc/refman/8.0/ja/json.html

## 特徴
文字列型と比べたJSON型の利点
- JSONドキュメントの自動検証
    - 無効な値はエラーにできる
- 記憶域形式の最適化
    - クイック読み取りできる形式に変換される
    - JSONの全ての値を読み取ることなく、キーまたは配列インデックスによってサブオブジェクトまたはネストされた値を検索できる

JSON値の部分更新
- 削除 → 挿入 ではなく、部分的な更新が可能

## JSONレコードの作成
確認用データベースを作成する
```sql
CREATE DATABASE json_sample;
use json_sample;
```

JSONカラムつきテーブルを作成する
```sql
CREATE TABLE items (
  id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  item JSON
);
```

JSONレコードをINSERTする
```sql
INSERT INTO items (item) VALUES
  ('{"name": "hoge", "numbers": [1, 2, 3]}'),
  ('{"name": "huga", "numbers": []}')
;
```

レコードを確認する

```sql
SELECT * FROM items;
+----+----------------------------------------+
| id | item                                   |
+----+----------------------------------------+
|  1 | {"name": "hoge", "numbers": [1, 2, 3]} |
|  2 | {"name": "huga", "numbers": []}        |
+----+----------------------------------------+
```

`JSON_PRETTY`でJSONを整形して出力できる

```sql
SELECT JSON_PRETTY(item) FROM items;
+--------------------------------------------------------------+
| JSON_PRETTY(item)                                            |
+--------------------------------------------------------------+
| {
  "name": "hoge",
  "numbers": [
    1,
    2,
    3
  ]
} |
| {
  "name": "huga",
  "numbers": []
}                        |
+--------------------------------------------------------------+
```

## JSONレコードの検索
`(カラム)->"$.(キー)"`もしくは`JSON_EXTRACT`でJSONの特定のキーの値を取得する

```sql
SELECT item->"$.name" FROM items;
+----------------+
| item->"$.name" |
+----------------+
| "hoge"         |
| "huga"         |
+----------------+

SELECT JSON_EXTRACT(item, "$.name") FROM items;
+------------------------------+
| JSON_EXTRACT(item, "$.name") |
+------------------------------+
| "hoge"                       |
| "huga"                       |
+------------------------------+
```

これだと余計な `"` が入ってるが、`->>`もしくは`JSON_UNQUOTE`で外せる

```sql
SELECT item->>"$.name" FROM items;
+-----------------+
| item->>"$.name" |
+-----------------+
| hoge            |
| huga            |
+-----------------+

SELECT JSON_UNQUOTE(JSON_EXTRACT(item, "$.name")) FROM items;
+--------------------------------------------+
| JSON_UNQUOTE(JSON_EXTRACT(item, "$.name")) |
+--------------------------------------------+
| hoge                                       |
| huga                                       |
+--------------------------------------------+
```

WHEREで検索する

```sql
 SELECT * FROM items WHERE item->"$.name" = "hoge";
+----+----------------------------------------+
| id | item                                   |
+----+----------------------------------------+
|  2 | {"name": "hoge", "numbers": [1, 2, 3]} |
+----+----------------------------------------+
```

`JSON_KEYS`でJSONのキー配列を取得する

```sql
SELECT JSON_KEYS(item) FROM items;
+---------------------+
| JSON_KEYS(item)     |
+---------------------+
| ["name", "numbers"] |
| ["name", "numbers"] |
+---------------------+
```

N番目のキーを取得する
```sql
 SELECT JSON_EXTRACT(JSON_KEYS(item), "$[1]") FROM items;
+---------------------------------------+
| JSON_EXTRACT(JSON_KEYS(item), "$[1]") |
+---------------------------------------+
| "numbers"                             |
| "numbers"                             |
+---------------------------------------+
```

JSONの配列のN番目の値を取得する
```sql
SELECT JSON_EXTRACT(item->"$.numbers", "$[0]") FROM items;
+-----------------------------------------+
| JSON_EXTRACT(item->"$.numbers", "$[0]") |
+-----------------------------------------+
| 1                                       |
| NULL                                    |
+-----------------------------------------+
```


## JSONレコードの更新
`JSON_INSERT`でJSONにキーと値を追加できる

```sql
UPDATE items SET item = JSON_INSERT(item, "$.new", "うおお") WHERE id=1;

SELECT * FROM items;
+----+------------------------------------------------------------+
| id | item                                                       |
+----+------------------------------------------------------------+
|  1 | {"new": "うおお", "name": "hoge", "numbers": [1, 2, 3]}    |
|  2 | {"name": "huga", "numbers": []}                            |
+----+------------------------------------------------------------+
```

`JSON_REPLACE`でキーの値を上書きできる

```sql
UPDATE items SET item = JSON_REPLACE(item, "$.new", "うおおお") WHERE id=1;

SELECT * FROM items;
+----+---------------------------------------------------------------+
| id | item                                                          |
+----+---------------------------------------------------------------+
|  1 | {"new": "うおおお", "name": "hoge", "numbers": [1, 2, 3]}     |
|  2 | {"name": "huga", "numbers": []}                               |
+----+---------------------------------------------------------------+
```

##  JSONからデータを削除する

```sql
UPDATE items SET item = JSON_REMOVE(item, "$.new") WHERE id=1;

mysql> SELECT * FROM items;
+----+----------------------------------------+
| id | item                                   |
+----+----------------------------------------+
|  1 | {"name": "hoge", "numbers": [1, 2, 3]} |
|  2 | {"name": "huga", "numbers": []}        |
+----+----------------------------------------+
```

