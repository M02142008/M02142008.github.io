# SQLi-Labs Less-1 与 Less-5 对比：数据是怎么走到页面上的

> 这两关的 SQL 写法几乎一样（`id='$id'`），闭合也一样（`'`）。
>
> **但 LESS-1 能直接把数据打出来，Less-5 只能回答"是 / 不是"。**
>
> 这篇把"数据流"从头到尾走一遍，搞清楚每一步是谁在干活。

---

## 一、先认识几个名词

看后面流程时，遇到不认识的词回来查这张表。

| 名词 | 一句话解释 | 在流程哪一步 |
| --- | --- | --- |
| **Payload** | 你输入的内容，比如 `1' --+` | 起点 |
| **`$_GET['id']`** | PHP 用这个收 URL 里的参数 | 第 2 步 |
| **字符串拼接** | 把 payload 塞进 SQL 模板里 | 第 3 步 |
| **SQL 语句** | 拼好后发给数据库的那一整串文本 | 第 3 步产物 |
| **`mysqli_query()`** | PHP 把 SQL 发给数据库的函数 | 第 4 步 |
| **数据库 / MySQL** | 真正执行 SQL、存数据的地方 | 第 5 步 |
| **结果集** | 数据库返回的"若干行数据" | 第 5 步产物 |
| **`$row`** | PHP 里一个数组，装着"一行数据" | 第 6 步 |
| **`echo`** | PHP 把东西打印到页面上的语句 | **第 7 步（关键）** |
| **HTML / DOM** | 浏览器看到的页面结构 | 第 8 步 |
| **回显位** | 页面上"能显示数据的位置" | 看第 7 步有没有 echo |
| **盲注** | 页面不给数据，只能问"是/否" | 第 7 步没有 echo 时用 |

---

## 二、数据流全程图

这是整篇文章最重要的一张图，**从你按回车到看到结果，一共 8 步**。

```
【你】
  |
  | 1. 地址栏输入  ?id=1' --+
  v
【浏览器】
  |
  | HTTP 请求发给服务器
  v
【Apache + PHP】
  |
  | 2. PHP 收参数：
  |    $id = $_GET['id'];      // $id = "1' --+"
  v
【PHP 代码】
  |
  | 3. 字符串拼接（把 payload 塞进模板）：
  |    $sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
  |
  |    拼出来 = SELECT * FROM users WHERE id='1' --+ ' LIMIT 0,1
  v
【SQL 语句】（就是一串文本）
  |
  | 4. mysqli_query($con, $sql);
  |    （把这串文本发给数据库）
  v
【MySQL 数据库】
  |
  | 5. 解析 SQL：
  |    这一步才区分"值"和"代码"
  |    '1' --+ '   → id 等于 1，后面被注释
  |
  |    执行查询，返回结果集
  v
【结果集】（形如：一行 Dumb / Dumb）
  |
  | 6. $row = mysqli_fetch_array($result);
  |    （把一行取出来，放到 $row 数组里）
  v
【PHP 的 $row】
  |    $row = ['id'=>'1', 'username'=>'Dumb', 'password'=>'Dumb']
  |
  | 7. ★★★ echo ★★★
  |    （决定要不要把 $row 打印到页面上）
  |    ★ L1 和 L5 的差别就在这一步 ★
  v
【HTML 页面】
  |
  | 8. 浏览器渲染 HTML
  v
【你看到的画面】
```

---

## 三、关键概念：`echo` 是"出口"

### 3.1 什么是 echo

`echo` 是 PHP 的"打印语句"。

```php
echo "你好";        // 页面上出现"你好"
```

**在数据流里，`echo` 是唯一的"出口"。**

**数据到了 `$row` 里，如果没有 `echo` 去读它，它就永远出不来。**

### 3.2 一张图看清"出口"

```
       数据库
          |
          | 返回 (Dumb, Dumb)
          v
      PHP 的 $row
          |
          |  ← 数据到这里为止是"在内存里"
          |
     ┌────┴────┐
     |         |
  有 echo   没 echo
     |         |
     v         v
  页面显示   数据被丢掉
```

**有没有 `echo`，决定了这一关能不能"直接看数据"。**

---

## 四、Less-1 的完整流程

### 4.1 页面代码

```php
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result = mysqli_query($con1, $sql);
$row = mysqli_fetch_array($result);

if($row){
    echo 'Your Login name:'. $row['username'];   // ← 出口 1
    echo 'Your Password:' .$row['password'];     // ← 出口 2
}
```

### 4.2 走一遍

