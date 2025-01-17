---
title: 数据类型
---

import FunctionDescription from '@site/src/components/FunctionDescription';

<FunctionDescription description="引入或更新: v1.2.100"/>

本页解释了数据类型的各个方面，包括数据类型列表、数据类型转换、转换方法，以及处理NULL值和NOT NULL约束。

## 数据类型列表

以下是Databend中通用数据类型的列表：

| 数据类型                                                            | 别名  | 存储大小 | 最小值                   | 最大值                          |
| ------------------------------------------------------------------- | ------ | ------------ | ------------------------ | ------------------------------ |
| [BOOLEAN](./00-data-type-logical-types.md)                          | BOOL   | 1 byte       | N/A                      | N/A                            |
| [TINYINT](./10-data-type-numeric-types.md#integer-data-types)       | INT8   | 1 byte       | -128                     | 127                            |
| [SMALLINT](./10-data-type-numeric-types.md#integer-data-types)      | INT16  | 2 bytes      | -32768                   | 32767                          |
| [INT](./10-data-type-numeric-types.md#integer-data-types)           | INT32  | 4 bytes      | -2147483648              | 2147483647                     |
| [BIGINT](./10-data-type-numeric-types.md#integer-data-types)        | INT64  | 8 bytes      | -9223372036854775808     | 9223372036854775807            |
| [FLOAT](./10-data-type-numeric-types.md#floating-point-data-types)  | N/A    | 4 bytes      | -3.40282347e+38          | 3.40282347e+38                 |
| [DOUBLE](./10-data-type-numeric-types.md#floating-point-data-types) | N/A    | 8 bytes      | -1.7976931348623157E+308 | 1.7976931348623157E+308        |
| [DECIMAL](./11-data-type-decimal-types.md)                          | N/A    | 16/32 bytes  | -10^P / 10^S             | 10^P / 10^S                    |
| [DATE](./20-data-type-time-date-types.md)                           | N/A    | 4 bytes      | 1000-01-01               | 9999-12-31                     |
| [TIMESTAMP](./20-data-type-time-date-types.md)                      | N/A    | 8 bytes      | 0001-01-01 00:00:00      | 9999-12-31 23:59:59.999999 UTC |
| [VARCHAR](./30-data-type-string-types.md)                           | STRING | N/A          | N/A                      | N/A                            |

以下是Databend中半结构化数据类型的列表：

| 数据类型                              | 别名 | 示例                         | 描述                                                                                                         |
| -------------------------------------- | ----- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| [ARRAY](./40-data-type-array-types.md) | N/A   | [1, 2, 3, 4]                   | 相同数据类型的值的集合，通过其索引访问。                                              |
| [TUPLE](./41-data-type-tuple-types.md) | N/A   | ('2023-02-14','Valentine Day') | 不同数据类型的值的有序集合，通过其索引访问。                                   |
| [MAP](./42-data-type-map.md)           | N/A   | `{"a":1, "b":2, "c":3}`        | 一组键值对，其中每个键是唯一的，并映射到一个值。                                              |
| [VARIANT](./43-data-type-variant.md)   | JSON  | `[1,{"a":1,"b":{"c":2}}]`      | 不同数据类型的元素集合，包括`ARRAY`和`OBJECT`。                                     |
| [BITMAP](44-data-type-bitmap.md)       | N/A   | 0101010101                     | 用于表示一组值的二进制数据类型，其中每个位表示值的存在或不存在。 |

## 数据类型转换

### 显式转换

我们有两种表达式将一个值转换为另一种数据类型。

1. `CAST` 函数，如果在转换过程中发生错误，它会抛出错误。

我们还支持pg风格的转换：`CAST(c as INT)` 与 `c::Int` 相同

2. `TRY_CAST` 函数，如果在转换过程中发生错误，它返回NULL。

### 隐式转换（“强制”）

关于“强制”（自动转换）的一些基本规则

1. 所有整数数据类型都可以隐式转换为`BIGINT`（即`INT64`）数据类型。

例如：

```sql
Int --> bigint
UInt8 --> bigint
Int32 --> bigint
```

2. 所有数值数据类型都可以隐式转换为`Double`（即`Float64`）数据类型。

例如：

```sql
Int --> Double
Float --> Double
Int32 --> Double
```

3. 所有非空数据类型`T`都可以隐式转换为`Nullable(T)`数据类型。

例如：

```sql
Int --> Nullable<Int>
String -->  Nullable<String>
```

4. 所有数据类型都可以隐式转换为`Variant`数据类型。

例如：

```sql
Int --> Variant
```

5. 字符串数据类型是最低级别的数据类型，不能隐式转换为其他数据类型。
6. `Array<T>` --> `Array<U>` 如果 `T` --> `U`。
7. `Nullable<T>` --> `Nullable<U>` 如果 `T`--> `U`。
8. `Null` --> `Nullable<T>` 对于任何`T`数据类型。
9. 数值可以隐式转换为其他数值数据类型，如果不会丢失精度。

### 常见问题

> 为什么数值类型不能自动转换为字符串类型。

这在其他流行的数据库中是微不足道的，甚至可以工作。但它会引入歧义。

例如：

```sql
select 39 > '301';
select 39 = '  39  ';
```

我们不知道如何根据数值规则或字符串规则进行比较。因为根据不同的规则，它们的结果是不同的。

`select 39 > 301` 是 false，而 `select '39' > '301'` 是 true。

为了使语法更精确且减少歧义，我们抛出错误给用户，以获得更精确的SQL。

> 为什么布尔类型不能自动转换为数值类型。

这也会带来歧义。
例如：

```sql
select true > 0.5;
```

> 错误信息是什么：“无法从可空数据转换为非空类型”。

这意味着您的源列中有一个空值。您可以使用`TRY_CAST`函数或将目标类型设为可空类型。

> `select concat(1, col)` 不起作用

您可以将SQL改进为 `select concat('1', col)`。

我们可能会在未来改进表达式，如果可能的话，将字面量`1`解析为字符串值（concat函数只接受字符串参数）。

## NULL值和NOT NULL约束

NULL值用于表示数据不存在或未知。在Databend中，每一列本质上都能够包含NULL值，这意味着一列可以容纳NULL值和常规数据。

如果您需要一个不允许NULL值的列，请使用NOT NULL约束。如果一列在Databend中配置为不允许NULL值，并且在插入数据时未明确提供该列的值，则会自动应用与该列数据类型关联的默认值。

| 数据类型                 | 默认值                                              |
| ------------------------- | ---------------------------------------------------------- |
| 整数数据类型        | 0                                                          |
| 浮点数据类型 | 0.0                                                        |
| 字符和字符串      | 空字符串 ('')                                          |
| 日期和时间数据类型  | '1970-01-01' 对于 DATE, '1970-01-01 00:00:00' 对于 TIMESTAMP |
| 布尔数据类型         | False                                                      |

例如，如果您创建如下表：

```sql
CREATE TABLE test(
    id Int64,
    name String NOT NULL,
    age Int32
);

DESC test;

Field|Type   |Null|Default|Extra|
-----+-------+----+-------+-----+
id   |BIGINT |YES |NULL   |     |
name |VARCHAR|NO  |''     |     |
age  |INT    |YES |NULL   |     |
```

- "id" 列可以包含NULL值，因为它没有 "NOT NULL" 约束。这意味着它可以存储整数或留空以表示缺失数据。

- "name" 列必须始终有一个值，因为 "NOT NULL" 约束不允许NULL值。

- "age" 列，像 "id" 一样，也可以包含NULL值，因为它没有 "NOT NULL" 约束，允许空条目或NULL值表示未知年龄。

以下INSERT语句插入一行，其中 "age" 列为NULL值。这是允许的，因为 "age" 列没有NOT NULL约束，因此它可以包含NULL值以表示缺失或未知数据。

```sql
INSERT INTO test (id, name, age) VALUES (2, 'Alice', NULL);
```

以下INSERT语句将一行插入 "test" 表，其中 "id" 和 "name" 列有值，但没有为 "age" 列提供值。这是允许的，因为 "age" 列没有NOT NULL约束，因此它可以留空或分配NULL值以表示缺失或未知数据。

```sql
INSERT INTO test (id, name) VALUES (1, 'John');
```

以下INSERT语句尝试插入一行，但没有为 "name" 列提供值。将应用列类型的默认值。

```sql
INSERT INTO test (id, age) VALUES (3, 45);
```