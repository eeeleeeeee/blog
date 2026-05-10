---
date: 2026-05-10
slug: mysql-collation-pad-space
categories:
  - 資料庫
tags:
  - MySQL
  - Collation
---

# 為什麼 MySQL 的等號比較會忽略結尾空格？

## Overview

我們通常直覺地認為 SQL 的「等號 (`=`)」代表的是精準比較，也就是預期 MySQL 會嚴格比較每一個 byte，包含大小寫與所有的空白字元。然而，實際執行 Query 時發現明明搜尋條件後方多了一個空格，或是資料庫內的儲存資料結尾帶有空白（Trailing Space），MySQL 依然將它們視為「相等」並回傳結果。

主因就是隱藏在 MySQL 字元比較機制中的「Collation Pad Attributes」。

<!-- more -->

## Description

假設資料表內有以下兩筆看似相同，實則長度不同的資料：

```sql
CREATE TABLE `my_category` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `product_type` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
```

| id | product_type |
|----|---------------|
| 1  | `BUNDLES Products` |
| 2  | `BUNDLES Products␣` |

> `␣` 代表一個半形空格，下文同。

```sql
SELECT * FROM my_category WHERE product_type = 'BUNDLES Products '; -- 結尾有空格
```

**預期行為**：當搜尋條件結尾「有空格」時，應該只會出現資料 2。

**實際結果**：MySQL 卻同時回傳了資料 1 與資料 2，彷彿字串結尾的空格不存在。

## Root of Cause

問題出在資料表所設定的 COLLATION。COLLATION 決定了資料的排序與比較方式，包含 PAD SPACE 與 NO PAD。

當 Table 的 COLLATION 設為 `utf8mb4_bin` 時，MySQL 在使用 `=` 比較的過程就會採用 PAD SPACE 機制：

- **忽略結尾空格**：具備 PAD SPACE 屬性的 COLLATION，MySQL 在進行 `=` 比較時會忽略結尾的空格。
- **結果**：造成 `'BUNDLES Products'` 與 `'BUNDLES Products '` 在等號比較下被視為相同。

### PAD Attribute 比較

| Attribute | 比較（CHAR, VARCHAR 與 TEXT）行為 | Example（比較 `'a'` 與 `'a '`） |
|-----------|-----------------------------------|--------------------------------|
| PAD SPACE | 忽略字串結尾的空格 | `SELECT 'a' = 'a ' COLLATE utf8mb4_bin;` → 結果為 `true`，相等 |
| NO PAD    | 將字串結尾的空格視為一般的字元，不忽略 | `SELECT 'a' = 'a ' COLLATE utf8mb4_0900_ai_ci;` → 結果為 `false`，不相等 |

## Solution

### 修改 Collation

檢查 Schema、Table 與 Column 的 Collation，將 Collation 改成主流的 `utf8mb4_0900_ai_ci`，設定具有繼承性，子層的設定會 Override 掉父層的值。

```sql
-- Schema
SELECT SCHEMA_NAME, DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME = 'my_schema';

-- Table & Column
SELECT TABLE_NAME, COLUMN_NAME, COLLATION_NAME, CHARACTER_SET_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'my_schema' AND TABLE_NAME = 'my_category';
```

> MySQL 8.0 以上的版本在 Collation 預設使用 `utf8mb4_0900_ai_ci`。

#### Alter Schema

在 Schema 層級設定 COLLATION，以確保未來新建立的 Table 可以採用預設值。

```sql
ALTER DATABASE `my_schema`
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;
```

#### Alter Table & Columns

一次修改資料表內所有字串欄位的 COLLATION：

```sql
ALTER TABLE `my_table`
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;
```

只修改指定字串欄位的 COLLATION：

```sql
ALTER TABLE `my_table`
MODIFY `my_column` VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

> 修改 Table 與 Column 時，必須注意字元遺失、空間與長度限制、重建索引與鎖表的風險。

### 修改 Query 條件

#### LIKE

`LIKE` 不受 PAD SPACE 影響，會嚴格比較包含結尾空格在內的每一個字元。

```sql
WHERE product_type LIKE 'BUNDLES Products '; -- 結尾有空格
```

#### BINARY

強迫資料庫進行逐 byte 的二進位比對，忽略 COLLATION 的比較規則。

```sql
WHERE BINARY product_type = 'BUNDLES Products '; -- 結尾有空格
```

#### COLLATE

在比較時設定 COLLATION 指定比較規則。

```sql
WHERE product_type = 'BUNDLES Products ' COLLATE utf8mb4_0900_ai_ci; -- 結尾有空格
```

> 在查詢條件修改字串的比較條件，可能導致索引失效。

## Conclusion

當在 MySQL 使用 `=` 查詢的結果比預期還「寬鬆」且字串結尾包含空格時，可以檢查各層級（Schema、Table 與 Column）的 Collation 是否屬於 PAD SPACE。

## Reference

- [MySQL 8.0 Reference Manual :: 12.3.2 Server Character Set and Collation](https://dev.mysql.com/doc/refman/8.0/en/charset-server.html)
- [MySQL 8.0 Reference Manual :: 12.3.5 Column Character Set and Collation](https://dev.mysql.com/doc/refman/8.0/en/charset-column.html)
- [MySQL 8.0 Reference Manual :: 12.8.5 The binary Collation Compared to _bin Collations](https://dev.mysql.com/doc/refman/8.0/en/charset-binary-collations.html)
- [MySQL 8.0 Reference Manual :: 12.10.1 Unicode Character Sets](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-sets.html)
- [MySQL 8.0 Reference Manual :: 13.3.2 The CHAR and VARCHAR Types](https://dev.mysql.com/doc/refman/8.0/en/char.html)