| 步骤 | 发生什么 | 具体内容 |
| --- | --- | --- |
| 1 | 你输入 | `?id=1` |
| 2 | PHP 收参数 | `$id = "1"` |
| 3 | 拼接 SQL | `SELECT * FROM users WHERE id='1' LIMIT 0,1` |
| 4 | 发给数据库 | |
| 5 | 数据库执行 | 返回一行：`1, Dumb, Dumb` |
| 6 | 取出一行 | `$row = ['username'=>'Dumb','password'=>'Dumb']` |
| 7 | **echo** | 打印 `Dumb` / `Dumb` |
| 8 | 页面 | 显示 `Your Login name:Dumb` |

### 4.3 注入时也一样

```
?id=-1' union select 1,2,3 --+
```

| 步骤 | 内容 |
| --- | --- |
| 3 | `SELECT * FROM users WHERE id='-1' UNION SELECT 1,2,3 --+ ' LIMIT 0,1` |
| 5 | 返回一行：`1, 2, 3` |
| 6 | `$row = ['id'=>'1','username'=>'2','password'=>'3']` |
| 7 | **echo** `$row['username']` → 打印 `2` |
| 8 | 页面显示 `Your Login name:2` |

**所以你能看到 `2` 和 `3`。**

---

## 五、Less-5 的完整流程

### 5.1 页面代码

```php
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result = mysqli_query($con1, $sql);
$row = mysqli_fetch_array($result);

if($row){
    echo 'You are in...........';        // ← 只有一句固定文字
}else{
    print_r(mysqli_error($con1));        // ← 或者打印报错
}
```

### 5.2 走一遍（正常查询）

| 步骤 | 内容 |
| --- | --- |
| 3 | `SELECT * FROM users WHERE id='1' LIMIT 0,1` |
| 5 | 返回一行：`1, Dumb, Dumb` |
| 6 | `$row = ['username'=>'Dumb',...]`（**数据确实在**） |
| 7 | `if($row)` → true → `echo 'You are in'` |
| 8 | 页面显示 `You are in` |

**注意第 6 步：`$row` 里有 Dumb，但第 7 步没人读它。**

### 5.3 走一遍（union 注入）

```
?id=-1' union select 1,2,3 --+
```

| 步骤 | 内容 |
| --- | --- |
| 3 | `SELECT * FROM users WHERE id='-1' UNION SELECT 1,2,3 --+ ' LIMIT 0,1` |
| 5 | 返回一行：`1, 2, 3` |
| 6 | `$row = ['id'=>'1','username'=>'2','password'=>'3']`（**数据在**） |
| 7 | `if($row)` → true → `echo 'You are in'` |
| 8 | 页面显示 `You are in` |

**`2` 和 `3` 就躺在 `$row` 里，但页面只显示 `You are in`。**

---

## 六、两关的差别只有一步

### 6.1 并排对比

```
【Less-1】

  $row = ['username'=>'2', ...]
        |
        |  echo $row['username']
        v
      页面显示 "2"          ✅ 数据出来了


【Less-5】

  $row = ['username'=>'2', ...]
        |
        |  if($row) → echo 'You are in'
        v
      页面显示 "You are in"  ❌ 数据没出来
```

**差在第 7 步：有没有 `echo $row[...]`。**

### 6.2 对比表

| 步骤 | Less-1 | Less-5 |
| --- | --- | --- |
| 1 输入 | 一样 | 一样 |
| 2 收参数 | 一样 | 一样 |
| 3 拼接 SQL | 一样 | 一样 |
| 4 发查询 | 一样 | 一样 |
| 5 数据库返回 | 一样 | **一样（数据都返回了）** |
| 6 取到 $row | 一样 | **一样（数据都在）** |
| **7 echo** | **打印 username** | **只打印固定文字** |
| 8 页面 | 看到数据 | 看不到数据 |

**前三步、第四步、第五步、第六步，全部一模一样。**

**唯一的差别是第 7 步。**

---

## 七、为什么 union 在 Less-5 "失效"

### 7.1 union 其实没失效

```sql
?id=-1' UNION SELECT 1,2,3 --+
```

**这条 SQL 是成功执行的。**

**数据库返回了 `1,2,3`，PHP 也把它放进了 `$row`。**

**union 完全正常工作——只是数据没人打印。**

### 7.2 准确的说法

| 说法 | 对不对 |
| --- | --- |
| "union 在 L5 里不能用了" | ❌ 不准确 |
| "union 能用，但数据没有出口" | ✅ 正确 |

### 7.3 那为什么还要用盲注

**因为你能"观察到"的，只剩下两件事：**

| 能观察到 | 从哪来 |
| --- | --- |
| 页面有没有 `You are in` | `echo 'You are in'` |
| 报错信息是什么 | `print_r(mysqli_error())` |

**除此之外，页面什么都不告诉你。**

---

## 八、所以 Less-5 有两个"出口"

### 8.1 出口一：`You are in`（1 个比特）

```php
if($row){ echo 'You are in'; }
```

**有行 → 显示，没行 → 空白。**

**只能传"真/假"——这就是布尔盲注。**

```sql
?id=1' and 1=1 --+   →  You are in     （真）
?id=1' and 1=2 --+   →  （空白）        （假）
```

### 8.2 出口二：`mysqli_error()`（32 个字符）

```php
}else{ print_r(mysqli_error($con1)); }
```

**SQL 报错时，错误信息会打印出来。**

**可以故意让它报错，把数据塞进错误里——这就是报错注入。**

```sql
?id=1' and updatexml(1,concat(0x7e,database()),1) --+
```

返回：

```
XPATH syntax error: '~security'
```

### 8.3 两个出口对比

| 出口 | 每次能传 | 方法 | 速度 |
| --- | --- | --- | --- |
| `You are in` | 1 个比特 | 布尔盲注 | 极慢 |
| `mysqli_error()` | 32 字符 | 报错注入 | 快 |

---

## 九、怎么快速判断一关能用什么方法

**方法：先看源码里有没有 `echo $row[...]`。**

```
打开 Less-X/index.php
        |
        | 搜索 "echo"
        v
  ┌─────────────────────────┐
  │ 有 echo $row['xxx'] 吗？ │
  └───────────┬─────────────┘
              |
      ┌───────┴────────┐
     有               没有
      |                |
      v                v
  有回显位          没有回显位
      |                |
      v                v
  用 union       看有没有 mysqli_error()
  联合注入             |
              ┌───────┴────────┐
             有                没有
              |                 |
              v                 v
          报错注入          布尔盲注
```

### 对照前几关

| 关卡 | `echo $row[...]` | `mysqli_error()` | 能用 |
| --- | --- | --- | --- |
| L1 | ✅ 有 | ✅ 有 | union |
| L2 | ✅ 有 | ✅ 有 | union |
| L3 | ✅ 有 | ✅ 有 | union |
| L4 | ✅ 有 | ✅ 有 | union |
| **L5** | ❌ **没有** | ✅ 有 | **报错注入 / 盲注** |

**这一张表就能解释为什么前四关长得一样、第五关突然变了。**

---

## 十、完整流程图（合并版）

```
                    你输入 payload
                          |
                          v
                   +--------------+
                   |   $_GET['id'] |   ← PHP 收参数
                   +--------------+
                          |
                          v
                   +--------------+     字符串拼接
                   |   SQL 语句    |  <-- "SELECT ... id='$id' ..."
                   +--------------+
                          |
                          v
                   +--------------+
                   |  mysqli_query |   ← PHP 发包
                   +--------------+
                          |
                          v
                   +--------------+
                   |    MySQL     |   ← 解析 + 执行
                   +--------------+
                          |
                          v
                   +--------------+
                   |  结果集 (rows) |
                   +--------------+
                          |
                          v
                   +--------------+
                   |   PHP $row    |   ← 取出一行
                   +--------------+
                          |
              +-----------+-----------+
              |                       |
        echo $row['x']          echo 'You are in'
              |                       |
              v                       v
        页面显示数据             页面显示固定文字
         （L1~L4）                 （L5）
              |                       |
              v                       v
        能用 union               只能盲注 / 报错
```

---

## 十一、总结

### 一句话

> **数据库一直在给你数据，是 PHP 决定要不要打印出来。**
>
> **`echo` 就是那个"出口"。有出口，用 union；没出口，用盲注。**

### 名词速记

| 名词 | 记成 |
| --- | --- |
| payload | 你输入的坏东西 |
| SQL 语句 | 你的输入被塞进去的那句话 |
| 数据库 | 真正执行那句话的地方 |
| 结果集 | 数据库返回的一堆行 |
| `$row` | PHP 手里的一行数据 |
| `echo` | 把数据递到页面上的手 |
| 盲注 | 手被绑住了，只能点头摇头 |

### 两关的核心差异

| | Less-1 | Less-5 |
| --- | --- | --- |
| SQL 写法 | `id='$id'` | `id='$id'`（一样） |
| 闭合 | `'` | `'`（一样） |
| 数据库返回数据 | 返 | 返（一样） |
| `$row` 里有数据 | 有 | 有（一样） |
| **有没有 echo** | **有** | **没有** |
| 结果 | 能直接看 | 只能问是/否 |

---

## 参考

- [sqli-labs GitHub](https://github.com/Audi-1/sqli-labs)
- [PHP: mysqli_fetch_array](https://www.php.net/manual/zh/mysqli-result.fetch-array.php)
- [PHP: mysqli_error](https://www.php.net/manual/zh/mysqli.error.php)
- [OWASP: Blind SQL Injection](https://owasp.org/www-community/attacks/Blind_SQL_Injection)
